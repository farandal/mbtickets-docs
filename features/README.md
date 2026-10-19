# MBTickets — Documentación técnica de features

Una documentación por feature terminada: arquitectura, modelo de datos, flujos (Mermaid), API, código,
pruebas y pendientes. Los requerimientos están en [`../requirements`](../requirements/01-resumen-ejecutivo.md).

| # | Feature | Requerimientos | Estado |
|---|---|---|---|
| F01 | [Inventario y reservas sin sobreventa](F01-inventario-y-reservas.md) | NFR-01, FR-05, FR-05.3, FR-06, FR-07, FR-01 | Terminada (rama `feat/inventory-booking`) |
| F02 | [Pasarelas de pago online en el core](F02-pasarelas-de-pago-checkout-core.md) | Base de FR-26 | Terminada (ramas `feat/checkout-core`) |
| F03 | [Entradas con QR firmado y control de acceso](F03-entradas-firmadas-y-control-de-acceso.md) | FR-08, FR-09, NFR-03, NFR-04 | Terminada (rama `feat/signed-tickets`) |
| F04 | [Pago online de reservas y app del comprador](F04-pago-online-y-app-del-comprador.md) | FR-26, FR-01, HU-29 | Terminada (ramas `feat/booking-checkout`, `feat/buyer-app`) |
| F05 | [Cancelaciones, reembolsos y correos al comprador](F05-cancelaciones-reembolsos-y-correos.md) | FR-25, FR-26.3, FR-10.2 | Terminada (ramas `feat/cancellations-refunds`, core `feat/checkout-core`) |

Repositorios:

| Repo | Rol |
|---|---|
| `mbtickets-backend-domain` | Dominio Laravel (montado sobre el core `dash-backend`). |
| `mbtickets-backend-docker` | Runtime local (`./mbtickets.sh`). |
| `mbtickets-frontend-domain` | `mbtickets-app` (comprador), `mbtickets-web` (venue), `mbtickets-system` (plataforma), `mbt-events`, `mbt-common`. |
| `mbtickets-docs` | Requerimientos y esta documentación. |

Audiencias: el **comprador** usa solo `mbtickets-app` (app pública); el **venue** y la cadena
usan `mbtickets-web`; la **plataforma**, `mbtickets-system`.
