# MBTickets — Historias de Usuario

> Documento 6 de 9. Formato: Como [rol], quiero [acción], para [beneficio]. Incluye criterios de aceptación (AC) y referencia al FR/NFR relacionado.
> Numeración: este documento contiene HU-01 a HU-18 y HU-27 en adelante. HU-19 a HU-19e están en `08-feature-vanexa-ai-kiosk-integration.md` y HU-20 a HU-26b en `09-feature-pos-payment-terminal-integration.md`.
> Modelo de referencia: Tenancy (cadena) → Tenant = Venue → Sala → EventType → EventSession (ver `02-arquitectura.md` §4). Los mapas de Sala son opcionales e integrados desde Inamap (§4.3).

## Visitante (Guest, no autenticado) / Comprador (usuario final autenticado)

**HU-01:** Como visitante, quiero buscar eventos por categoría, fecha, venue y sala sin necesidad de iniciar sesión, para decidir a cuál asistir. *(FR-01)*
- AC: los resultados se filtran en tiempo real; la disponibilidad mostrada refleja el inventario real al momento de la consulta (no una copia cacheada obsoleta).

**HU-01b:** Como visitante de un cine, quiero ver la cartelera completa del venue (todas las salas, todas las funciones del día), para elegir horario y sala desde app app.mbtickets.sample *(FR-01.3)*
Se debe poder acceder al tenant por {slug}.mbtickets.com o app.mbtickets.sample/app/{slug}

**HU-02:** Como comprador, quiero seleccionar un asiento específico en el mapa de la sala (cuando la sala tiene mapa asociado desde Inamap) o una cantidad de admisión general/sector, para asegurar mi lugar antes de pagar. *(FR-05)*
- AC: al seleccionar, el inventario queda en estado de *hold* temporal visible solo para mí durante el checkout; si no completo el pago dentro del tiempo límite, el hold se libera automáticamente.

**HU-03:** Como comprador, quiero que mi compra se rechace por completo si ya no hay suficientes entradas disponibles, para no quedar con una compra parcial confusa. *(FR-06)*
- AC: si pido 5 y solo quedan 3, la operación completa falla con un mensaje claro; no se me cobran ni confirman 3 de 5.

**HU-04:** Como comprador, quiero recibir mi entrada como un código QR firmado inmediatamente después de pagar, para presentarlo en la entrada del evento. *(FR-08, FR-10)*
- AC: el QR llega por el canal configurado (email/push) en segundos tras la confirmación de pago, de forma asíncrona, sin bloquear la respuesta del checkout.

**HU-05:** Como comprador, quiero que mi entrada no pueda ser usada por nadie más aunque alguien capture una foto de mi QR, para evitar fraude de reingreso con mi entrada. *(FR-09)*
- AC: el primer escaneo válido marca la entrada como canjeada; cualquier escaneo posterior (mío o de terceros), en cualquier puerta del venue, es rechazado.

## Tenancy Admin

**HU-05b:** Como administrador de una cadena (Tenancy Admin), quiero ver y gestionar todos los venues (Tenants) de mi organización desde un solo panel, para supervisar la operación de todas mis sucursales/recintos. *(FR-02.2)*
- AC: puedo entrar al detalle de cualquiera de mis venues sin necesidad de credenciales separadas por sucursal; un Organizer de un venue específico no ve los demás venues de la cadena salvo que yo se lo asigne.

**HU-05c:** Como Tenancy Admin, quiero crear un nuevo venue (Tenant) dentro de mi organización y asignarle un Organizer responsable, para incorporar una nueva sucursal/recinto a la plataforma. *(FR-14)*

## Organizer

**HU-06:** Como organizador, quiero crear un evento definiendo nombre, descripción, duración, precio y capacidad, para publicarlo al público. *(FR-03)*
- AC: el evento no aparece en el descubrimiento público (FR-01) hasta que lo publico explícitamente.

**HU-06b:** Como organizador de un venue con múltiples salas (ej. un multiplex), quiero crear y administrar cada sala por separado (capacidad, nombre, mapa asociado si corresponde), para luego programar funciones de distintas películas/eventos en cada una de forma independiente. *(FR-14.1, FR-14.2)*

**HU-06c:** Como organizador, quiero definir hasta cuándo se pueden comprar entradas de mi evento (una fecha máxima, o un plazo antes de cada función), para cerrar la venta a tiempo para la logística del recinto. *(FR-05.3)*
- AC: después del cierre, ningún canal (online, boletería, kiosko) acepta una compra para esa función, y la función deja de aparecer en el buscador público; si defino una fecha y también un plazo, rige el que cierre antes.

**HU-07:** Como organizador, quiero configurar horarios recurrentes y excepciones de fecha para un evento (ej. funciones todos los sábados, excepto feriados), para no tener que crear cada función manualmente. *(FR-03, modelo EventType/ScheduleRule/DateOverride)*
- AC: una excepción de fecha anula correctamente la regla recurrente para esa fecha puntual, sin afectar las demás funciones.

**HU-07b:** Como organizador de un cine, quiero programar varias funciones por día en la misma sala (distintos horarios, misma o distinta película), para maximizar el uso de la sala sin que se solapen entre sí. *(FR-16)*
- AC: si intento programar una función que se solapa en horario con otra ya existente en la misma sala, el sistema rechaza el guardado y me indica el conflicto; se respeta además el tiempo de buffer configurado entre funciones.

**HU-07c:** Como organizador, quiero asignarle a mi evento una categoría genérica (ej. Entretenimiento, Música, Deportes, Familia) de una lista predefinida de la plataforma, para que aparezca bien clasificado en el buscador público. *(FR-18.1, FR-18.2)*
- AC: la categoría es la misma lista para todos los venues de la plataforma (no la creo yo por venue); un venue de un solo tipo (ej. un teatro) puede tener eventos de distintas categorías.

**HU-07d:** Como organizador, quiero agregar atributos propios a mi evento (ej. clasificación y formato para una película, o equipos y competencia para un partido) como pares libres de dato, sin que el sistema me obligue a un formulario fijo por tipo de evento, para describir cada evento con el detalle que mi vertical necesita. *(FR-18.3, FR-18.4)*
- AC: los atributos que ingreso se muestran en la página de detalle del evento tal como los escribí; el sistema no los valida ni les impone una estructura.

**HU-08:** *(Retirada.)* Esta historia pedía dibujar el mapa de asientos dentro de MBTickets con guardado atómico. Fue reemplazada por la integración con Inamap: el mapa es opcional, se crea y edita en Inamap (que aplica allí el guardado atómico) y se asocia en MBTickets. Ver HU-08b. *(FR-03.1, FR-17)*

**HU-08b:** Como organizador, quiero asociar una Sala de MBTickets a un mapa/layout ya creado en Inamap, para que mis funciones en esa sala tengan selección de asiento numerado sin tener que dibujar el mapa de nuevo. *(FR-17.1, FR-17.2)*
- AC: al asociar, MBTickets trae automáticamente los asientos (`Location`) del layout de Inamap y los deja disponibles como inventario de la sala; si el layout cambia después en Inamap, puedo volver a sincronizar sin perder el estado de venta de los asientos ya vendidos.

**HU-08c:** Como organizador de una sala sin mapa numerado (ej. admisión general en un festival al aire libre), quiero gestionar la capacidad como un simple cupo por sector, sin necesidad de pasar por Inamap. *(FR-17.4)*

**HU-09:** Como organizador, quiero ver en tiempo real cuántas entradas se han vendido, cuántas están en proceso de pago (hold) y cuántas quedan disponibles, por sector/sala y a nivel de todo el venue, para tomar decisiones durante la venta. *(FR-03.2)*
- AC: el dashboard refleja cambios realizados por cualquier canal (online, POS, kiosko, taquilla) sin necesidad de refrescar manualmente más allá de un intervalo razonable.

**HU-10:** Como organizador, quiero ver reportes de embudo de venta y de throughput de check-in el día del evento, desglosados por sala, para evaluar el desempeño de la venta y de la entrada en cada una. *(FR-13)*

## Administrator

**HU-11:** Como administrador, quiero gestionar cuentas de usuario de toda la plataforma, para dar soporte y resolver incidentes. *(FR-04)*

**HU-12:** Como administrador, quiero poder hacer *override* manual de una reserva (ej. reabrir un cupo liberado incorrectamente, o forzar la confirmación de un pago verificado fuera de banda), dejando registro de auditoría de mi acción. *(FR-04.1)*
- AC: el override queda registrado con mi identidad, timestamp y motivo en el log de auditoría, visible para otros administradores.

**HU-13:** Como administrador, quiero inspeccionar logs de auditoría a nivel de todo el sistema (todos los Tenancies/venues), para investigar incidentes cross-tenant. *(FR-04.2)*

## Operador de POS / Kiosko / Boletería (taquilla)

**HU-14:** Como operador de punto de venta físico, quiero vender entradas a un cliente walk-up sobre el mismo inventario que el canal online, para no vender un asiento que ya se vendió por internet. *(FR-07)*
- AC: si un comprador online confirma el último asiento 1 segundo antes que yo, mi intento de venta se rechaza de inmediato con un mensaje claro, no después de cobrar.

**HU-14b:** Como operador de boletería (taquilla) de un teatro o cine, quiero buscar la función exacta (venue → sala → horario) y vender varias entradas contiguas en una sola transacción a un cliente presencial sin cuenta, para atender filas de venta rápido. *(FR-15)*
- AC: puedo seleccionar asientos contiguos en el mapa de la sala para un grupo; la venta se confirma o rechaza como una sola operación atómica (ver FR-06).

**HU-14c:** Como operador de boletería, quiero que mi ventanilla compita en igualdad de condiciones con otras ventanillas del mismo venue por el mismo inventario, sin que ninguna tenga prioridad ni copia local de disponibilidad, para evitar sobreventa entre ventanillas. *(FR-07.3)*

**HU-15:** Como operador de kiosko de autoservicio, quiero que el cliente pueda completar su compra y recibir su QR impreso o enviado a su teléfono sin intervención de personal, para agilizar la fila de venta presencial. *(FR-07, FR-08)*

## Staff de entrada / Validación

**HU-16:** Como personal de entrada con un escáner de mano, quiero validar la entrada de un asistente en menos de medio segundo, para no generar fila en la puerta. *(NFR-03)*

**HU-17:** Como personal de entrada, quiero que el escáner siga funcionando aunque se caiga el internet del recinto, validando firmas y evitando duplicados localmente, para no detener el ingreso durante una falla de red. *(NFR-04)*
- AC: durante el corte, el escáner rechaza correctamente un segundo intento de canje del mismo ticket ya usado localmente; al reconectar, el estado se sincroniza con el servidor central sin duplicar canjes.

**HU-17b:** Como personal de entrada de un estadio con múltiples puertas de acceso, quiero que un ticket canjeado en la puerta Norte sea rechazado automáticamente si se presenta luego en la puerta Sur, para evitar que una misma entrada ingrese a dos personas por distintas puertas. *(FR-09.1)*

## Device / Integración (credenciales no humanas)

**HU-18:** Como administrador de plataforma, quiero asignar una credencial individual y revocable a cada torniquete/scanner/POS/kiosko/boletería, para poder desactivar un dispositivo robado o comprometido sin afectar a los demás. *(FR-11, NFR-05)*
- AC: revocar la credencial de un dispositivo específico no afecta la operación de ningún otro dispositivo del mismo venue (a diferencia de la limitación conocida de credencial única compartida vista en el sistema hermano Inamap).

## Integrador externo (API de inventario y webhooks)

**HU-27:** Como integrador de un sistema externo (ej. un punto de venta de terceros o un canal de reventa autorizado), quiero consultar la disponibilidad en tiempo real de una función por la API, para mostrar a mis clientes cupos que reflejen el inventario real. *(FR-24.1)*
- AC: la disponibilidad que devuelve la API coincide con la que ve el comprador online en el mismo instante; no es una copia cacheada.

**HU-27b:** Como integrador, quiero retener, confirmar y liberar asientos desde mi sistema con una clave de idempotencia, para que un reintento por falla de red no duplique ni pierda una venta. *(FR-24.2, FR-24.3)*
- AC: repetir la misma petición con el mismo `Idempotency-Key` devuelve el resultado original sin modificar el inventario; pedir 5 unidades cuando quedan 3 falla por completo, igual que en cualquier otro canal (FR-06).

**HU-27c:** Como integrador, quiero que mi credencial solo me permita operar sobre los venues que se me asignaron y con los permisos que se me concedieron, para que un error o una fuga de mi lado no comprometa otros recintos. *(FR-24.4, FR-24.5)*
- AC: intentar operar sobre una función de otro venue responde 404; una credencial sin scope `inventory:confirm` no puede confirmar ventas.

**HU-27d:** Como organizador, quiero registrar un endpoint de webhook y elegir qué eventos recibir, para que mis sistemas se enteren de ventas, cancelaciones y cambios de disponibilidad sin consultar la API continuamente. *(FR-24.6)*
- AC: el secreto de firma se muestra una sola vez al crear el endpoint; puedo cambiar la suscripción de eventos sin recrearlo.

**HU-27e:** Como desarrollador que recibe webhooks, quiero verificar la firma y el timestamp de cada entrega y deduplicar por `event_id`, para rechazar mensajes falsos o repetidos y procesar cada evento una sola vez. *(FR-24.8, FR-24.9)*
- AC: una entrega con firma inválida o timestamp antiguo puede rechazarse con certeza; si mi endpoint falla, el sistema reintenta con espera creciente y, tras repetidos fallos, suspende el endpoint y avisa al organizador.

**HU-27f:** Como organizador o administrador, quiero ver el historial de llamadas de API y de entregas de webhook, y reenviar una entrega fallida, para investigar discrepancias entre sistemas. *(FR-24.10, NFR-07)*
- AC: el historial muestra intentos, códigos de respuesta y latencia; reenviar una entrega conserva el mismo `event_id`.

**HU-27g:** Como administrador de plataforma, quiero revocar la credencial de una integración específica sin afectar a las demás, para cortar el acceso de un integrador comprometido o dado de baja. *(FR-24.4, NFR-05)*

## Cancelaciones y reembolsos

**HU-28:** Como comprador, quiero cancelar mi reserva dentro del plazo permitido y ver cuánto se me reembolsará antes de confirmar, para decidir con información completa. *(FR-25.1, FR-25.2)*
- AC: el monto a reembolsar se muestra antes del paso final; al confirmar, mi entrada deja de ser válida de inmediato y recibo la confirmación por el canal configurado.

**HU-28b:** Como comprador, quiero entender por qué no puedo cancelar cuando el plazo venció o mi entrada ya fue usada, para saber qué opciones me quedan. *(FR-25.1, FR-25.3)*
- AC: el mensaje indica el motivo (fuera de plazo, política sin reembolso o entrada ya canjeada) y a quién contactar.

**HU-28c:** Como organizador, quiero definir la política de cancelación de cada evento (plazo y porcentaje de reembolso) con un valor por defecto para mi venue, para no configurarla función por función. *(FR-25.1)*

**HU-28d:** Como organizador que debe cancelar una función (ej. por una falla técnica), quiero cancelarla en bloque y que todos los compradores sean notificados y reembolsados, para no gestionar cada reserva a mano. *(FR-25.7)*
- AC: todas las entradas de esa función quedan anuladas y se encola el reembolso total; las notificaciones se envían de forma asíncrona sin bloquear mi pantalla; cancelar es distinto de despublicar, que conserva las ventas (FR-03.3).

**HU-28e:** Como personal de entrada, quiero que el escáner rechace una entrada anulada aunque su QR tenga firma válida, para no dejar entrar a quien ya fue reembolsado. *(FR-25.4, NFR-04)*
- AC: con conexión, el rechazo es inmediato; sin conexión, el escáner rechaza las entradas anuladas incluidas en la última lista sincronizada y marca para revisión cualquier canje posterior a una anulación no sincronizada.

**HU-28f:** Como operador de taquilla, quiero poder cancelar una venta hecha en mi ventanilla y devolver el dinero al cliente en la misma visita, para corregir errores en el momento. *(FR-25.2, FR-25.5)*
- AC: si el pago fue con tarjeta y el conector admite reembolso, se solicita automáticamente; si fue en efectivo, queda registrada la devolución manual con mi identidad.

**HU-28g:** Como administrador u organizador, quiero ver los reembolsos que requieren acción manual (conector sin soporte, efectivo o falla) y marcarlos como resueltos con referencia y motivo, para que ningún reembolso quede olvidado. *(FR-25.6, FR-25.9)*
- AC: la cola solo muestra reembolsos de mis venues si soy Organizer (FR-23.5); cada resolución queda en el log de auditoría.

## Pago del canal online

**HU-29:** Como comprador online, quiero que mis asientos sigan retenidos mientras completo el pago en la pasarela, para no perderlos por demorarme en ingresar los datos de la tarjeta. *(FR-26.2)*
- AC: al iniciar el pago el plazo del hold se extiende una vez hasta el límite de la pasarela; si no pago, los asientos se liberan al vencer ese plazo.

**HU-29b:** Como comprador, quiero ser reembolsado automáticamente si mi pago se aprobó después de que mis asientos ya se liberaron, para no perder dinero por un retraso ajeno a mí. *(FR-26.3, FR-25.10)*
- AC: el sistema no me asigna asientos que otro comprador pudo haber tomado; recibo un aviso claro del reembolso.

**HU-29c:** Como comprador, quiero que mi pago se procese una sola vez aunque recargue la página de retorno o la pasarela envíe la notificación repetida, para no recibir cobros ni entradas duplicadas. *(FR-26.4)*
- AC: una liquidación repetida de la misma transacción no genera una segunda reserva, un segundo QR ni un segundo cobro.
