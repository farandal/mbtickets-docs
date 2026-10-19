# MBTickets — Features Comunes y Flujos (con diagramas Mermaid)

> Documento 7 de 9. Consolida la decisión de reutilización módulo por módulo y los flujos end-to-end principales.
> Modelo de referencia: Tenancy (cadena) → Tenant = Venue → Sala → EventType → EventSession (ver `02-arquitectura.md` §4). Mapas de Sala: integración opcional con Inamap (§4.3).

## 1. Matriz de reutilización final

| Módulo/Feature | Fuente | Veredicto | Notas |
|---|---|---|---|
| Autenticación (Sanctum, OAuth) | Core | ✅ Reutilizar tal cual | |
| RBAC 2 capas (rutas + policies) | Core | ✅ Reutilizar tal cual | Solo se agregan permisos `mbtickets.*` |
| Multi-tenancy (Tenancy→Tenant=Venue) + `scopeVisibleTo` | Core + patrón Inamap | ✅ Reutilizar patrón | `Tenant` se reutiliza con significado de negocio "Venue"; Tenancy Admin ve todos sus Tenants/venues |
| Billing/suscripciones + límites por plan | Core + patrón Inamap/KitchnTabs | ✅ Reutilizar | Conteo propio del dominio (`MbTicketsPlanLimits`), incluye límite de nº de Salas por venue |
| Pasarelas de pago (Flow/Transbank/Rebill/Interno) | KitchnTabs (`Services/Payments`) | ✅ Reutilizar arquitectura de gateway intercambiable | Genérico, no depende de comida |
| Notificaciones multicanal + websockets + colas | Core (Reverb/Horizon) | ✅ Reutilizar infraestructura | El **patrón de invocación** debe cambiar: eventos desacoplados, no llamada síncrona directa |
| Panel admin genérico (`dash-auto-admin`) | Core | ✅ Reutilizar tal cual | |
| DeviceCore (heartbeat, comandos, presencia) | Core | ✅ Reutilizar como base de POS/kiosko/boletería/torniquete | Falta lógica de dominio de validación de tickets encima |
| Generación visual de QR (`endroid/qr-code`) | Core | ✅ Reutilizar solo el render | Sin firma — se envuelve con firma propia |
| Auditoría (Activitylog + Revisionable) | Core | ✅ Reutilizar tal cual | |
| **Mapa de asientos de Sala** | **Inamap (sistema externo)** | 🔗 **Integración, no reutilización de código** | El mapa se crea y vive en Inamap; MBTickets solo importa/sincroniza `Location` como `InventoryUnit`. Ver §2 y Flujo 7 |
| Patrón de token opaco hasheado + TTL | Inamap (`RouteSessionService`) | ✅ Reutilizar patrón | Para el canal de entrega del QR (no para su verificación offline) |
| Reportes de embudo/throughput | KitchnTabs (`TabsFunnelReport`, `TabsThroughputReport`) | ✅ Adaptar | Embudo de venta y de check-in, desglosado por Sala |
| Impresión térmica ESC/POS | KitchnTabs (`kitchntabs-python-service`) | ✅ Reutilizar tal cual | Para impresión física de entradas en taquilla/POS |
| Motor de locking/concurrencia de inventario | — | 🆕 Construir de cero | Sin precedente en ningún sistema hermano (NFR-01); corre 100% en MBTickets, independiente de Inamap |
| Entidad Sala + EventType + ScheduleRule + DateOverride + disponibilidad | — | 🆕 Construir de cero | `TenantSchedule` del core solo resuelve horario de tienda |
| EventCategory (tag genérico) + custom_attributes (JSON libre) por EventType | — | 🆕 Construir de cero | Tag plano de descubrimiento (ej. Entretenimiento/Música/Deportes), independiente del tipo de venue/sala; no ramifica locking, QR ni notificaciones (FR-18) |
| Validación de no-solapamiento de funciones por Sala | — | 🆕 Construir de cero | Regla de calendario, distinta del locking de inventario |
| Firma criptográfica Ed25519 + validación offline single-use | — | 🆕 Construir de cero | Ningún sistema hermano firma ni valida offline |
| Eventos de dominio desacoplados (`BookingConfirmed`, `TicketValidated`) | — | 🆕 Construir de cero (patrón Job/Listener del core como molde) | KitchnTabs usa Notifications acopladas, no Integration Events |
| Flujo de venta POS/kiosko/boletería de entradas | — | 🆕 Construir de cero | KitchnTabs solo tiene kiosko de *pedido*, Inamap solo de *consulta* |
| API externa de inventario + webhooks salientes firmados (FR-24) | — | 🆕 Construir de cero | Sin precedente en el core ni en hermanos. Las escrituras reutilizan el motor de NFR-01; la entrega corre sobre Horizon. Ver Flujo 9 |
| Cancelación, anulación de QR (`VOID`) y reembolso (FR-25) | — + `refundPayment()` de KitchnTabs + `refund()` de doc 09 | 🆕 Construir de cero (reutiliza los contratos de reembolso) | La lista de `VOID` hacia dispositivos offline es nueva. Ver Flujo 8 |
| Pago del canal online conciliado con el hold (FR-26) | KitchnTabs (`CheckoutGateway`) | ✅ Reutilizar la pasarela, 🆕 construir la conciliación con el hold | `completeTransaction()` como patrón de liquidación idempotente |
| Pago presencial con terminal físico (FR-20 a FR-23) | Capacidad nueva del **core** (`09-feature-pos-payment-terminal-integration.md`) | 🆕 Se construye en el core, MBTickets la consume | Ni MBTickets ni KitchnTabs tienen precedente |
| Canal de kiosko conversacional (FR-19) | Vanexa (`08-feature-vanexa-ai-kiosk-integration.md`) | 🔗 Integración | Nuevo canal sobre el mismo motor de reservas, no un motor nuevo |
| Catálogo de menú/productos, Stock de ingredientes, KDS, agente de voz con personaje | KitchnTabs | ❌ No reutilizar | Específico del dominio comida, sin relación con cupos de evento (el canal conversacional de Vanexa se integra aparte, ver fila anterior) |

## 2. Jerarquía de dominio: Venue multi-sala (y mapa opcional vía Inamap)

```mermaid
flowchart TD
    TC[Tenancy: Cadena/Organización] --> T1[Tenant = Venue: Cine Hoyts Parque Arauco]
    TC --> T2[Tenant = Venue: Teatro Municipal]
    TC --> T3[Tenant = Venue: Estadio Nacional]

    T1 --> S1[Sala 1]
    T1 --> S2[Sala 2 IMAX]
    T1 --> S3[Sala 3 - sin mapa, solo GA]
    S1 --> E1A["EventSession: Avatar 3 - 15:00 (cat: Entretenimiento)"]
    S1 --> E1B[EventSession: Avatar 3 - 19:30]
    S2 --> E2A[EventSession: Dune P2 - 18:00]

    T2 --> S4[Escenario Principal]
    S4 --> E3A["EventSession: Hamlet - Sáb 21:00 (cat: Entretenimiento)"]
    S4 --> E3B["EventSession: Recital solista - Vie 20:30 (cat: Música)"]

    T3 --> S5[Cancha + Tribunas - sectorizada]
    S5 --> E4A["EventSession: Clásico Universitario (cat: Deportes)"]

    subgraph INAMAP[Sistema externo: Inamap]
      L1[Layout: Sala 1]
      L2[Layout: Sala 2 IMAX]
    end

    S1 -.asociada a.-> L1
    S2 -.asociada a.-> L2
```

**Lectura clave:** la validación de no-solapamiento (FR-16) se aplica entre `EventSession` de la **misma Sala**; salas distintas del mismo venue son completamente independientes y pueden correr en simultáneo. Sala 3 no tiene mapa asociado y opera en modo admisión general gestionado 100% dentro de MBTickets. La categoría (`EventCategory`, FR-18) es ortogonal a esta jerarquía: el mismo Escenario Principal del Teatro Municipal aloja tanto una obra (*Entretenimiento*) como un recital (*Música*) — la categoría describe el `EventType`, no el venue ni la sala que lo aloja.

## 3. Flujo 1 — Venta online (comprador)

```mermaid
sequenceDiagram
    actor Guest as Comprador
    participant FE as Frontend (web/app)
    participant API as API MBTickets
    participant INV as Inventario (Postgres, transacción) — por EventSession
    participant PAY as Pasarela de pago
    participant BUS as Message Broker (eventos)
    participant NOTIF as Servicio de Notificaciones

    Guest->>FE: Busca venue/sala/función y selecciona asiento(s)
    FE->>API: POST /bookings/hold (event_session_id, asientos/cantidad)
    API->>INV: Inicia transacción + lock/decremento condicional
    INV-->>API: Hold confirmado (expires_at)
    API-->>FE: Hold OK, checkout habilitado
    FE->>API: POST /bookings/confirm (datos de pago)
    API->>PAY: Procesa pago
    PAY-->>API: Pago aprobado
    API->>INV: Confirma venta (hold → vendido) en transacción
    INV-->>API: Commit OK
    API->>BUS: Publica evento BookingConfirmed
    API-->>FE: Reserva confirmada
    BUS->>NOTIF: Consume BookingConfirmed (async)
    NOTIF-->>Guest: Envía QR firmado (email/push)
```

## 4. Flujo 2 — Venta en POS / Kiosko / Boletería (walk-up, múltiples ventanillas)

```mermaid
sequenceDiagram
    actor Cliente as Cliente walk-up
    participant POS as Terminal POS/Kiosko/Taquilla
    participant API as API MBTickets
    participant INV as Inventario (misma fuente de verdad, por EventSession)
    participant PAY as Terminal de pago físico
    participant BUS as Message Broker

    Cliente->>POS: Selecciona venue/sala/función/asiento
    POS->>API: POST /bookings/hold (credencial de dispositivo, event_session_id)
    API->>INV: Lock/decremento condicional sobre el MISMO inventario de la EventSession
    alt Cupo disponible
        INV-->>API: Hold confirmado
        POS->>PAY: Cobra (tarjeta/efectivo)
        PAY-->>POS: Pago aprobado
        POS->>API: POST /bookings/confirm
        API->>INV: Confirma venta
        API->>BUS: Publica BookingConfirmed
        API-->>POS: Entrada emitida (QR impreso o enviado)
    else Sin cupo (ya vendido por otra ventanilla u online)
        INV-->>API: Rechazo (remaining < qty)
        API-->>POS: Venta rechazada, sin cobro
    end
    Note over POS,INV: Varias ventanillas de taquilla del mismo venue compiten por igual contra el mismo inventario — ninguna tiene prioridad ni copia local
```

## 5. Flujo 3 — Emisión y validación de entrada (con resiliencia offline y múltiples puertas)

```mermaid
sequenceDiagram
    participant API as API MBTickets
    participant KEY as Clave privada del tenant/venue
    participant Guest as Comprador
    participant SCAN as Torniquete / Scanner de mano (cualquier puerta)
    participant LOCAL as Estado local del dispositivo

    API->>KEY: Firma payload (ticket_id, event_session_id, asiento/sector, ts)
    API-->>Guest: QR = payload + firma Ed25519
    Note over SCAN: Clave pública del tenant ya cacheada vía DeviceCore, compartida entre todas las puertas del venue

    Guest->>SCAN: Presenta QR en cualquier puerta
    SCAN->>SCAN: Verifica firma localmente (sin red)
    alt Firma válida y no canjeado localmente
        SCAN->>LOCAL: Marca REDEEMED localmente
        SCAN-->>Guest: Acceso concedido (<500ms)
        opt Hay conectividad
            SCAN->>API: Sincroniza TicketValidated
        end
    else Firma inválida o ya canjeado (en esta u otra puerta)
        SCAN-->>Guest: Acceso denegado
    end

    Note over SCAN,API: Al reconectar tras un corte: SCAN sincroniza su log local de canjes con el servidor central y resuelve conflictos cross-puerta (anti-passback)
```

## 6. Flujo 4 — Eventos de dominio desacoplados

```mermaid
flowchart LR
    A[BookingConfirmed] --> Q[(Cola / Broker)]
    B[TicketValidated] --> Q
    C[BookingCancelled] --> Q
    Q --> L1[Listener: Notificación al comprador]
    Q --> L2[Listener: Actualización de analítica/reportes por Sala]
    Q --> L3[Listener: Log de auditoría]
    Q --> L4[Listener: Sincronización de dashboard Organizer]
```

## 7. Flujo 5 — Motor de locking de inventario (detalle del componente de mayor riesgo)

```mermaid
flowchart TD
    Start([Solicitud de reserva: N unidades de una EventSession]) --> Tx{Abrir transacción}
    Tx --> Check["Lock de fila / UPDATE condicional\nWHERE remaining >= N"]
    Check -->|remaining >= N| Decr[Decrementar remaining en N\nCrear Hold con expires_at]
    Check -->|remaining < N| Reject[Rechazo atómico total\nNinguna unidad se confirma]
    Decr --> Commit[Commit transacción]
    Reject --> Rollback[Rollback / sin cambios]
    Commit --> HoldOK([Hold confirmado, checkout habilitado])
    Rollback --> RejectOK([Respuesta de cupo insuficiente])

    HoldOK --> Timeout{Pago dentro de expires_at?}
    Timeout -->|Sí| ConfirmSale[Confirmar venta: Hold → Vendido]
    Timeout -->|No| ReleaseJob[Job programado libera el Hold\nremaining += N]
```

**Nota:** el diagrama muestra la variante de contador (admisión general o sector). Para asientos numerados la unidad es la fila de `InventoryUnit`: se bloquean todas las filas solicitadas en orden de id y pasan de `disponible` a `retenido`. Si alguna ya no está disponible, se rechaza la solicitud completa. El hold, su expiración y la liberación son los mismos (NFR-01, "Dos modelos de inventario bajo el mismo motor"). Si el comprador inició el pago, el hold pasa a `payment_pending` y el job de liberación lo respeta hasta que la transacción se liquide o venza el plazo de la pasarela (FR-26.2).

## 8. Flujo 6 — Programación de funciones sin solapamiento de Sala (FR-16)

```mermaid
flowchart TD
    Start([Organizer programa nueva EventSession en una Sala]) --> Range[Calcular rango fecha/hora + buffer configurado]
    Range --> Query[Buscar EventSession existentes en la MISMA Sala]
    Query --> Overlap{¿Algún rango se solapa\ncon el nuevo + buffer?}
    Overlap -->|Sí| RejectSchedule[Rechazar guardado\nMostrar conflicto al Organizer]
    Overlap -->|No| SaveSession[Guardar EventSession\nCrear InventoryUnit iniciales]
    SaveSession --> Done([Función publicable])
    RejectSchedule --> Done2([Organizer ajusta horario])
```

## 9. Flujo 7 — Asociación y sincronización de mapa de Sala con Inamap (NUEVO)

```mermaid
sequenceDiagram
    actor Org as Organizer
    participant INAMAP as Inamap (editor de mapas, sistema externo)
    participant MBT as MBTickets (backend)
    participant INV as InventoryUnit (por Sala)

    Org->>INAMAP: Crea/edita el Layout del recinto (Zonas, Estructuras, Locations = asientos)
    Note over Org,INAMAP: Guardado atómico de documento compuesto (patrón Inamap), corre DENTRO de Inamap

    Org->>MBT: Asocia la Sala a ese Inamap + Layout
    MBT->>INAMAP: GET Locations del Layout (vía credencial de integración)
    INAMAP-->>MBT: Lista de Location (id, nombre, posición)
    MBT->>INV: Crea/actualiza InventoryUnit por cada Location (guarda inamap_location_id)
    MBT-->>Org: Mapa importado, sala lista para programar funciones

    rect rgb(245,245,245)
    Note over INAMAP,MBT: Cambios posteriores en Inamap
    Org->>INAMAP: Modifica el Layout (agrega/mueve/elimina asientos)
    MBT->>INAMAP: Re-sincroniza (pull programado o por webhook, a definir)
    INAMAP-->>MBT: Locations actualizadas
    MBT->>INV: Actualiza InventoryUnit (nuevas altas, bajas solo si no tienen ventas asociadas)
    end
```

**Nota de diseño:** el estado de venta (`disponible`/`retenido`/`vendido`/`REDEEMED`) de cada `InventoryUnit` vive y se protege **exclusivamente en MBTickets** (motor de locking, Flujo 5). Inamap nunca conoce ni decide sobre ese estado — solo aporta la geometría/identidad del asiento.

## 10. Flujo 8 — Cancelación, anulación de QR y reembolso (FR-25)

```mermaid
flowchart TD
    Start([Solicitud de cancelación:\ncomprador, taquilla, organizer, admin o sistema]) --> Redeemed{¿Entrada ya REDEEMED?}
    Redeemed -->|Sí| OnlyAdmin{¿Override de Administrator?}
    OnlyAdmin -->|No| Deny([Rechazar: entrada ya canjeada])
    OnlyAdmin -->|Sí| Policy
    Redeemed -->|No| Policy{¿Dentro de la política de cancelación\no es origen organizer/admin/sistema?}
    Policy -->|No| Deny2([Rechazar: fuera de plazo o sin reembolso])
    Policy -->|Sí| Tx[Transacción única]
    Tx --> Void[TicketCredential → VOID]
    Void --> Release[Liberar InventoryUnit\no devolver cupo al contador]
    Release --> Commit[Commit]
    Commit --> Events["Publicar BookingCancelled\nTicketVoided"]
    Events --> Sync[Propagar lista VOID\na dispositivos vía DeviceCore]
    Events --> Refund{¿Pago reembolsable\npor el mismo medio?}
    Refund -->|"Sí: pasarela o conector con refund()"| Req[refund_pending → solicitar reembolso]
    Refund -->|"No: efectivo o stub 'no implementado'"| Manual[refund_manual_required\ncola de revisión]
    Req -->|OK| Done([refunded + BookingRefunded])
    Req -->|Falla| Failed[refund_failed → cola de revisión]
```

**Lectura clave:** la anulación del QR y la liberación del inventario ocurren en la misma transacción, con el QR anulado antes de que el asiento pueda revenderse. El reembolso es un proceso aparte y tiene su propio estado: que falle nunca deja la entrada activa. Cancelar una función completa (FR-25.7) ejecuta este mismo flujo para cada reserva, por lotes y en cola, con política de cargos desactivada.

## 11. Flujo 9 — API externa de inventario y entrega de webhooks (FR-24)

```mermaid
sequenceDiagram
    participant EXT as Sistema externo
    participant API as API externa MBTickets
    participant INV as Motor de inventario (NFR-01)
    participant BUS as Message Broker
    participant WH as Job de entrega de webhook
    participant RCV as Endpoint del receptor

    EXT->>API: POST /external/v1/holds (ApiClient, Idempotency-Key)
    API->>API: Autentica, valida scope y venue, registra llamada (auditoría)
    API->>INV: Mismo servicio transaccional que los demás canales
    alt Cupo disponible
        INV-->>API: Hold confirmado
        API-->>EXT: 201 + expires_at
        API->>BUS: Publica InventoryChanged
    else Sin cupo
        INV-->>API: Rechazo atómico total
        API-->>EXT: 409 cupo insuficiente
    end

    BUS->>WH: Consume evento (async, cola dedicada)
    WH->>RCV: POST payload + event_id + timestamp + firma HMAC-SHA256
    alt Respuesta 2xx
        RCV-->>WH: OK
        WH->>WH: Registra entrega (intento, latencia, hash)
    else Falla o timeout
        WH->>WH: Reintento con backoff exponencial
        Note over WH,RCV: Tras N fallos consecutivos el endpoint se suspende y se alerta al Organizer
    end
```

**Nota de diseño:** la API externa no es un camino paralelo al inventario: usa el mismo servicio de dominio que online, POS, kiosko y taquilla. El webhook `InventoryChanged` es informativo y eventual (FR-24.9); un receptor que necesite certeza debe consultar la disponibilidad o intentar el `hold`, que es lo único que decide.

## 12. Resumen de decisión

El diseño de MBTickets reutiliza agresivamente todo lo que el core Dash-Admin y sus dominios hermanos (Inamap, KitchnTabs) ya resolvieron como problemas de **plataforma** (identidad, RBAC, multi-tenancy, billing, notificaciones, panel admin, dispositivos de borde), **integra con Inamap** para la parte de mapas de asientos (en vez de reconstruirla), y construye de cero, con el mismo estándar de calidad (servicios PHP puros + pruebas unitarias aisladas), los componentes que constituyen la propuesta de valor única de MBTickets: el motor de concurrencia de inventario sin sobreventa (por `EventSession`), la validación de no-solapamiento de Salas, y la cadena de confianza criptográfica de la credencial de entrada con validación offline — todo ello soportando, con un único modelo de datos, los casos de uso de teatro, cine multiplex, estadio/arena y boletería física de taquilla.
