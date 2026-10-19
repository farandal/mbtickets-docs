# MBTickets — Modelo entidad-relación

Este documento describe las entidades del sistema, qué representan en el negocio y cómo se
relacionan. Se divide en áreas, de la cuenta del cliente a la entrada validada en la puerta.

- Las entidades **del core** (`dash-backend`) son compartidas con los otros productos de la plataforma:
  cuentas, usuarios, suscripciones, pasarelas de pago y auditoría.
- Las entidades **del dominio** (`mbtickets-backend-domain`) son propias de MBTickets. Sus tablas
  llevan el prefijo `mbt_`.

Todas las entidades del dominio:

- usan UUID como clave;
- se borran de forma lógica (`deleted_at`);
- llevan `tenant_id` (el local dueño) y `tenancy_id` (la cadena). Un usuario solo ve las filas de su
  alcance: una fila ajena no existe para él.

## Vista general

```mermaid
flowchart LR
    subgraph Cuenta["Cuenta (core)"]
        TY["Tenancy<br/>cadena"] --> TN["Tenant<br/>local"]
        TY --> SUB["Suscripción<br/>plan"]
        TN --> US["Usuarios"]
    end
    subgraph Recinto["Recinto"]
        RM["Sala"] --> SC["Sector"] --> ST["Butaca"]
    end
    subgraph Cartelera["Cartelera"]
        CAT["Categoría"] --> ET["Evento"] --> ES["Función"]
    end
    subgraph Venta["Venta"]
        IU["Unidad de inventario"] --> BK["Reserva"] --> TK["Entrada"]
        BK --> RF["Reembolso"]
    end
    subgraph Pago["Pago (core)"]
        GW["Pasarela"] --> TX["Transacción"]
    end
    subgraph Puerta["Control de acceso"]
        SK["Clave de firma"] --> TK
        TK --> VL["Registro de validación"]
    end

    TN --> RM
    RM --> ET
    ES --> IU
    TX -. paga .-> BK
```

## 1. Cuenta, locales y usuarios (core)

```mermaid
erDiagram
    TENANCY ||--o{ TENANT : "tiene locales"
    TENANCY ||--o{ TENANCY_SUBSCRIPTION : "contrata"
    SUBSCRIPTION_PLAN ||--o{ TENANCY_SUBSCRIPTION : "define"
    TENANCY ||--o{ USER : "administradores"
    TENANT ||--o{ USER : "personal"
    USER }o--o{ ROLE : "tiene"

    TENANCY {
        uuid id PK
        string public_name
        string legal_name
        string status "trial, active, suspended..."
        timestamp trial_ends_at
        json settings
    }
    TENANT {
        uuid id PK
        uuid tenancy_id FK
        string name
        string slug
        json settings "incluye ajustes mbtickets_*"
    }
    TENANCY_SUBSCRIPTION {
        uuid tenancy_id FK
        int subscription_plan_id FK
        string status
        timestamp current_period_end
    }
    SUBSCRIPTION_PLAN {
        int id PK
        string name
        json metadata "limites: max_tenants, max_rooms, max_active_events"
    }
    USER {
        uuid id PK
        uuid tenancy_id FK
        uuid tenant_id FK
        string email
    }
    ROLE {
        string name "System, TenancyAdmin, Tenant, User"
    }
```

| Entidad | En pantalla | Qué representa |
|---|---|---|
| **Tenancy** | Cuenta / cadena | El cliente de MBTickets: la empresa que contrata el servicio. Puede operar uno o varios locales. Es la que paga la suscripción. |
| **Tenant** | Local (venue) | Un recinto: un teatro, un cine, un centro cultural. Es la unidad operativa: las salas, la cartelera, las ventas y las claves de firma de las entradas pertenecen a un local. Sus ajustes guardan los valores por defecto del local (tiempo entre funciones, política de cancelación). |
| **SubscriptionPlan** / **TenancySubscription** | Suscripción | El plan contratado y su período vigente. El plan fija los límites: cuántos locales, salas y eventos publicados a la vez. |
| **User** | Usuario | Una persona que entra al panel. El administrador de cadena se ata a la tenancy; el administrador de local y el personal, a un local. El comprador **no** es un usuario: compra como invitado. |
| **Role** | Rol | Qué puede hacer el usuario. **System**: la plataforma completa. **TenancyAdmin**: todos los locales de su cadena. **Tenant**: su local. **User**: boletería y puerta de su local. Cada endpoint de la API es un permiso. |

## 2. Recinto: salas, sectores y butacas

```mermaid
erDiagram
    TENANT ||--o{ MBT_ROOM : "tiene"
    MBT_ROOM ||--o{ MBT_ROOM_SECTOR : "se divide en"
    MBT_ROOM_SECTOR ||--o{ MBT_ROOM_SEAT : "contiene"

    MBT_ROOM {
        uuid id PK
        uuid tenant_id FK
        string name
        string mode "general_admission, numbered_seats"
        int capacity
        int session_buffer_minutes "null = el del local"
        boolean is_active
    }
    MBT_ROOM_SECTOR {
        uuid id PK
        uuid room_id FK
        string name
        string kind "general, seated"
        int capacity "solo general"
        decimal price "null = el del evento"
        int sort
    }
    MBT_ROOM_SEAT {
        uuid id PK
        uuid sector_id FK
        string row_label
        int number
        string label "A-1"
        string inamap_location_id "mapa de Inamap, futuro"
        boolean is_active
    }
```

| Entidad | En pantalla | Qué representa |
|---|---|---|
| **Room** (`mbt_rooms`) | Sala | Un espacio físico del local donde ocurren las funciones. Es la **plantilla** del inventario: todo lo que se vende sale de su forma. Define el tiempo mínimo entre dos funciones (limpieza, montaje); dos funciones no pueden solaparse en la misma sala. Una sala con eventos no se borra: se desactiva. |
| **RoomSector** (`mbt_room_sectors`) | Sector | Una zona de la sala con su propio precio ("Platea", "Balcón"). Puede ser **de admisión general** (un cupo: se vende por cantidad) o **de butacas numeradas** (se vende cada asiento). Una sala sin sectores se vende como un único bloque general. |
| **RoomSeat** (`mbt_room_seats`) | Butaca | Un asiento concreto de un sector numerado: fila y número ("A-1"). Se generan por rangos de filas. |

## 3. Cartelera: categorías, eventos y funciones

```mermaid
erDiagram
    MBT_EVENT_CATEGORY ||--o{ MBT_EVENT_TYPE : "clasifica"
    MBT_ROOM ||--o{ MBT_EVENT_TYPE : "se presenta en"
    MBT_EVENT_TYPE ||--o{ MBT_EVENT_SESSION : "se programa en"
    MBT_ROOM ||--o{ MBT_EVENT_SESSION : "ocupa"

    MBT_EVENT_CATEGORY {
        uuid id PK
        string slug
        string name
        boolean is_active
    }
    MBT_EVENT_TYPE {
        uuid id PK
        uuid room_id FK
        uuid event_category_id FK
        string name
        int duration_minutes
        decimal base_price
        string currency
        string status "draft, published, retired"
        timestamp sales_end_at "fecha maxima de compra"
        int sales_cutoff_minutes "cierre antes de cada funcion"
        boolean cancellation_allowed "null = el del local"
        int cancellation_deadline_hours
        decimal refund_percentage
        decimal cancellation_fee
        boolean partial_cancellation_allowed
        jsonb custom_attributes "libre del organizador"
    }
    MBT_EVENT_SESSION {
        uuid id PK
        uuid event_type_id FK
        uuid room_id FK
        timestamptz starts_at
        timestamptz ends_at
        int capacity "null = la del evento"
        string status "draft, published, cancelled"
    }
```

| Entidad | En pantalla | Qué representa |
|---|---|---|
| **EventCategory** (`mbt_event_categories`) | Categoría | Una etiqueta de descubrimiento común a toda la plataforma ("Música", "Arte y Cultura"), para filtrar la cartelera. La administra la plataforma, no cada local. |
| **EventType** (`mbt_event_types`) | Evento | **Lo que se presenta**: una obra, una película, un concierto. Reúne lo que comparten todas sus funciones: sala, duración, precio base y moneda, cierre de la venta y política de cancelación. Nace en borrador y no aparece en la cartelera hasta publicarlo. `custom_attributes` guarda datos libres del organizador (elenco, clasificación) que ningún proceso interpreta. |
| **EventSession** (`mbt_event_sessions`) | Función | **Una presentación** del evento en un día y una hora, en su sala. Es la unidad que se vende: cada función tiene su propio inventario. También se publica por separado. Cancelarla anula todas sus ventas y las reembolsa. |

## 4. Inventario y ventas

```mermaid
erDiagram
    MBT_EVENT_SESSION ||--o{ MBT_INVENTORY_UNIT : "tiene a la venta"
    MBT_ROOM_SECTOR ||--o{ MBT_INVENTORY_UNIT : "origina"
    MBT_ROOM_SEAT ||--o| MBT_INVENTORY_UNIT : "origina, por funcion"
    MBT_EVENT_SESSION ||--o{ MBT_BOOKING : "recibe"
    MBT_BOOKING ||--|{ MBT_BOOKING_ITEM : "se compone de"
    MBT_INVENTORY_UNIT ||--o{ MBT_BOOKING_ITEM : "se reserva en"
    MBT_BOOKING ||--o{ MBT_TICKET : "emite"
    MBT_BOOKING_ITEM ||--|{ MBT_TICKET : "una por cupo"
    MBT_BOOKING ||--o{ MBT_REFUND : "devuelve"

    MBT_INVENTORY_UNIT {
        uuid id PK
        uuid event_session_id FK
        uuid sector_id FK
        uuid seat_id FK
        string kind "seat, ga"
        string label "Platea A-1"
        int capacity
        int held "en pago"
        int sold
        int blocked
        int available "capacity - held - sold - blocked"
        decimal unit_price
    }
    MBT_BOOKING {
        uuid id PK
        uuid event_session_id FK
        string code "MBT-3A53XC"
        string status "held, payment_pending, confirmed, expired, released, cancelled"
        string channel "online, box_office, kiosk, api"
        string buyer_name
        string buyer_email
        string access_token_hash "comprador invitado"
        decimal total_amount
        string payment_method
        string payment_reference
        timestamp expires_at
        string cancellation_origin
        decimal refunded_amount
        string refund_status
    }
    MBT_BOOKING_ITEM {
        uuid id PK
        uuid booking_id FK
        uuid inventory_unit_id FK
        string label
        int quantity
        decimal unit_price
    }
    MBT_TICKET {
        uuid id PK
        uuid booking_id FK
        uuid booking_item_id FK
        int sequence
        string label
        string status "valid, redeemed, void"
        string kid "clave que la firmo"
        timestamp redeemed_at
        timestamp voided_at
    }
    MBT_REFUND {
        uuid id PK
        uuid booking_id FK
        decimal amount
        string method "checkout_gateway, cash, card_terminal, transfer"
        string status "pending, refunded, failed, manual_required"
        string kind "cancellation, late_payment"
        string origin "buyer, box_office, organizer, admin, system"
        uuid checkout_transaction_id
        string reference
    }
```

| Entidad | En pantalla | Qué representa |
|---|---|---|
| **InventoryUnit** (`mbt_inventory_units`) | Disponibilidad | **Lo que se puede vender de una función.** Se genera desde la sala al crear la función: una unidad por butaca (capacidad 1) o una por sector general (capacidad = el cupo). Sus contadores dicen cuánto está disponible, retenido mientras alguien paga, vendido o bloqueado. Es la única fuente de verdad de la disponibilidad: todos los canales descuentan de aquí, y la base de datos impide que la suma no cuadre. Así nunca se vende una butaca dos veces. |
| **Booking** (`mbt_bookings`) | Reserva / venta | **Una compra**, de cualquier canal (app, boletería, kiosko o API externa). Nace **retenida**: aparta el inventario por unos minutos. Con el pago pasa a **vendida**; si no se paga, **vence** o se **libera**. Una vendida puede **cancelarse** completa o por entradas. Guarda al comprador (invitado, sin cuenta), el medio de pago y el estado de sus reembolsos. El comprador la consulta con un token secreto que solo tiene su navegador; en la base se guarda solo su huella. |
| **BookingItem** (`mbt_booking_items`) | Línea de la reserva | Qué se compró y a qué precio: una butaca, o N cupos de un sector general. Congela el precio del momento de la compra. |
| **Ticket** (`mbt_tickets`) | Entrada | **El derecho a entrar una vez**: una por cada cupo vendido. Su QR está firmado y no se guarda: se recalcula. Lo que decide si alguien entra es su estado: **válida**, **usada** (el primer escaneo la marca) o **anulada** (al cancelar). |
| **Refund** (`mbt_refunds`) | Reembolso | **Una devolución de dinero** de una reserva, por el mismo medio con que se pagó. La generan una cancelación o un pago aprobado tarde (que no llegó a convertirse en venta). Las de pasarela online se ejecutan solas; las de efectivo o transferencia esperan que el personal las marque como devueltas, con una referencia. Su estado es independiente del de la reserva: una reserva cancelada con un reembolso fallido sigue con sus entradas anuladas. |

## 5. Pagos online (core: CheckoutCore)

```mermaid
erDiagram
    SYSTEM_CHECKOUT_GATEWAY ||--o{ CHECKOUT_GATEWAY : "se instancia como"
    TENANCY ||--o{ CHECKOUT_GATEWAY : "configura"
    CHECKOUT_GATEWAY ||--o{ CHECKOUT_GATEWAY_TENANT : "se habilita en"
    TENANT ||--o{ CHECKOUT_GATEWAY_TENANT : "usa"
    CHECKOUT_GATEWAY ||--o{ CHECKOUT_GATEWAY_TRANSACTION : "procesa"
    MBT_BOOKING ||--o{ CHECKOUT_GATEWAY_TRANSACTION : "se paga con (payable)"
    CHECKOUT_GATEWAY_TRANSACTION ||--o{ MBT_REFUND : "se devuelve en"

    SYSTEM_CHECKOUT_GATEWAY {
        int id PK
        string name "DashTest, Transbank Webpay, Mercado Pago"
        string class "proveedor"
        boolean is_active
    }
    CHECKOUT_GATEWAY {
        int id PK
        uuid tenancy_id FK
        int system_checkout_gateway_id FK
        string name
        json connection_params "credenciales del proveedor, cifradas"
        boolean active
    }
    CHECKOUT_GATEWAY_TENANT {
        int checkout_gateway_id FK
        uuid tenant_id FK
        boolean is_default
    }
    CHECKOUT_GATEWAY_TRANSACTION {
        uuid id PK
        int checkout_gateway_id FK
        uuid tenant_id FK
        string payable_type "Booking"
        uuid payable_id
        decimal amount
        string status "pending, paid, rejected, failed, partially_refunded, refunded"
        string external_session_id "token o id del proveedor"
        json data "historial de reembolsos"
    }
```

| Entidad | En pantalla | Qué representa |
|---|---|---|
| **SystemCheckoutGateway** | Catálogo de pasarelas | Los proveedores de pago que la plataforma soporta: DashTest (banco de demostración), Transbank Webpay y Mercado Pago. |
| **CheckoutGateway** | Pasarela | La cuenta de una cadena en uno de esos proveedores, con sus credenciales. Es la que cobra el dinero de las ventas online. |
| **CheckoutGatewayTenant** | Pasarela del local | En qué locales se usa cada pasarela, y cuál es la predeterminada. Sin una pasarela activa, el local solo vende en la boletería. |
| **CheckoutGatewayTransaction** | Transacción | **Un intento de pago** en la pasarela. Apunta a lo que paga (en MBTickets, una reserva). Cuando el proveedor la aprueba, la reserva pasa a vendida; si la rechaza, el inventario se libera. Lleva el historial de reembolsos, totales o parciales. Es compartida con KitchnTabs, que la usa para pagar pedidos. |

## 6. Control de acceso

```mermaid
erDiagram
    TENANT ||--o{ MBT_SIGNING_KEY : "firma con"
    MBT_SIGNING_KEY ||--o{ MBT_TICKET : "firma (kid)"
    MBT_TICKET ||--o{ MBT_VALIDATION_LOG : "se escanea en"
    MBT_EVENT_SESSION ||--o{ MBT_VALIDATION_LOG : "registra"

    MBT_SIGNING_KEY {
        uuid id PK
        uuid tenant_id FK
        string kid
        string algorithm "Ed25519"
        string public_key "publica: la descargan los lectores"
        text secret_key "cifrada"
        boolean is_active
    }
    MBT_VALIDATION_LOG {
        uuid id PK
        uuid ticket_id FK
        uuid event_session_id FK
        string result "accepted, already_redeemed, void, invalid, wrong_session, unknown_ticket, duplicate_offline"
        string gate
        boolean offline
        boolean needs_review
        timestamp scanned_at
    }
```

| Entidad | En pantalla | Qué representa |
|---|---|---|
| **SigningKey** (`mbt_signing_keys`) | — (interna) | El par de claves con que un local firma sus entradas. La pública se entrega a los lectores de la puerta, para que verifiquen un QR **sin conexión**. La privada se guarda cifrada y nunca sale del servidor. Rotarla no invalida las entradas ya emitidas. |
| **ValidationLog** (`mbt_validation_logs`) | Lecturas de la puerta | Cada escaneo de una entrada, aceptado o no: en qué puerta, cuándo y con qué resultado. Incluye los escaneos hechos sin red, que se sincronizan al volver la conexión. Si una entrada entró dos veces en lectores distintos durante un corte, se marca para revisión. |

## 7. Auditoría (core)

| Entidad | Qué representa |
|---|---|
| **Activity log** (`activity_log`) | El registro de quién hizo qué y cuándo, con su motivo. En MBTickets se audita: cada cancelación (origen, motivo, entradas, monto devuelto), cada reembolso resuelto a mano, los reembolsos de pagos tardíos y la cancelación de funciones. |

## 8. Ciclos de vida

| Entidad | Estados | Cómo cambia |
|---|---|---|
| Evento | `draft` → `published` → `retired` | Solo por las acciones Publicar / Volver a borrador / Retirar. Publicar respeta el límite del plan. |
| Función | `draft` → `published` → `cancelled` | Publicar la pone a la venta; cancelarla anula y reembolsa sus ventas. |
| Reserva | `held` → `payment_pending` → `confirmed` → `cancelled`; o `expired` / `released` | Solo la cambia el motor de inventario. |
| Entrada | `valid` → `redeemed` o `void` | El primer escaneo la marca usada; una cancelación la anula. |
| Reembolso | `pending` → `refunded`, `failed` o `manual_required` | La pasarela, un reintento o la resolución manual con referencia. |

## Para saber más

- Requerimientos: [`../requirements/02-arquitectura.md`](../requirements/02-arquitectura.md).
- Detalle técnico por feature (flujos, API, decisiones): [`../features`](../features/README.md).
- Manual del administrador de cadena: [`../user/TenancyAdminManual.md`](../user/TenancyAdminManual.md).
