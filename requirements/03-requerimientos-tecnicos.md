# MBTickets — Requerimientos Técnicos

> Documento 3 de 9. Ver `02-arquitectura.md` para el contexto de qué se hereda del core.

## 1. Stack tecnológico (heredado del core, confirmado por inspección directa)

**Backend**
- PHP 8.3 (requiere ^8.2), Laravel 11.
- PostgreSQL (sin PostGIS en los dominios hermanos — geometría/datos espaciales no aplican a MBTickets, pero la convención de "sin extensiones exóticas, algoritmos propios donde haga falta" sí aplica al motor de locking).
- Redis / Valkey: cache, colas, broadcasting.
- Laravel Horizon (supervisión de colas), Laravel Reverb (WebSockets, compatible Pusher) o Pusher.com directo.
- Laravel Sanctum (tokens API) + Socialite (OAuth Google).
- Spatie: `laravel-permission`, `laravel-activitylog`, `laravel-medialibrary`.
- `endroid/qr-code` ^5.0 (ya en el core) para render visual de QR.
- Pasarelas de pago ya integradas a nivel de core/dominio hermano: Transbank, Flow, Rebill, Mercado Pago, gateway interno.
- Despliegue: Docker Compose (dominio bind-mounteado sobre el core), nginx + php-fpm tras AWS ALB, Cloudflare DNS, IaC vía CDK en repos hermanos.

**Frontend**
- Monorepo pnpm + Turborepo.
- React 19, TypeScript 5.9, Vite 7.
- MUI (v7–9 según paquete), Redux Toolkit, TanStack Query, Axios.
- `react-admin` 5.x para paneles de gestión (Organizer/Administrator), consumiendo el contrato `react-admin-methods.php` del backend vía `dash-auto-admin`.
- Paquetes propios bajo scope `@dashadmin/*` (publicados a npm/Verdaccio) + paquetes de dominio `mbt-*` análogos a `ina-*`/`kt-*`.
- Capacitor (apps móviles) y Electron (apps de escritorio, incluyendo kiosko/POS) disponibles en el core para los clientes nativos.
- `qrcode.react` para render de QR en el visor de entradas del comprador (patrón visto en `ina-viewer`).

## 2. Librerías y componentes NUEVOS que MBTickets debe incorporar (no están en el core)

| Necesidad | Librería/enfoque propuesto | Justificación |
|---|---|---|
| Firma criptográfica de credenciales (Ed25519) | `paragonie/sodium_compat` o extensión `libsodium` nativa de PHP (ya disponible en PHP 8.3) | El core solo trae `endroid/qr-code` (render), sin firma asimétrica. Ed25519 permite firmas compactas, verificación rápida (<500ms, NFR-03) y verificación completamente offline con solo la clave pública cacheada en el dispositivo. |
| Verificación offline en el dispositivo de validación | SDK/cliente ligero embebido en la app de torniquete/handheld (Electron o nativo) con la clave pública del tenant cacheada localmente + cola local de eventos `TicketValidated` a sincronizar cuando vuelva la red | NFR-04. Ningún sistema hermano resuelve esto; se construye específicamente para MBTickets sobre la infraestructura de `DeviceCore` (heartbeat/comandos ya existentes). |
| Transacciones serializables / locking de fila | Eloquent + `DB::transaction(fn () => ..., $attempts)` con `lockForUpdate()` sobre la fila de `InventoryUnit`/contador de capacidad, o patrón `UPDATE ... SET remaining = remaining - :qty WHERE remaining >= :qty RETURNING id` para admisión general sin lock explícito | PostgreSQL soporta `SERIALIZABLE` nativamente; se evalúa en diseño detallado si se usa aislamiento `SERIALIZABLE` con reintento automático en conflicto (40001) o locking pesimista explícito — ver `05-requerimientos-no-funcionales.md` NFR-01. |
| Reserva temporal (hold) de inventario | Tabla `InventoryHold` con `expires_at` + job programado de liberación, o TTL en Redis sincronizado con el estado en Postgres | Necesario para el flujo "seleccionar asiento → pagar" sin bloquear la fila todo el checkout. |
| Eventos de dominio desacoplados | Clases `Event` + `Listener` de Laravel (patrón ya usado en `Billing/*` del core) despachadas a cola Redis/Horizon, nunca invocación síncrona directa como hace hoy `TabsNotificationService` en KitchnTabs | Cumple FR-10 y el requerimiento arquitectónico 3.3 de desacople mediante Message Broker. |
| Webhooks salientes (FR-24) | Listener por evento de dominio que encola un `Job` de entrega por `WebhookEndpoint`; firma HMAC-SHA256 sobre `timestamp.body` con secreto por endpoint; reintentos con backoff exponencial acotado; `event_id` estable para deduplicación en el receptor | Propuesta. No existe precedente en el core ni en los dominios hermanos. Se construye sobre Horizon, sin infraestructura nueva. |
| API externa de inventario (FR-24) | Rutas versionadas (`/api/external/v1/...`) autenticadas con `ApiClient` (credencial hasheada, scopes, revocable), `Idempotency-Key` obligatorio en escrituras, rate limiting por credencial | Las escrituras invocan el mismo servicio de dominio de NFR-01. No hay un segundo camino de escritura sobre el inventario. |
| Lista de credenciales anuladas en dispositivos (FR-25.4) | Sincronización incremental de `TicketCredential` en estado `VOID` hacia cada dispositivo de validación vía `DeviceCore` (mismo canal que la clave pública) | Sin esto, una entrada anulada con firma válida seguiría pasando en un torniquete offline. |
| Reembolsos de pagos online | `AbstractCheckoutGatewayProvider::refundPayment()` de la capa `CheckoutGateway` de KitchnTabs, que MBTickets reutiliza (stub por defecto, implementable por pasarela) | Para pagos con terminal físico, `refund()` del conector (`09-feature-pos-payment-terminal-integration.md` §4.2). |

## 3. Convenciones de datos (heredadas de Inamap, a replicar)

- PK UUID en toda tabla de negocio (`HasUuids`).
- `tenancy_id` + `tenant_id` denormalizados en toda tabla, y además el identificador del agregado padre denormalizado en tablas hijas de alta frecuencia de lectura (ej. `event_session_id` en `InventoryUnit`) para evitar joins en el camino caliente de validación de entrada.
- Borrado lógico (`deleted_at`) en todas las tablas.
- **MBTickets no edita ni es dueño de la geometría del mapa de asientos.** El mapa se crea y mantiene en Inamap (`02-arquitectura.md` §4.3). `InventoryUnit` guarda solo `inamap_location_id` y la etiqueta mínima para mostrar el asiento (sector, fila, número). Si el frontend necesita geometría para dibujar el mapa al comprador, se obtiene de Inamap o se guarda como snapshot de solo lectura en `jsonb`, sin extensiones espaciales (a definir en diseño detallado, junto con la pregunta abierta de frecuencia de cambio de mapas).
- El patrón de guardado atómico de documento compuesto (`LayoutGeometryController` de Inamap) corre del lado de Inamap y no se replica en MBTickets. Ver la nota de corrección de `02-arquitectura.md` §3.
- Atributos libres de `EventType` (`custom_attributes`) en `jsonb`, con índice GIN solo si se usan como filtro (`02-arquitectura.md` §4.4).

## 4. Estrategia de pruebas (obligatoria, no opcional)

Dado que ni Inamap ni KitchnTabs tienen precedente de código para el motor de concurrencia ni para la firma criptográfica, ambos componentes se construyen como **servicios PHP puros sin dependencia de Laravel** (mismo estilo que `RouteEngine`/`Graph` en Inamap), con:
- Batería de pruebas unitarias aisladas (sin HTTP, sin base de datos real cuando sea posible, o con transacciones de prueba revertidas).
- Pruebas de concurrencia dirigidas: simular N requests simultáneas contra 1 unidad de inventario y verificar que exactamente 1 confirma.
- Pruebas de corrupción/forjado de QR: firma inválida, timestamp expirado, doble canje (replay).
- Pruebas de resiliencia offline: validación local sin red, reconciliación al reconectar.

## 5. Infraestructura y despliegue

- Reutiliza el patrón de `dash-backend-docker`: Compose local con bloque de puertos dedicado para MBTickets (análogo al de Inamap: API, Reverb, Postgres, Redis en puertos propios).
- Producción sobre la misma infraestructura core (nginx/php-fpm, ALB, Cloudflare, CDK) — sin IaC nuevo salvo lo específico de MBTickets (si aplica, colas o buckets propios).
- Horizon dedicado o compartido para los jobs de notificación/eventos de dominio — a decidir en diseño detallado según volumen esperado (festivales de 20.000 entradas en minutos).

## 6. Dependencias pendientes de validación con el equipo

- ¿`dash-fe-core` ya está en condiciones de usarse como base limpia para un producto nuevo, o conviene partir clonando `kitchntabs-frontend`/`inamap-frontend-domain` y despojar marca? (Recomendación del análisis: usar `dash-fe-core`, es el camino pensado para esto.)
- ¿Horizon/Redis compartido con otros dominios o instancia dedicada para MBTickets, dado el patrón de picos de tráfico (venta de festivales)?
- ¿Se requiere impresión térmica de entradas físicas en POS? (De ser así, `kitchntabs-python-service` es reutilizable tal cual para ESC/POS.)

## 7. Anexo técnico: integración con terminales de pago físicos — Transbank y Mercado Pago (investigación)

> Este anexo documenta hallazgos de investigación externa (documentación pública de desarrolladores de Transbank y Mercado Pago, consultada en octubre de 2026) que fundamentan y ajustan el diseño de integración de punto de venta físico especificado en `09-feature-pos-payment-terminal-integration.md`. Se incluye en Requerimientos Técnicos porque documenta capacidades y restricciones reales de terceros — no decisiones de producto de MBTickets — que condicionan qué es técnicamente posible construir. Las brechas de información quedan marcadas explícitamente como tales, en vez de asumidas.

### 7.1 Transbank — "POS Integrado"

**Qué es:** modo de operación del terminal de pago de Transbank en el cual la caja del comercio controla el cobro (envía el monto) y el terminal ejecuta la transacción, imprime el comprobante y devuelve el resultado a la caja.

**Hardware confirmado:** terminales Verifone VX520/VX520C e Ingenico Desk 3500 (equipos externos, independientes de cualquier tablet/kiosko).

**Protocolo y transporte — hallazgo clave:** la integración es por **cable serial (RS-232) o USB**, no por red ni Bluetooth:
- Apertura de puerto por nombre (`COM4`, `/dev/tty...`) y baud rate (115.200 por defecto).
- El SDK web (JavaScript) no habla directo al terminal: requiere un **agente de escritorio** que expone un servidor WebSocket local en el puerto 8090, y es ese agente el que efectivamente habla serial/USB con el terminal.
- Ethernet se menciona en la documentación solo para fijar la IP del terminal, no como protocolo de integración de la venta.

**SDKs oficiales disponibles:** .NET (`TransbankPosSDK`, NuGet), Node.js (`transbank-pos-sdk`, npm), Java (`transbank-sdk-pos-java` v3.1.1, Maven), Python (`transbank-pos-sdk`, PyPI), JavaScript/web (`transbank-pos-sdk-web`, requiere el agente de escritorio descrito arriba), y acceso a los comandos nativos documentados para integraciones en C u otros lenguajes.

**No existe SDK Android nativo/embebible.** La documentación pública de Transbank Developers revisada no menciona ningún SDK Android ni un modo de pairing Bluetooth entre un tablet y el terminal. Esto ajusta una premisa del diseño en doc 09: el camino "Android-paired terminal" (Opción A) **no aplica tal cual a Transbank**, salvo que el propio tablet-kiosko exponga un puerto físico USB-OTG y se ejecute, dentro de una app Android, la librería Java del SDK hablando el protocolo serial contra el terminal conectado por cable — variante técnicamente posible, pero que en la práctica tiene la misma dependencia de cableado físico directo que conectar el terminal a un Raspberry Pi, y no es un "SDK Android nativo del fabricante". **Conclusión:** para Transbank, el camino recomendado y mejor soportado en v1 es la Opción B (puente Raspberry Pi), usando el SDK oficial Python o Node.js del protocolo POS Integrado.

**Flujo de venta (consistente entre SDKs):** la caja envía monto + número de ticket (con flag opcional para recibir mensajes intermedios de estado, entregados como evento en .NET, listener en Java o callback en JavaScript) → el terminal ejecuta la transacción, imprime el voucher con el número de ticket → devuelve una `SaleResponse` (o `char*` en C) con campos como `Response` (ej. "Aprobado"), `Authorization Code`, `Amount`, `Operation Number`, `Card Brand`. Los errores se señalizan como excepción (`TransbankSaleException` en .NET/Java).

**Otras operaciones del protocolo, relevantes para mapear a `PosPaymentConnectionClient`:**
- Última venta (reimpresión del comprobante más reciente).
- Anulación (`cancel()`/`refund()` del contrato): requiere el identificador de operación de la venta original.
- Totales y detalle de transacciones en memoria del terminal (útil para conciliación/reportería, FR-13).
- Cierre de lote contra el autorizador (la documentación indica que no soporta carga de lote).
- Carga de llaves de trabajo, poll de conexión, cambio a modo normal.

**Nota — "POS Multicódigo Integrado":** variante en la que un mismo terminal físico opera bajo múltiples códigos de comercio (RUT), asignando cada venta al RUT correspondiente seleccionado en el momento de cobrar (requiere que tanto el RUT responsable como cada RUT proveedor estén afiliados a Transbank, con el anexo de contrato correspondiente). Relevante únicamente si algún venue necesitara compartir un único terminal físico entre varios Organizers/eventos del mismo recinto, liquidando cada venta a una cuenta distinta; no es necesario para el v1, pero queda documentado como variante disponible del mismo producto.

**Brechas — no resuelto por documentación pública, pendiente de validar directamente con Transbank:** proceso de certificación del comercio/integración; disponibilidad y costos de los modelos de terminal en Chile; tiempos de habilitación del modo "POS Integrado" por comercio.

### 7.2 Mercado Pago — "Point"

**Modelos de terminal confirmados en documentación vigente:** Point Smart (1 y 2), Point Plus y Point Air (listados como dispositivos disponibles vía la API de integración). Point Mini existe como producto comercial (confirmado en blogs y fichas de producto de Mercado Pago) pero no aparece nombrado en la documentación de integración API vigente revisada — no se pudo confirmar si mantiene una vía de integración propia distinta de la de Point Smart/Plus/Air.

**Se identificaron tres patrones de integración distintos, no uno solo:**

**(a) Cloud API** ("Point Integration API", en migración hacia una nueva "Orders API" según advierte la propia documentación): el backend de MBTickets llama directamente a la API de Mercado Pago (`GET /point/integration-api/devices` para listar equipos, `POST /point/integration-api/devices/{deviceid}/payment-intents` para crear el cobro, monto en unidades menores — ej. 1500 = $15,00 —, asociándolo a un `device_id` específico). El operador presiona el botón de cobro en el propio terminal (ej. "Cobrar" en Point Smart); el mecanismo exacto por el cual el terminal obtiene el intent creado (polling interno del equipo vs. push) no está documentado públicamente. El resultado se recibe preferentemente por webhook ("notificaciones de integración", mecanismo recomendado pero cuyo payload no se detalla en la página revisada) o, alternativamente, por polling de `GET /point/integration-api/payment-intents/{id}` (solo disponible durante los 3 meses posteriores a su creación; un estado `confirmation_required` es final y obliga a verificar el pago en el propio equipo antes de entregar el producto/servicio). **Esta ruta no requiere ningún dispositivo adicional de MBTickets**: el terminal es autónomo y solo necesita su propia conectividad a internet (WiFi/datos). Corresponde exactamente al transporte "Cloud API" ya modelado en `09-feature-pos-payment-terminal-integration.md` §5.

**(b) "Smart Apps" (aplicaciones instaladas directamente en el terminal):** dado que Point Smart es en sí mismo un dispositivo Android, Mercado Pago permite compilar e instalar aplicaciones Android propias (nativo, o Ionic/Flutter para "Mini Apps") directamente en el terminal, ya sea como "Main App" (reemplaza la interfaz principal del equipo) o como "Mini App" (instalable desde el marketplace del propio dispositivo). La app propia genera la orden a través del SDK on-device y el gestor de pagos nativo de Mercado Pago la procesa, devolviendo un callback de resultado; también permite imprimir recibos personalizados y usar cámara/Bluetooth del propio equipo (lector de códigos, teclados, impresoras). **Patrón no contemplado originalmente en doc 09:** en vez de un kiosko separado (Vanexa) más un terminal pareado, el propio terminal Point Smart podría actuar como el único dispositivo del punto de venta, corriendo una app de MBTickets embebida en él. Fuera de alcance para v1 (el diseño vigente asume kiosko Vanexa + terminal de pago separados), pero queda documentado como opción arquitectónica futura para puntos de venta muy simples (solo cobro, sin experiencia de kiosko conversacional).

**(c) Integración "app a app" vía Intent/Deep Link** (patrón de referencia público en el repositorio `mercadopago/point-android_integration`): a diferencia de lo asumido originalmente en doc 09, este patrón **no es un SDK embebible que dialogue directo por Bluetooth con el hardware del lector**. Es, en cambio, una **delegación app-a-app**: la app oficial "Mercado Pago Point" debe estar instalada en el mismo dispositivo Android que la app propia; la app propia invoca un `Intent` (acción `com.mercadopago.PAYMENT_ACTION`, con extras de monto, descripción y credenciales de aplicación) o un deep link equivalente, y es la app oficial de Mercado Pago la que efectivamente dialoga con el hardware del lector (vía su propio pairing Bluetooth interno, no expuesto a terceros); el resultado vuelve por `onActivityResult` o por parámetros de la URL de retorno (`result_status`, `payment_id`, y, en caso de falla, códigos de error como cancelación del usuario, tarjeta expirada o pago rechazado). **Esto ajusta una premisa de doc 09 §5/§7:** para Mercado Pago, la app companion Android no necesita (ni puede) implementar su propio pairing Bluetooth con el lector — debe en cambio invocar la app oficial de Mercado Pago Point instalada en el mismo tablet, lo cual es más simple de construir pero introduce una dependencia operativa: esa app oficial debe estar instalada y vigente en cada kiosko que use este patrón.

**Conectividad:** en el patrón (a), el terminal necesita solo su propia conectividad a internet, independiente de la red local del kiosko. En el patrón (c), ambas apps (la oficial de Mercado Pago y la companion propia) deben coexistir en el mismo dispositivo Android físico — no es "pairing entre dos dispositivos", sino coexistencia de dos apps en el mismo hardware.

**Brechas — no confirmado por documentación pública:** mecanismo exacto de entrega del payment-intent al terminal en (a) (polling interno del equipo vs. push); vigencia y vía de integración API específica para Point Mini (lector Bluetooth económico sin pantalla) como dispositivo autónomo distinto de Point Smart/Plus/Air; estructura exacta del payload de las notificaciones/webhooks de (a).

### 7.3 Implicancias para el diseño de la integración de punto de venta físico

Estos hallazgos ajustan — sin invalidar — el diseño ya especificado en `09-feature-pos-payment-terminal-integration.md`:

1. **Mercado Pago + Point Smart vía Cloud API (patrón a) es, en la práctica, el conector más simple de construir para v1:** no requiere companion Android ni puente Raspberry Pi, solo llamadas HTTPS desde el backend más un webhook/polling. Se recomienda que sea el **primer conector construido**, coherente con la prioridad #1 ya decidida (doc 09 §1.1).
2. **Para Transbank, no existe hoy un SDK Android nativo del fabricante.** La Opción A ("Android-paired terminal") solo sería viable vía USB-OTG ejecutando el SDK Java/Node dentro de la companion app — en la práctica, equivalente en complejidad de cableado físico a la Opción B. Se recomienda priorizar directamente el **puente Raspberry Pi (Opción B)** para Transbank en v1, usando el SDK oficial Python o Node.js del protocolo "POS Integrado".
3. **Para Mercado Pago, si en el futuro se requiere un lector Bluetooth económico pareado directo al kiosko** (en vez de un terminal autónomo como Point Smart), **la vía pública de integración no es un SDK Bluetooth propio**, sino delegar el cobro a la app oficial "Mercado Pago Point" instalada en el mismo tablet, vía Intent/deep link. La companion app Android de MBTickets para Mercado Pago, de llegar a construirse, debe diseñarse como un **invocador** de la app oficial, no como un cliente Bluetooth directo del hardware del lector.
4. Se identifica un tercer patrón de integración de Mercado Pago ("Smart Apps" embebidas en el propio terminal Point Smart) que doc 09 no contempla; queda documentado como opción arquitectónica futura, fuera de alcance de v1.

### 7.4 Fuentes consultadas

- Transbank — [POS Integrado · Transbank Developers](https://www.transbankdevelopers.cl/documentacion/posintegrado)
- Transbank — [POS Fijo Integrado · Portal Público](https://publico.transbank.cl/productos-y-servicios/soluciones-para-ventas-presenciales/pos-fijo-integrado)
- Transbank — [¿Qué es POS Multicódigo Integrado? · Centro de Ayuda](https://ayuda.transbank.cl/que-es-pos-multicodigo-integrado)
- Mercado Pago — [Point — Overview](https://www.mercadopago.com.mx/developers/en/docs/mp-point/overview)
- Mercado Pago — [Point (legacy) — Payment processing](https://www.mercadopago.com.ar/developers/en/docs/mp-point-legacy/integration-configuration/integrate-with-pdv/payment-processing)
- Mercado Pago — [Point — Payment processing (endpoints vigentes)](https://www.mercadopago.com.mx/developers/en/docs/mp-point/integration-configuration/integrate-with-pdv/payment-processing)
- Mercado Pago — [Smart Apps — Overview](https://www.mercadopago.com.mx/developers/en/docs/smartapps/overview)
- Mercado Pago — [point-android_integration (repositorio de referencia) · GitHub](https://github.com/mercadopago/point-android_integration)
