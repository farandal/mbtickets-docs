# MBTickets — Resumen Ejecutivo

> Documento 1 de 9 del levantamiento de requerimientos de MBTickets.
> Serie completa: 01-resumen-ejecutivo · 02-arquitectura · 03-requerimientos-tecnicos · 04-requerimientos-funcionales · 05-requerimientos-no-funcionales · 06-historias-de-usuario · 07-features-comunes-y-flujos · 08-feature-vanexa-ai-kiosk-integration · 09-feature-pos-payment-terminal-integration
> Los documentos 01–07 están en español; 08 y 09 son addenda de features escritos en inglés (su numeración de FR/HU continúa la de 04 y 06, ver §6).

## 0. Qué es MBTickets

MBTickets es una **Plataforma Híbrida de Venta y Validación de Entradas y Sistema de Gestión de Eventos (EMS)**, capaz de operar desde un simposio departamental de 150 butacas hasta un festival de 20.000 entradas que se agota en minutos. Se construye sobre el mismo framework interno **Dash-Admin** (core Laravel + React) que ya sostiene a Vanexa, KitchnTabs e Inamap, siguiendo su patrón estricto de separación **core / dominio**: el core nunca se modifica; MBTickets es un nuevo repositorio de dominio (`mbtickets-backend-domain` / `mbtickets-frontend`) montado sobre él.

## 1. Soluciones

- Descongestión de filas
- Autoatención

## 2. Problemas

1. **Inventario sobre-comprometido (race conditions):** lecturas/escrituras de disponibilidad sin aislamiento transaccional estricto permiten que canales simultáneos confirmen la misma última unidad. No se resuelve en el cliente ni en el navegador — debe resolverse en la capa de transacción de base de datos.

2. **Fragmentación administrativa:** planillas, hilos de correo, libros de caja de POS separados y listas en papel en la puerta generan registros divergentes a medida que el evento escala.

3. **Verificación de entradas insegura y lenta:** PDFs estáticos o códigos de barra sin firma pueden capturarse, reenviarse y reutilizarse; el staff de entrada o bien revisa listas manualmente (cuello de botella) o inspecciona visualmente PDFs (fraude), agravado por el riesgo de que el Wi-Fi del recinto caiga durante el check-in.

## 3. Retos de ingeniería

**concurrencia unificada de inventario entre canales híbridos**. El sistema debe garantizar que un comprador online y un cliente walk-up en un POS físico o kiosko de autoservicio nunca puedan comprar el mismo asiento ni agotar el último cupo de admisión general al mismo tiempo — y que los torniquetes de entrada puedan validar entradas de forma rápida y segura incluso durante cortes de internet locales.

En tanto el inventario entre canales híbridos, debe ser resuelto por la capacidad de interoperabilidad de la plataforma de implementar en su api los endpoints para actualizar la disponibilidad de asientos de forma externa y webhooks necesarios para notificar a sistemas externos; todos estos eventos deben quedar auditables. Esta capacidad se especifica en FR-24 (`04-requerimientos-funcionales.md`) y NFR-07 (`05-requerimientos-no-funcionales.md`); toda escritura externa pasa por el mismo motor transaccional de NFR-01, nunca por un camino paralelo.

## 4. Estrategia de construcción: core + dos referencias de dominio

MBTickets se levanta evaluando dos proyectos hermanos ya construidos sobre Dash-Admin:

- **Inamap** se usa como **referencia de arquitectura de plataforma**: patrón de montaje core/dominio, multi-tenancy (jerarquía System → Tenancy → Tenant → User del core; los dominios operan sobre Tenancy → Tenant), RBAC vía Spatie + seeders JSON sobre roles existentes, convención de PK UUID + soft delete + `tenancy_id`/`tenant_id` denormalizados, patrón de guardado atómico de "documento compuesto" en una sola transacción, y un mecanismo de token opaco hasheado con TTL para credenciales de un solo uso. Es un dominio maduro en estos aspectos de plataforma, pero **no resuelve concurrencia de inventario** (no la necesita: es un motor de mapas, no de ventas) ni firma criptográfica de credenciales.
- **KitchnTabs** se evalúa como **candidato de reutilización de módulos de dominio de e-commerce**: su capa de pagos (gateways intercambiables Flow/Transbank/MercadoPago/Interno), su motor de notificaciones multicanal por cambio de estado, su infraestructura de broadcasting/colas (Reverb + Redis/Horizon), su panel admin y sus reportes de embudo/tiempos de ciclo son reutilizables con bajo esfuerzo, renombrando conceptos (`Tab`→`Booking`, `Order`→`Ticket`). Sin embargo la logica de dominio de MBTickets debe resolver el problema de cupos/asientos (nunca se bloquea ni se descuenta atómicamente al pedir), y el mecanismo de QR/sesión self-service tiene fallas de seguridad documentadas (sesiones creadas sin escaneo real de QR) que no deben heredarse, y cuyas mejoras deben implementarse en ambos sistemas. 

DashAdmin provee autenticación, RBAC, multi-tenancy, notificaciones, panel admin, pero el desarrollo de los componentes más críticos de la logica de MBTickets deben desarrollarse: (a) el motor transaccional de bloqueo de inventario serializable, y (b) la firma criptográfica de credenciales con validación offline de un solo uso. Ambos se diseñan y construyen de cero en la capa de dominio de MBTickets, siguiendo el estilo arquitectónico ya validado (servicios PHP puros, testeables sin Laravel, como `RouteEngine`).

## 5. Alcance v1 (propuesto)

- Descubrimiento público de eventos sin autenticación.
- Reserva/compra de entradas (asiento numerado o admisión general) por canal online y POS/kiosko, sobre un único inventario.
- Emisión de credencial QR firmada criptográficamente por entrada confirmada.
- Validación de entrada en torniquete/scanner de mano, de un solo uso, con tolerancia a cortes de internet locales.
- Roles Visitante (Guest, sin autenticar), Comprador (usuario final autenticado), Organizer, Tenancy Admin y Administrator con RBAC estricto; los dispositivos (POS, kiosko, taquilla, torniquete) operan con credenciales propias, no como usuarios humanos (FR-02.3, FR-11).
- Notificaciones asíncronas de confirmación/cancelación vía el broker de eventos.
- Cancelación y reembolso de reservas con política configurable, y anulación de la credencial QR (FR-25).
- Pago del canal online sobre pasarelas intercambiables, conciliado con el hold de inventario (FR-26); pago presencial con terminal físico según `09-feature-pos-payment-terminal-integration.md` (FR-20 a FR-23).
- API externa de inventario y webhooks salientes auditables (FR-24).
- Canal de kiosko conversacional Vanexa según `08-feature-vanexa-ai-kiosk-integration.md` (FR-19).

**Fuera de alcance en esta etapa:**

- Reventa/transferencia de entradas entre usuarios.
- Marketplace de eventos de terceros.
- Marketplaces o plataformas externas de ticketing como *producto* propio de MBTickets. La API y los webhooks de FR-24 sí permiten que un sistema externo autorizado opere sobre el inventario de un venue, pero no se construyen conectores específicos de terceros en v1.

> Nota: el agente conversacional de voz en kiosko (Vanexa) figuraba aquí como fuera de alcance; fue incorporado como canal en `08-feature-vanexa-ai-kiosk-integration.md` (FR-19).

## 6. Cómo leer el resto de la serie

| Documento | Contenido |
|---|---|
| `02-arquitectura.md` | Arquitectura de referencia (core Dash-Admin), qué se hereda vs. qué se construye, modelo de dominio propuesto |
| `03-requerimientos-tecnicos.md` | Stack tecnológico, convenciones de datos, librerías nuevas requeridas, estrategia de pruebas |
| `04-requerimientos-funcionales.md` | FR-01 a FR-10+ detallados |
| `05-requerimientos-no-funcionales.md` | NFR-01 a NFR-06+ detallados, con enfoque técnico propuesto |
| `06-historias-de-usuario.md` | Historias de usuario por rol con criterios de aceptación |
| `07-features-comunes-y-flujos.md` | Features reutilizadas vs. nuevas, diagramas Mermaid de los flujos principales |
| `08-feature-vanexa-ai-kiosk-integration.md` | Addendum (EN): canal de kiosko conversacional Vanexa. FR-19, HU-19 a HU-19e |
| `09-feature-pos-payment-terminal-integration.md` | Addendum (EN): pago presencial con terminal físico (capacidad del core). FR-20 a FR-23, HU-20 a HU-26b |

Numeración: FR-01 a FR-18 y FR-24 a FR-26 viven en `04`; FR-19 a FR-23 en `08`/`09`. HU-01 a HU-18 y HU-27 en adelante viven en `06`; HU-19 a HU-26b en `08`/`09`.
