# MBTickets — Requerimientos Funcionales (FR)

> Documento 4 de 9. Ver `02-arquitectura.md` §4 para el modelo de negocio Tenancy → Tenant (Venue) → Sala → EventType → EventSession, y §4.3 para la integración con Inamap.
>
> **Numeración:** este documento define FR-01 a FR-18 y FR-24 a FR-26. FR-19 está en `08-feature-vanexa-ai-kiosk-integration.md` y FR-20 a FR-23 en `09-feature-pos-payment-terminal-integration.md`.
>
> **Terminología de roles:** *Visitante (Guest)* es quien navega sin autenticarse. *Comprador* es el usuario final autenticado. *Organizer*, *Tenancy Admin* y *Administrator* son roles de gestión. Los dispositivos usan credenciales propias (FR-02.3, FR-11).

## 5.1 Acceso por rol y descubrimiento

**FR-01 — Descubrimiento público:** Los visitantes (sin autenticación) pueden explorar, filtrar y buscar eventos por categoría, fecha, venue y sala.
- FR-01.1: Filtro combinable por categoría + rango de fechas + venue + sala + disponibilidad (con/sin cupo).
- FR-01.2: Página de detalle de función (`EventSession`) con mapa de sala (si tiene butacas numeradas vía Inamap, ver FR-17) o indicador de admisión general/sector con cupo restante en tiempo real.
- FR-01.3: Listado de cartelera por venue (todas las salas, todas las funciones del día) — caso de uso cine/multiplex.

**FR-02 — Aislamiento de roles (RBAC):** El sistema aplica control de acceso basado en roles sin solapamiento entre Comprador (usuario autenticado), Organizer, Tenancy Admin y Administrator, heredando el RBAC de dos capas del core (permisos por ruta + policies por instancia vía `scopeVisibleTo`).
- FR-02.1: Un Organizer solo ve/gestiona los venues (Tenants) y salas que le pertenecen; nunca los de otro Tenancy, salvo bypass explícito de Administrator.
- FR-02.2: Un **Tenancy Admin** ve y gestiona todos los Tenants (venues) de su organización (ej. todas las sucursales de una cadena de cines), no solo uno.
- FR-02.3: Operadores de POS/kiosko/boletería/torniquete se modelan como credenciales de dispositivo (no usuarios humanos), con permisos acotados exclusivamente a venta y validación, y asociados a un Venue y, cuando aplica, a una Sala específica.

**FR-03 — Gestión del Organizer:** Los Organizers pueden crear, editar y retirar venues, salas y eventos (`EventType`), configurar horarios recurrentes (`ScheduleRule`) y excepciones de fecha (`DateOverride`), y ver métricas de asistentes y capacidad en tiempo real para los eventos que poseen.
- FR-03.1: **Asociación opcional de mapa de Sala vía Inamap** — el Organizer no dibuja el mapa de asientos dentro de MBTickets; en su lugar, selecciona/asocia un `Inamap`+`Layout` ya creado externamente en Inamap (ver FR-17). Una Sala sin mapa asociado funciona en modo admisión general/por sector, con capacidad gestionada como contador simple dentro de MBTickets.
- FR-03.2: Dashboard de capacidad en tiempo real: vendidos / retenidos (hold) / disponibles, por sección/sector, por sala y por venue.
- FR-03.3: Posibilidad de retirar (despublicar) un evento o una función sin eliminar el historial de ventas/validaciones ya realizadas.
- FR-03.4: Calendario/cartelera por Sala: vista de todas las `EventSession` programadas en una Sala, para planificar sin solapamientos (ver FR-16).

**FR-04 — Administración del sistema:** Los Administrators pueden gestionar cuentas de usuario, hacer *override* de reservas cuando sea necesario, e inspeccionar logs de auditoría a nivel de todo el sistema.
- FR-04.1: Override de booking queda registrado en el log de auditoría (heredado de `HasAuditLog`/Activitylog del core) con el usuario administrador responsable.
- FR-04.2: Vista de auditoría cross-tenant (solo Administrator), reutilizando `ReportCore` del core.

## 5.2 Reserva e inventario

**FR-05 — Reserva de entradas:** Usuarios autenticados y operadores de POS/boletería pueden reservar una o más entradas para una `EventSession`, sujeto a la capacidad y los horarios disponibles.
- FR-05.1: Selección de asiento específico (sala con mapa importado de Inamap) o cantidad de admisión general/por sector (GA), según el tipo de sala/evento.
- FR-05.2: *Hold* temporal del inventario seleccionado durante el checkout (con expiración configurable), antes de la confirmación de pago.

**FR-06 — Rechazo atómico:** El sistema rechaza una solicitud de reserva por completo —nunca de forma parcial— cuando la cantidad solicitada excede la capacidad restante de la `EventSession` (o del sector/asiento solicitado).
- FR-06.1: Un intento de comprar 5 entradas cuando solo quedan 3 debe fallar en su totalidad (ninguna de las 5 se confirma), con mensaje claro de cupo insuficiente.

**FR-07 — Inventario unificado:** El checkout online, las terminales POS/kiosko y la **boletería de taquilla física** descuentan de una única fuente de verdad para el inventario de asientos y de admisión general de cada `EventSession`.
- FR-07.1: Ninguna terminal (online, POS, kiosko, taquilla) mantiene una copia local de "cupos restantes" como fuente de verdad; todas consultan/mutan el mismo inventario central bajo transacción.
- FR-07.2: Confirmación de venta en POS/kiosko/taquilla dentro del mismo límite de latencia que el checkout online (ver NFR-02).
- FR-07.3: La boletería física puede operar con múltiples puntos de venta simultáneos dentro del mismo venue (varias ventanillas), todos compitiendo por el mismo inventario sin sobreventa entre ellos.

## 5.3 Credenciales, validación y notificaciones

**FR-08 — Emisión de QR firmado:** El sistema emite una credencial QR única, firmada y a prueba de manipulación para cada entrada confirmada.
- FR-08.1: El payload codifica, como mínimo: id de ticket, id de `EventSession` (venue + sala + función), asiento/sector (si aplica), timestamp de emisión, y firma Ed25519/ECDSA del payload.
- FR-08.2: El token en claro se entrega una sola vez al comprador (patrón de token opaco de Inamap); el servidor solo conserva lo necesario para la verificación (clave pública del tenant, estado del ticket), nunca el secreto del comprador.

**FR-09 — Validación de un solo uso:** El sistema valida cada credencial QR como máximo una vez, rechazando escaneos duplicados o firmas forjadas en el punto de entrada.
- FR-09.1: Primer escaneo válido marca el ticket como `REDEEMED` y concede acceso; cualquier escaneo posterior del mismo ticket se rechaza (anti-passback), incluso si ocurrió en otra puerta/torniquete del mismo venue o sala.
- FR-09.2: Durante un corte de red local, el torniquete valida la firma localmente y mantiene su propio registro de canjeados; al reconectar, sincroniza y resuelve conflictos (ver NFR-04).

**FR-10 — Notificaciones orientadas a eventos:** Al confirmarse o cancelarse una reserva, el sistema despacha de forma asíncrona notificaciones de confirmación y entrega de entrada vía el message broker.
- FR-10.1: Los eventos de dominio (`BookingConfirmed`, `BookingCancelled`, `BookingRefunded`, `TicketVoided`, `TicketValidated`) se publican de forma desacoplada; los listeners (notificación, analítica, log de auditoría) se suscriben de forma independiente — nunca invocación síncrona directa dentro del controlador de reserva.
- FR-10.2: Canal de entrega configurable (email, push, u otros) reutilizando el motor de notificaciones multicanal del core.

## 5.4 Requerimientos funcionales adicionales (venues multi-sala, boletería e integración Inamap)

**FR-11 — Credenciales por dispositivo:** A diferencia del modelo de "una credencial compartida por recinto" visto en Inamap (limitación documentada por ellos mismos), cada terminal POS/kiosko/boletería/torniquete/scanner de mano de MBTickets tiene su propia credencial de dispositivo, individualmente revocable, sobre `DeviceCore`, y asociada a un Venue y opcionalmente a una Sala.

**FR-12 — Límites por plan:** Cada Tenant (venue) tiene límites configurables por plan de suscripción (máximo de salas, máximo de eventos activos, máximo de entradas/mes, etc.), reutilizando `PlanLimitsService` del core con conteo propio del dominio MBTickets (mismo patrón que `InamapPlanLimits`).

**FR-13 — Reportes operativos:** El Organizer y el Administrator acceden a reportes de embudo de venta (visitas → reserva iniciada → pagada) y de throughput de check-in (entradas validadas por minuto en el punto de entrada), desglosados por venue y por sala, adaptando el patrón de reportes de KitchnTabs (`TabsFunnelReport`, `TabsThroughputReport`) al dominio de ticketing.

**FR-14 — Gestión de Venue y Salas:** Un Tenancy Admin puede crear y administrar múltiples Tenants (venues); cada Venue puede tener una o más Salas, cada una con su propio calendario de funciones.
- FR-14.1: Un venue tipo cine puede tener entre 1 y N salas (sin límite técnico, sujeto a límite de plan); un venue tipo teatro típicamente 1–2; un venue tipo estadio típicamente 1 sala con múltiples sectores internos.
- FR-14.2: Las salas de un mismo venue pueden alojar `EventSession` simultáneas de distintos `EventType`, sin interferir entre sí (inventario, calendario y validación de entrada son independientes por sesión).

**FR-15 — Boletería de taquilla física (caso de uso transversal):** El sistema soporta venta presencial en ventanilla/taquilla, para cualquier vertical (teatro, cine, estadio, arena), como un canal más sobre el inventario unificado (ver FR-07).
- FR-15.1: El operador de taquilla puede buscar y vender entradas para cualquier `EventSession` activa del venue en el que está autenticado, sin necesidad de que el comprador tenga cuenta.
- FR-15.2: Soporta venta de múltiples entradas en una sola transacción (ej. grupo familiar), con selección de asientos contiguos cuando el mapa de la sala lo permite.

**FR-16 — Prevención de solapamiento de Sala:** Al crear o editar el calendario de funciones (`EventSession`) de una Sala, el sistema rechaza una función cuyo rango de fecha/hora se solape con otra función ya programada en la misma Sala.
- FR-16.1: Esta validación es independiente del motor de locking de inventario (NFR-01): ocurre en el momento de programar el calendario (baja concurrencia, por el Organizer), no en el momento de venta.
- FR-16.2: El sistema debe considerar un tiempo de "buffer" configurable entre funciones de la misma Sala (ej. tiempo de limpieza/montaje en cine o teatro) al validar el solapamiento.

**FR-17 — Integración bidireccional con Inamap para mapas de Sala (NUEVO):** El mapa de asientos de una Sala es **opcional** y, cuando existe, se origina y mantiene en Inamap, no en MBTickets. Ver `02-arquitectura.md` §4.3 para el diseño completo.
- FR-17.1: El sistema debe permitir al Organizer **asociar** una Sala existente de MBTickets a un `Inamap`+`Layout` existente, mediante un selector que consulte la API de Inamap del mismo Tenancy/Tenant.
- FR-17.2: Tras la asociación, el sistema debe **sincronizar (pull)** las `Location` del layout asociado para generar el inventario de asientos (`InventoryUnit`) de esa Sala, conservando la referencia externa (`inamap_location_id`) por asiento.
- FR-17.3: El sistema debe detectar cambios posteriores en el mapa de Inamap (asientos agregados/movidos/eliminados) y re-sincronizar el inventario correspondiente, sin afectar el estado de venta ya existente de los asientos no modificados.
- FR-17.4: Una Sala sin mapa asociado debe operar en modo admisión general/por sector de forma normal, sin bloquear ninguna otra funcionalidad del sistema.
- FR-17.5 *(a confirmar con el equipo de Inamap)*: Opcionalmente, el sistema puede escribir de vuelta en Inamap una referencia de que una `Location` está en uso por una Sala de MBTickets, para prevenir su eliminación accidental desde la administración de Inamap.

**FR-18 — Categorización y atributos propios de un EventType (NUEVO):** El sistema permite clasificar y enriquecer cada `EventType` sin ramificar el modelo de datos ni la lógica de negocio por vertical. Ver `02-arquitectura.md` §4.4 para el diseño completo.
- FR-18.1: El Organizer debe poder asignar a cada `EventType` una **`EventCategory`** (etiqueta genérica de descubrimiento, ej. *Entretenimiento*, *Música*, *Deportes*, *Familia*, *Arte y Cultura*), de una lista plana administrada a nivel de plataforma (no por venue individual).
- FR-18.2: `EventCategory` no define estructura ni comportamiento propio: es exclusivamente un valor de clasificación usado en el descubrimiento público (FR-01.1) y en reportes agregados por categoría (FR-13). Es independiente del tipo de venue/sala (§4.2): un mismo venue puede alojar `EventType` de distintas categorías, y una misma categoría puede darse en distintos tipos de venue.
- FR-18.3: El Organizer debe poder definir, para cada `EventType`, un conjunto libre de **atributos propios** (`custom_attributes`, pares clave-valor en JSON) sin esquema fijo impuesto por el sistema, para datos de presentación específicos de la vertical (ej. clasificación/formato/idioma de una película, elenco/director de una obra, equipos/competencia de un partido).
- FR-18.4: El sistema no valida ni interpreta el contenido de `custom_attributes`; se muestran tal cual en la página de detalle del evento (FR-01.2) y pueden usarse como filtro adicional de búsqueda dentro de una categoría.
- FR-18.5: Ni `EventCategory` ni `custom_attributes` deben ramificar el motor de locking de inventario (NFR-01), el motor de validación de QR (FR-09) ni el motor de notificaciones (FR-10) — un único código de estos tres motores sirve para cualquier categoría o conjunto de atributos.

## 5.5 Interoperabilidad externa, cancelaciones y pagos online

**FR-24 — API externa de inventario y webhooks salientes (NUEVO):** Un sistema externo autorizado puede consultar y modificar la disponibilidad de una `EventSession` mediante una API, y recibir notificaciones por webhook de los eventos de dominio relevantes. Toda interacción queda auditada. Ver NFR-07 para las metas de rendimiento y auditoría.
- FR-24.1: **Consulta de disponibilidad.** La API expone, por `EventSession`, la disponibilidad actual (por asiento numerado y por sector/GA) con el mismo detalle que el descubrimiento público (FR-01.2), más el estado `retenido`/`vendido` cuando la credencial tiene el scope correspondiente.
- FR-24.2: **Operaciones de escritura.** La API permite `hold`, `confirm`, `release` y bloqueo/desbloqueo administrativo de unidades (ej. cortesías, asientos con visibilidad reducida, reservados por producción). Todas se ejecutan a través del mismo servicio transaccional de NFR-01 y con el mismo rechazo atómico total de FR-06. Ningún canal externo escribe directamente sobre contadores o filas de `InventoryUnit`.
- FR-24.3: **Idempotencia.** Toda escritura exige un `Idempotency-Key`. Repetir la misma petición con la misma clave devuelve el resultado original sin volver a mutar el inventario.
- FR-24.4: **Credenciales de integración (`ApiClient`).** Cada integración tiene su propia credencial, acotada a uno o más Tenants (venues) autorizados, con scopes explícitos (`availability:read`, `inventory:hold`, `inventory:confirm`, `inventory:block`, `webhooks:manage`), individualmente revocable y mostrada una sola vez al crearla. Reutiliza el modelo de FR-11, pero para sistemas externos, no para dispositivos.
- FR-24.5: **Aislamiento.** Una credencial solo puede operar sobre los venues que se le asignaron (`scopeVisibleTo`). Una unidad de otro venue responde 404, nunca 403, para no revelar su existencia.
- FR-24.6: **Webhooks salientes.** El Organizer (o un `ApiClient` con scope `webhooks:manage`) registra endpoints HTTPS y elige a qué eventos suscribirse: `BookingConfirmed`, `BookingCancelled`, `BookingRefunded`, `TicketValidated`, `TicketVoided`, `InventoryChanged` (cambio de disponibilidad por sesión o sector, incluido agotado y reabierto) y `EventSessionCancelled`.
- FR-24.7: **Desacople.** Los webhooks se despachan desde listeners de los eventos de dominio de FR-10.1, por cola. Nunca se invocan de forma síncrona dentro del flujo de reserva, y una falla del receptor no afecta la reserva.
- FR-24.8: **Autenticidad y entrega.** Cada entrega incluye `event_id` único y estable, `timestamp` y firma HMAC-SHA256 con el secreto del endpoint (mostrado una sola vez). La entrega es *at-least-once* con reintentos y backoff exponencial acotado. Tras N fallos consecutivos el endpoint se suspende automáticamente y se alerta al Organizer.
- FR-24.9: **Eventualidad.** Un webhook `InventoryChanged` es informativo y puede llegar con retraso o fuera de orden. La fuente de verdad sigue siendo la consulta de FR-24.1 y la validación transaccional de FR-24.2. El receptor no debe usarlo para decidir una venta.
- FR-24.10: **Auditoría y operación.** Cada llamada a la API y cada intento de entrega queda registrado (ver NFR-07). El Organizer ve el historial de entregas de sus endpoints y puede reenviar manualmente una entrega fallida. El Administrator puede inspeccionar el registro cross-tenant (FR-04.2).

**FR-25 — Cancelación y reembolso de reservas (NUEVO):** Una reserva confirmada puede cancelarse según una política configurable. La cancelación libera el inventario, anula la credencial QR y dispara el reembolso por el medio de pago original cuando corresponde. Define el efecto de dominio que `09-feature-pos-payment-terminal-integration.md` FR-23.3 deja pendiente.
- FR-25.1: **Política de cancelación.** El Organizer define por `EventType` (con valor por defecto a nivel de Tenant) si se permite cancelar, hasta cuántas horas antes del inicio de la `EventSession`, y qué porcentaje o cargo fijo se reembolsa. El comprador ve el monto a reembolsar antes de confirmar la cancelación. *Valores por defecto: a definir con negocio.*
- FR-25.2: **Quién puede cancelar.** El Comprador (sobre sus propias reservas, dentro de la política), el operador de taquilla (reservas hechas en su venue, sujeto a permiso), el Organizer (reservas de sus venues) y el Administrator (override con motivo, FR-04.1). Cada cancelación registra el origen: `buyer`, `box_office`, `organizer`, `admin` o `system`.
- FR-25.3: **Alcance.** La cancelación puede ser de la reserva completa o de entradas individuales (`BookingItem`), según lo que habilite la política. Una entrada ya `REDEEMED` no puede cancelarse; solo el Administrator puede revertirla con override auditado.
- FR-25.4: **Efecto atómico.** En una sola transacción, la cancelación libera las `InventoryUnit` (o devuelve el cupo al contador) y pasa la `TicketCredential` a `VOID`. La credencial se anula antes de que el inventario pueda revenderse. La lista de credenciales `VOID` se propaga a los dispositivos de validación vía `DeviceCore`, de modo que una entrada anulada con firma válida se rechaza también offline (NFR-04, con el riesgo residual allí documentado).
- FR-25.5: **Reembolso.** Tras la cancelación se solicita el reembolso por el mismo medio de pago: pasarela online (`refundPayment()`) o conector del terminal físico (`refund()`, ver doc 09 FR-23.2). El estado de la reserva (`cancelled`) y el del reembolso (`refund_pending`, `refunded`, `refund_failed`, `refund_manual_required`) son independientes. Un reembolso fallido nunca deja la entrada activa ni el inventario bloqueado.
- FR-25.6: **Conector sin soporte de reembolso, y pagos en efectivo.** Si el conector devuelve el stub "no implementado", o el pago fue manual/efectivo, el reembolso pasa a `refund_manual_required` y aparece en una cola de revisión del Organizer/Administrator, que lo marca como resuelto con referencia y motivo. Se reutiliza la Admin Payments Management UI (doc 09 FR-23).
- FR-25.7: **Cancelación por el Organizer de una función.** Cancelar una `EventSession` (distinto de despublicarla, FR-03.3) cancela todas sus reservas confirmadas, anula todas sus credenciales y encola el reembolso total de cada una, sin aplicar política de cargos. La notificación masiva a compradores es asíncrona y por lotes (FR-10). Reprogramar una función no es una cancelación: se especifica aparte *(abierto)*.
- FR-25.8: **Eventos y notificaciones.** Se publican `BookingCancelled`, `TicketVoided` y `BookingRefunded` por el pipeline de FR-10; los listeners notifican al comprador, actualizan reportes (FR-13) y alimentan los webhooks de FR-24.
- FR-25.9: **Auditoría.** Cada cancelación y reembolso registra identidad, origen, motivo, monto y timestamp en el log de auditoría, con el mismo patrón que el override de FR-04.1.
- FR-25.10: **Pago aprobado tarde.** Un pago que se aprueba después de que el hold expiró (caso HU-25 del doc 09 y FR-26.3) no confirma la reserva. Se registra como caso de conciliación y se resuelve con reembolso por el flujo de FR-25.5.

**FR-26 — Pago del canal online y conciliación con el hold (NUEVO):** El checkout online cobra mediante pasarelas intercambiables y liquida el resultado exactamente una vez, sin que el hold de inventario expire a mitad del pago.
- FR-26.1: **Pasarela intercambiable.** El checkout online usa la capa `CheckoutGateway` reutilizada de KitchnTabs (Flow, Transbank, Mercado Pago, gateway interno). Igual que FR-21.5, el motor de reservas solo reacciona al estado liquidado de la transacción, sin conocer qué pasarela la procesó.
- FR-26.2: **Hold durante el pago.** Al iniciar el pago, el hold pasa a `payment_pending` y su `expires_at` se extiende una única vez hasta el timeout de la pasarela (máximo configurable). El job de liberación no libera holds en `payment_pending` hasta que la transacción se liquide o venza ese plazo.
- FR-26.3: **Aprobación tardía.** Si la pasarela aprueba un pago cuando el hold ya fue liberado, el sistema no reconfirma la reserva ni toma inventario que otro comprador pudo haber adquirido. La transacción se registra para conciliación y se reembolsa conforme a FR-25.5/FR-25.10. Es la misma regla que el doc 09 aplica a pagos presenciales.
- FR-26.4: **Liquidación única e idempotente.** El resultado de un pago se aplica una sola vez aunque la pasarela reenvíe el webhook o el comprador regrese varias veces a la URL de retorno. Se reutiliza el patrón de `completeTransaction()` de doc 09 §8: se comprueba si la transacción ya estaba en estado terminal antes de aplicar efectos secundarios.
- FR-26.5: **Resultado de pago.** `approved` confirma la reserva y emite `BookingConfirmed`. `declined`, `cancelled`, `error` y `timeout` liberan el hold inmediatamente. El comprador recibe un mensaje claro en cada caso y nunca queda cobrado sin entrada ni con entrada sin cobro.
