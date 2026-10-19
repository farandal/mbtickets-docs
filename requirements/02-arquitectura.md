# MBTickets — Arquitectura de Referencia

> Documento 2 de 9. Ver `01-resumen-ejecutivo.md` para contexto general.

## 1. Patrón core / dominio (heredado, sin modificar)

Dash-Admin es un framework SaaS multi-tenant reutilizable: *"El core es dueño de la plataforma. El dominio es dueño del negocio. Ninguno se filtra en el otro."* (`dash-backend/INTRO.md`). Un producto nuevo no es un fork: es un repositorio de dominio separado, montado sobre el core.

- **Backend (`dash-backend`, Laravel 11 / PHP 8.3):** usa `wikimedia/composer-merge-plugin` para fusionar `domain/composer.json`; autoload PSR-4 reserva `Domain\App\*`; `AppDomainServiceProvider` del dominio extiende al del core y se registra solo si la clase existe (`class_exists()` guard — el core corre standalone sin dominio); rutas de dominio se incluyen vía `file_exists()`; migraciones del dominio viven en `domain/database/migrations`; en runtime el dominio se bind-mountea vía Docker Compose.
- **Frontend:** dos vías — `dash-frontend-core` (monorepo donde se desarrollan y publican los paquetes `@dashadmin/*`) y `dash-fe-core` (toolchain de arranque vacío que consume esos paquetes como dependencias npm normales y donde se montan las apps propias del producto, vía symlink o bind-mount Docker). **Recomendación:** MBTickets debe nacer sobre `dash-fe-core` (patrón limpio) y no clonar/despojar un repo de producto existente.

**Regla de oro heredada de Inamap:** *"el esqueleto es mínimo. Solo se replica la convención de carpetas y la plantilla CRUD `Example/Todo`."*

Repos a crear: `mbtickets-backend-domain` (montado sobre `dash-backend`) y `mbtickets-frontend` (montado sobre `dash-fe-core`, consumiendo `@dashadmin/*`).

## 2. Qué se hereda "gratis" del core (sin construir nada)

| Capacidad | Mecanismo en el core | Evidencia |
|---|---|---|
| Autenticación | Sanctum, OAuth Google (Socialite) | `dash-backend` |
| RBAC de dos capas | Spatie Role/Permission (rutas) + `MultiTenantAuthorizationTrait`/Policies (instancia) | `docs/MULTI_TENANT_POLICY_GUIDE.md` |
| Multi-tenancy | Jerarquía `System → Tenancy → Tenant → User` | `docs/TENANCY_ARCHITECTURE.md` |
| Scope automático por fila | `scopeVisibleTo($user)` + `canBeManagedBy` aplicado en `_setup()` del controller — fila ajena = 404, nunca fuga de existencia | patrón `InamapModel` |
| Límites por plan | `PlanLimitsService` (core) + conteo propio por dominio | patrón `InamapPlanLimits` |
| Billing/pagos de suscripción | Transbank, Flow, Rebill, Mercado Pago, gateway interno | `dash-backend` |
| Notificaciones multicanal + websockets + colas | `AppNotificationBuilder`, Reverb (Pusher-compatible), Horizon | `dash-backend` |
| Auditoría | Spatie Activitylog (`HasAuditLog`) + `venturecraft/revisionable` | `dash-backend` |
| Archivos/uploads | Spatie MediaLibrary | `dash-backend` |
| CRUD genérico | Contrato `react-admin-methods.php` ↔ paquete `dash-auto-admin` (frontend) | `dash-backend`/`dash-frontend-core` |
| Comunicación con dispositivos de borde (POS/kiosko/scanner) | `DeviceCore`: `Device`, `DeviceNode`, `DeviceCommand`, `DeviceSessionLog`, heartbeat, presencia en Redis, `DeviceStatusBroadcaster` | `app/DeviceCore` |
| Generación visual de QR (sin firma) | `endroid/qr-code` ya es dependencia del core | `composer.json` |
| Reportes | `ReportCore` (exportadores, fuentes de datos) | `dash-backend` |

**Decisión de diseño clave:** las entradas de MBTickets para dispositivos de validación (torniquetes, scanners de mano) se modelan como registros de `DeviceCore`, extendiendo su framework de heartbeat/comandos/presencia en vez de construir uno propio — solo se añade la lógica de dominio de verificación criptográfica encima.

## 3. Qué se reutiliza como *patrón* (no como código) desde Inamap

- Multi-tenancy de 2 niveles con `scopeVisibleTo`/`canBeManagedBy` — base literal para que un Organizer solo vea sus propios eventos y un Administrator vea todo.
- RBAC vía permisos nuevos (`mbtickets.*`) sembrados en JSON sobre los roles existentes del core — no se redefinen roles.
- PK UUID en toda tabla de negocio, soft delete (`deleted_at`), `tenancy_id`/`tenant_id` denormalizados en tablas hijas para evitar joins en caminos calientes (ej. verificación de QR en el torniquete).
- **Patrón de token opaco de un solo uso:** `random_bytes(32)` → base64url, se persiste solo el hash SHA-256, el valor en claro se entrega una única vez, va en el fragmento de la URL del QR (nunca en logs del servidor), con TTL y poda programada. Es la base del *canal de entrega* del QR de MBTickets (no de su verificación offline, ver §5).
- Separación de algoritmos críticos en **servicios PHP puros sin dependencia de Laravel**, testeables de forma aislada (ej. `RouteEngine`/`Graph` en Inamap) — es el estilo obligatorio para el motor de locking de inventario y el motor de firma/verificación de MBTickets.

> **Nota de corrección (reemplaza una decisión anterior de este documento):** el "guardado atómico de documento compuesto" de Inamap (`LayoutGeometryController`) ya **no** se reutiliza como *patrón de código* para un editor de mapas propio de MBTickets. El mapa de asientos de una Sala **no se construye dentro de MBTickets**: se integra directamente con Inamap como sistema externo (ver §4.3). El patrón de guardado atómico sigue siendo relevante, pero vive y se ejecuta del lado de Inamap, no de MBTickets.

## 4. Modelo de negocio: Tenancy → Tenant (Venue) → Sala → Evento

Aclaración de dominio confirmada con el equipo: en MBTickets, **el `Tenant` del core representa un Venue** (recinto físico operado por una organización): un teatro, un complejo de cines, un estadio, una arena. No se crea una entidad "Venue" separada — se reutiliza `Tenant` con ese significado de negocio, igual que Inamap reutilizó `Tenant` para "un recinto con mapa".

- **Tenancy** = la organización/cadena que administra uno o más recintos (ej. una cadena de cines, un municipio con varios teatros, un operador de estadios). Un **Tenancy Admin** ve/gestiona todos los Tenants (venues) de su organización.
- **Tenant = Venue** = un recinto físico concreto (ej. "Cine Hoyts Parque Arauco", "Teatro Municipal de Santiago", "Estadio Nacional"). Tiene su propia configuración, su propio equipo de Organizers, y su propio conjunto de Salas.
- **Sala** (nueva entidad de dominio, hija de Tenant/Venue) = un espacio reservable dentro del venue: una sala de cine, el escenario principal de un teatro, la cancha+tribunas de un estadio (que puede modelarse como una única "Sala" de capacidad muy grande con múltiples sectores). **Un venue puede tener múltiples salas, y cada sala puede alojar múltiples eventos en el tiempo (incluso simultáneamente en distintas salas del mismo venue).**
- **EventType** (igual que en los requerimientos originales) se asocia a una Sala, no directamente al Venue: define qué se ofrece (nombre, descripción, duración, precio, capacidad base).
- **EventSession** = una función/sesión concreta de un EventType en una fecha/hora específica dentro de su Sala. Es sobre el `EventSession` que corre el motor de inventario/locking (ver §6 y `05-requerimientos-no-funcionales.md` NFR-01).

```
Tenancy (cadena/organización)
  └─< Tenant = Venue (ej. "Cine Hoyts Parque Arauco")
        └─< Sala (ej. "Sala 1", "Sala IMAX", "Escenario Principal", "Cancha+Tribunas")
              └─< EventType (ej. "Avatar 3", "Hamlet", "Clásico Universitario")
                    └─< EventSession (función/fecha-hora concreta en esa Sala)
                          └─< InventoryUnit (asiento numerado o bloque GA de esa sesión)
```

### 4.1 Regla de negocio: no-solapamiento de Sala

Distinta de la concurrencia de *inventario* (NFR-01, que protege contra sobreventa del mismo asiento/cupo): **una misma Sala no puede tener dos `EventSession` con horarios que se solapen** (ej. dos películas no pueden proyectarse a la vez en la Sala 4). Esta validación ocurre al **crear/editar el calendario de funciones** (momento de programación, baja concurrencia), no al momento de venta (ver FR-16).

### 4.2 Casos de uso por vertical (mismo modelo, distinta instanciación)

| Vertical | Tenant (Venue) | Sala(s) | Patrón de EventSession | Tipo de inventario |
|---|---|---|---|---|
| **Teatro** | Un teatro | 1 (a veces 2) | Temporada de funciones del mismo EventType en la misma Sala, en distintas fechas/horas | Asientos numerados fijos (mapa Inamap opcional) o GA |
| **Cine (multiplex)** | Un complejo | Muchas (8–16 salas) | Alta rotación: cada Sala corre decenas de EventSessions por semana de distintos EventType (películas) | Asientos numerados (mapa Inamap opcional), reutilizado por todas las funciones de la sala |
| **Estadio / Arena** | Un estadio | Normalmente 1 Sala con **Sectores** | Eventos esporádicos o temporada, volumen de inventario extremo | Mezcla de asientos numerados (mapa Inamap) y admisión general por sector |
| **Boletería de taquilla física (cualquier vertical)** | El mismo Tenant/Venue | La(s) misma(s) Sala(s) que el canal online | Vende sobre la misma `EventSession` e inventario que el canal online y el kiosko — ver FR-07 | Igual que el canal online, mismo motor de locking |

### 4.3 Integración con Inamap para mapas de Sala (NUEVO — reemplaza el enfoque de "editor propio")

**Decisión de negocio confirmada:** el mapa de asientos/sectores de una Sala es **opcional**, y cuando existe, **se crea y mantiene en Inamap**, no en MBTickets. MBTickets consume ese mapa mediante una **integración bidireccional**, en vez de construir su propio editor gráfico de asientos.

**Flujo operativo:**
1. El Organizer (o quien administre el mapa) crea el recinto/piso en **Inamap** usando su editor existente (`Inamap` → `Layout` → `Zone`/`Structure`/`Location`), donde cada `Location` relevante representa un asiento o un punto de referencia del sector.
2. Desde **MBTickets**, el Organizer **asocia** la Sala a un `Inamap` + `Layout` ya existente (selector que consulta la API de Inamap para ese tenant).
3. MBTickets **sincroniza (pull)** las `Location` del layout asociado para generar/actualizar el inventario de asientos (`InventoryUnit`) de esa Sala, guardando una referencia externa (`inamap_location_id`) en cada `InventoryUnit`.

**Qué significa "bidireccional" en este diseño:**
- **MBTickets → Inamap (lectura):** MBTickets llama a la API de Inamap para obtener la geometría/listado de `Location` del layout asociado. Se propone reutilizar el mismo mecanismo de credencial que Inamap ya expone para integraciones externas (el patrón de `access_key` hasheada visto en `InamapAccessToken`, análogo a su integración con Vanexa), en vez de construir autenticación nueva.
- **Inamap → MBTickets (cambios):** si el mapa cambia en Inamap (se agrega, mueve o elimina un asiento) después de la sincronización inicial, MBTickets debe detectarlo y volver a sincronizar. **A definir en diseño detallado** si esto se resuelve por webhook/evento (si Inamap llega a exponerlo) o por *polling* con comparación de versión/checksum del layout.
- **MBTickets → Inamap (metadata de vuelta, opcional):** para que quien administra el mapa en Inamap sepa que una `Location` específica ya está en uso por ticketing (y no deba eliminarla sin coordinar), MBTickets podría escribir una referencia de vuelta en Inamap (ej. un campo/etiqueta en la `Location`). **Esto depende de que Inamap exponga un campo personalizado o de integración en `Location` — no confirmado en el código revisado; es una pregunta abierta para el equipo de Inamap.**

**Consecuencias de diseño:**
- **Salas sin mapa** (venues solo de admisión general, o eventos sin asiento numerado) **no requieren esta integración**: su inventario es un contador simple de capacidad por sector, gestionado íntegramente dentro de MBTickets.
- El **motor de locking de inventario (NFR-01) sigue corriendo 100% dentro de MBTickets**, sobre `InventoryUnit`. Inamap es solo la fuente de la geometría/identidad de cada asiento (dónde está, cómo se llama), **nunca de su estado de venta** (disponible/retenido/vendido) — eso evita acoplar el componente más crítico del sistema a la disponibilidad de un servicio externo.
- Si Inamap está caído o inalcanzable, la **creación o actualización** de una Sala con mapa nuevo no puede completarse (dependencia dura en ese momento puntual), pero la **venta** sobre asientos ya sincronizados **no depende** de que Inamap esté disponible (el inventario ya vive en MBTickets).

**Preguntas abiertas para el equipo (a resolver antes del diseño detallado):**
- ¿Inamap ya expone un endpoint de API pensado para integraciones externas tipo "exportar `Location` de un layout" (reutilizable tal cual, como hace con Vanexa), o hay que construir uno nuevo en el dominio de Inamap?
- ¿Soporta o soportaría Inamap un campo personalizado en `Location` para la referencia de vuelta hacia MBTickets?
- ¿Con qué frecuencia se espera que cambie un mapa ya usado en producción? (Define si el pull es bajo demanda, programado, o dirigido por evento/webhook.)
- ¿Un mismo mapa de Inamap puede asociarse a más de una Sala (ej. salas gemelas con igual layout) o la relación es siempre 1:1?

### 4.4 Categorización y atributos propios de un EventType (FR-18)

La variación entre verticales (cine, teatro, estadio) se resuelve con **datos**, no con ramas de código ni de esquema:

- **`EventCategory`**: tabla plana a nivel de plataforma (`id` UUID, `slug`, nombre traducible, `deleted_at`), administrada solo por el Administrator, no por cada venue. Es una etiqueta de clasificación, sin jerarquía ni comportamiento. `EventType` la referencia con `event_category_id`. Se usa únicamente para descubrimiento (FR-01.1) y agregación de reportes (FR-13).
- **`custom_attributes`**: columna `jsonb` en `EventType`, pares clave-valor libres definidos por el Organizer (ej. clasificación, formato e idioma de una película; elenco de una obra; equipos de un partido). Sin esquema impuesto: no se valida ni se interpreta. Se renderiza tal cual en el detalle del evento y se puede indexar con GIN para filtrar dentro de una categoría.
- **Invariante:** el motor de locking (NFR-01), el de validación de QR (FR-09) y el de notificaciones (FR-10) no leen `EventCategory` ni `custom_attributes`. Si alguno de los tres necesitara una rama por categoría, el diseño está mal planteado.
- Ortogonal a la jerarquía de §4: la categoría describe al `EventType`, no al venue ni a la sala que lo aloja.

## 5. Qué NO existe en ningún sistema hermano y debe construirse íntegramente

Esta es la sección más importante del documento: dos reportes independientes (análisis de Inamap y de KitchnTabs) confirmaron, buscando explícitamente patrones de `lockForUpdate`, `SERIALIZABLE`, `sharedLock` en ambos códigos, que **no existe ningún precedente de locking de inventario reservable** en el stack de la compañía:

- **Inamap** no tiene ningún recurso agotable que proteger (es un motor de mapas); su única transacción es el guardado atómico de geometría (todo-o-nada, sin relación con cupos).
- **KitchnTabs** sí tiene un `Stock` de ingredientes, pero **nunca se bloquea ni se descuenta atómicamente al crear un pedido** (`TabOrderManagementTrait::createTabWithOrder()` no contiene `lockForUpdate()` ni toca `Stock`). El único `lockForUpdate()` real del código (`AgentCartService::exclusively()`) protege el *carrito* de una sesión de IA contra sus propias tool-calls concurrentes — es concurrencia de sesión, no de stock.

Por lo tanto, **NFR-01 (zero overbooking) es, con evidencia de código, el componente de mayor riesgo y sin precedente del proyecto.** Debe diseñarse de cero como un servicio de dominio aislado (ver `03-requerimientos-tecnicos.md` §4) con batería de pruebas unitarias dedicada.

De igual forma, **no existe firma criptográfica de credenciales en ningún sistema hermano**:
- Inamap hashea tokens con SHA-256 y valida siempre contra la base de datos central — nunca offline.
- KitchnTabs genera QR con `endroid/qr-code` sobre una cadena arbitraria, sin firma, sin payload estructurado, sin estado "canjeado" de un solo uso — y de hecho tiene un incidente de seguridad documentado (`ISSUE.md`) de sesiones creadas sin QR real escaneado.

MBTickets necesita diseñar de cero: firma asimétrica (Ed25519/ECDSA) del payload del ticket, cacheo de la clave pública en cada dispositivo de validación, y verificación local sin red con estado "REDEEMED" sincronizado (anti-passback) cuando vuelve la conectividad.

Tampoco existe mensajería de dominio desacoplada lista para usar: ambos sistemas usan Notifications/Observers acoplados al ciclo de vida del modelo. El patrón Job+Listener del core (ej. `TenantProvisioningJob`) es el molde a replicar para los eventos concretos de MBTickets (`BookingConfirmed`, `TicketValidated`).

## 6. Modelo de dominio propuesto (detalle de agregados)

```
Tenancy ──< Tenant (= Venue) ──< Sala (opcional: referencia a Inamap Layout)
Sala ──< EventType (Name, Description, Duration, Price, Capacity, TimeZone)
EventType ──< ScheduleRule (reglas recurrentes)
EventType ──< DateOverride (excepciones de fecha)
EventType ──< EventSession (instancia concreta: fecha/hora, Sala heredada, capacidad efectiva)
EventSession ──< InventoryUnit (asiento con inamap_location_id opcional, o bloque GA, con estado disponible/retenido/vendido)
EventSession ──< Booking (reserva/compra) ──< BookingItem ──< TicketCredential (QR firmado, estado válido / REDEEMED / VOID)
Booking ──< PosPaymentTransaction | CheckoutGatewayTransaction (pago presencial u online, ver 09 y FR-26)
Booking ──< Refund (monto, estado, origen: comprador / organizer / admin / sistema; ver FR-25)
EventType ──> EventCategory (etiqueta de plataforma) · EventType.custom_attributes (jsonb libre) — ver §4.4
Device (torniquete/scanner/POS/kiosko) ──< ValidationLog
Tenant ──< ApiClient (credencial de integración externa, revocable, con scopes) ──< ApiRequestLog
Tenant ──< WebhookEndpoint (URL, secreto, eventos suscritos) ──< WebhookDelivery (intentos, estado, latencia)
```

`TicketCredential.VOID` se alcanza por cancelación o reembolso (FR-25). Los dispositivos de validación reciben la lista de credenciales anuladas por `DeviceCore`, igual que reciben la clave pública (NFR-04).

`ApiClient` y `WebhookEndpoint` (FR-24) son entidades de dominio de MBTickets; la autenticación de `ApiClient` reutiliza el patrón de credencial hasheada y revocable de FR-11, no un mecanismo nuevo.

`Sala`/`EventType`/`ScheduleRule`/`DateOverride`/`InventoryUnit` **no tienen equivalente en el core** (lo más cercano, `TenantSchedule`, solo resuelve horario de apertura semanal de una tienda) — se construyen 100% nuevos en el dominio de MBTickets.

## 7. Clasificación resumida (heredado vs. construir vs. integrar)

| Heredado del core (gratis) | Patrón reutilizable (adaptar) | Integración externa | Construir de cero en MBTickets |
|---|---|---|---|
| Auth, RBAC 2 capas, multi-tenancy (System → Tenancy → Tenant → User) | Multi-tenancy Tenancy→Tenant de Inamap, scopeVisibleTo | **Mapa de asientos de Sala ↔ Inamap Layout/Location** | Motor transaccional de locking de inventario (NFR-01) |
| Billing/suscripciones, pasarelas de pago | Capa de pagos de KitchnTabs (gateway intercambiable) | | Entidad Sala + EventType + ScheduleRule + DateOverride + disponibilidad |
| Notificaciones multicanal, websockets, colas | Patrón de notificación por cambio de estado de KitchnTabs | | Firma criptográfica Ed25519 + validación offline single-use |
| Auditoría, media/uploads | Patrón de token opaco hasheado de Inamap (canal de entrega del QR) | | Eventos de dominio desacoplados (BookingConfirmed, TicketValidated) |
| CRUD genérico (dash-auto-admin) | | | Flujo de venta POS/kiosko/boletería sobre DeviceCore |
| DeviceCore (heartbeat/comandos/presencia) | Reportes de embudo de KitchnTabs (adaptar a funnel de venta/check-in) | | Validación de no-solapamiento de Sala en el calendario de funciones |
| Generación visual de QR (`endroid/qr-code`) | | | Lógica de anti-passback y sincronización de estado REDEEMED; sincronización pull/webhook con Inamap |
