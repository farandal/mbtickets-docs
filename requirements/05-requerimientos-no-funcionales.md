# MBTickets — Requerimientos No Funcionales (NFR)

> Documento 5 de 9. Los enfoques técnicos propuestos aquí son hipótesis de diseño a validar, no decisiones cerradas.

## NFR-01 — Consistencia (el requisito crítico del proyecto)

**Especificación:** Bajo acceso concurrente a través de canales online y POS, las reservas confirmadas nunca deben exceder la capacidad declarada de un evento (objetivo de tolerancia cero, vía transacciones serializables / locking a nivel de fila).

**Por qué es el riesgo #1:** confirmado mediante inspección de código que ni Inamap ni KitchnTabs resolvieron nunca este problema — no existe `lockForUpdate()`, `SERIALIZABLE` ni patrón de reserva atómica de stock en ninguno de los dos dominios hermanos. No hay código a copiar; se diseña de cero.

**Enfoques técnicos a evaluar en diseño detallado:**
1. **Locking pesimista explícito:** `SELECT ... FOR UPDATE` sobre la fila de capacidad/asiento dentro de `DB::transaction()`, con reintento acotado ante *lock wait timeout*.
2. **Decremento atómico condicional (sin lock explícito):** `UPDATE inventory SET remaining = remaining - :qty WHERE remaining >= :qty RETURNING id`; si no retorna fila, se rechaza la reserva completa (cumple FR-06 de forma natural).
3. **Aislamiento `SERIALIZABLE` de PostgreSQL** con reintento automático de la transacción ante conflicto de serialización (código de error `40001`).

**Dos modelos de inventario bajo el mismo motor:** el enfoque 2 (contador `remaining`) es natural para admisión general y sectores sin asiento numerado. Para asientos numerados (con `inamap_location_id`) la unidad de exclusión es la fila de `InventoryUnit` y su estado (`disponible` → `retenido` → `vendido`), con el mismo rechazo atómico total de FR-06: se bloquean todas las filas solicitadas en un orden determinista (por id) para evitar deadlocks entre compras de asientos solapados, o ninguna. Un servicio de dominio único expone ambas variantes; los canales (online, POS, kiosko, taquilla, API externa de FR-24) no distinguen cuál se usa.

La elección debe probarse bajo carga realista (ver NFR-02) antes de congelarse, y debe implementarse como servicio PHP puro y aislado, con batería de pruebas de concurrencia dirigida (N requests simultáneas contra 1 unidad de inventario → exactamente 1 confirma).

## NFR-02 — Latencia de reserva

**Especificación:** El tiempo de respuesta p95 de una solicitud de reserva debe mantenerse por debajo de 250 ms bajo una carga sostenida de 500 usuarios concurrentes.

**Nota de referencia:** Inamap definió sus propios RNF de latencia de búsqueda con el mismo estilo de métrica (p95 < 300 ms con 10.000 elementos), validándolo con un servicio aislado y pruebas dedicadas — mismo enfoque de validación recomendado aquí. La estrategia de locking elegida en NFR-01 es, en la práctica, el principal factor de riesgo para cumplir este NFR bajo los picos de un festival de 20.000 entradas.

## NFR-03 — Throughput de check-in

**Especificación:** La verificación de QR en el punto de entrada debe completarse en menos de 500 ms por escaneo, para evitar colas físicas en los torniquetes.

**Enfoque técnico:** la verificación de firma Ed25519 es computacionalmente barata y se realiza localmente en el dispositivo (sin round-trip al servidor en el camino feliz), lo que hace este NFR alcanzable siempre que el payload del QR sea compacto y la clave pública esté cacheada localmente (ver NFR-04).

## NFR-04 — Resiliencia offline

**Especificación:** Los escáneres de entrada del recinto deben seguir siendo capaces de validar firmas criptográficas de QR y de prevenir *pass-back*/duplicación local durante cortes temporales de conectividad a internet.

**Por qué es un componente 100% nuevo:** Inamap siempre valida contra la base de datos central (nunca offline); KitchnTabs no tiene concepto de validación de entrada en absoluto. No hay precedente de resiliencia offline documentado en ningún sistema hermano.

**Enfoque técnico propuesto:**
- Cada dispositivo de validación cachea localmente la(s) clave(s) pública(s) del tenant/evento (sincronizadas vía `DeviceCore` mientras hay red).
- Verificación de firma 100% local y sin red en el camino feliz.
- Registro local de tickets ya canjeados (anti-passback) mientras dura el corte, con sincronización diferida al reconectar.
- Lista de credenciales anuladas (`VOID`, FR-25.4): cada dispositivo sincroniza de forma incremental esta lista mientras hay red y rechaza localmente cualquier ticket incluido en ella. **Riesgo residual documentado:** una entrada anulada durante un corte de red no puede rechazarse offline hasta que el dispositivo reciba la lista actualizada; el canje queda registrado y se marca para revisión al reconectar.
- Estrategia de reconciliación de conflictos al reconectar (ej. "primer canje gana", con alerta de doble canje para revisión manual) — a definir en diseño detallado junto al equipo de operaciones de recinto.

## NFR-05 — Seguridad

**Especificación:** Todos los tokens de autenticación deben estar firmados criptográficamente y acotados en el tiempo; todos los endpoints de Organizer y Administrator deben aplicar verificación de RBAC en el servidor, independiente del comportamiento del cliente.

**Heredado del core:** Sanctum ya emite tokens firmados y acotados en el tiempo; el RBAC de dos capas (permisos por ruta + policies por instancia) ya aplica verificación server-side independiente del cliente.

**Nuevo para MBTickets:** las credenciales de *dispositivo* (POS/kiosko/torniquete) deben ser individuales y revocables (a diferencia de la credencial única compartida de Inamap, limitación que ellos mismos documentan como pendiente de resolver) — ver FR-11. Protección de endpoints públicos (descubrimiento de eventos, compra) con throttling por IP, identificadores no secuenciales (UUID), y secretos siempre hasheados en reposo (mismo estándar que Inamap: *"UUID no secuenciales; tokens de 256 bits solo hasheados"*).

## NFR-06 — Escalabilidad y disponibilidad

**Especificación:** La capa de aplicación debe poder desplegarse en configuración horizontal, sin estado, sin más componente con estado que la base de datos relacional y el message broker.

**Heredado del core:** la infraestructura de despliegue (nginx/php-fpm sin estado tras ALB, Redis/Horizon para colas, Reverb para websockets) ya sigue este principio en los dominios hermanos. El diseño de MBTickets no debe introducir estado en la capa de aplicación (ej. nada de "cupos en memoria del proceso"); todo el estado de inventario vive en Postgres, protegido por el mecanismo de NFR-01.

**Consideración específica de MBTickets:** los picos de tráfico de venta de festivales (20.000 entradas agotadas en minutos) son más extremos que el patrón de tráfico observado en Inamap/KitchnTabs — se recomienda dimensionar Horizon/Redis de forma independiente (no necesariamente compartida con otros dominios) y definir un plan de prueba de carga específico antes del lanzamiento de cualquier evento de alta demanda.

## NFR-07 — Interoperabilidad externa y auditabilidad

**Especificación:** La API externa de inventario y los webhooks salientes (FR-24) deben operar sin comprometer la consistencia de NFR-01 ni la latencia de NFR-02, y toda interacción debe quedar registrada de forma auditable.

**Metas propuestas (hipótesis a validar con pruebas de carga, igual que NFR-02):**
- Los endpoints externos de `hold`/`confirm`/`release` cumplen el mismo p95 < 250 ms de NFR-02, medido bajo la mezcla de canales esperada (online + POS + API externa).
- Un webhook se entrega por primera vez en menos de 30 s tras el commit del evento de dominio, en condiciones normales de cola. La entrega es *at-least-once*, nunca *exactly-once*: el receptor deduplica por `event_id`.
- Una caída o lentitud del endpoint de un receptor nunca bloquea ni degrada el camino de reserva. La entrega vive en una cola dedicada y el tiempo de espera de cada intento es acotado (ej. 10 s).
- Rate limiting por `ApiClient`, para que una integración defectuosa no pueda agotar la capacidad que necesitan los demás canales en un pico de venta.

**Auditoría (requisito de `01-resumen-ejecutivo.md` §3):**
- Cada llamada de escritura de la API externa registra credencial, `Idempotency-Key`, recurso afectado, resultado y latencia.
- Cada entrega de webhook registra endpoint, `event_id`, número de intento, código de respuesta, latencia y hash del payload.
- Retención de estos registros: a definir con el equipo según costo de almacenamiento. Una propuesta razonable es 13 meses para cubrir un ciclo anual de eventos.

**Seguridad:** secretos de webhook y credenciales de `ApiClient` hasheados o cifrados en reposo y mostrados una sola vez (estándar de NFR-05); firma HMAC con timestamp para que el receptor pueda rechazar *replays*; URL de destino restringida a HTTPS y validada contra destinos internos (protección SSRF).
