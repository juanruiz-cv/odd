# FASE 5 — Movements (EPIC-006)

## Objetivo
Implementar PHASE 5 según `PHASES.md` §9: máquina de estados de `CargoStatus` (BR-016/BR-045),
endpoint de movimientos con observación obligatoria e historial (BR-006/007/008), validaciones de
ubicación/capacidad integradas (BR-004/005/033/034/035), transacciones de segmentos `CargoLocation`
(BR-039), movimientos parciales y residual en camión (BR-037/038/042), reversión ADMIN (BR-012),
alerta de capacidad (P5-T8) y la UI del flujo (P5-T4).

## Alcance autorizado
- Backend: `src/modules/movements` (nuevo), extensiones de `src/modules/cargo` (endpoint de
  movimientos, residual derivado, `locationId` en el alta directa), `prisma/seed.ts`.
- Frontend: `src/app/features/cargo` (MovementTimeline, ObservationDialog, ConfirmDialog).
- Git flow: ramas `feature/p5-*` desde `develop`, merges `--no-ff` a `develop`. Sin push/remote.
  Sin atribución AI. No se modifica nada bajo `cargoops-docs/`.

## Estado de partida (verificado por el orquestador)
- **Prisma YA tiene todo el modelo** (FASE 1 R-03): `Movement` (L415, con `observationId` `@unique`
  1:1, `quantityMoved`, `from/toCargoLocationId`, `reversionOfId`/`reversedById`/`reversedAt`,
  `metadata`), `Observation` (L457), `Alert` (L478). **5A–5D no requieren migraciones**; 5E sí
  agrega una para ampliar el alcance de `Alert` a ubicaciones.
- **`MovementKind` ya tiene los 9 valores** (`INGRESS, UNLOAD, MOVE, TO_SCANNER, TO_BALANZA,
  TO_REZAGO, TO_SECUESTRO, REVERSION, EXIT`). DATABASE.md §5.5 L374 **decidió** mantener el enum
  completo en v1 con validación de pares permitidos (kind → destino) en el servicio. Queda
  **cerrada** la "decisión pendiente" que mencionaba PHASES.md §9; no hay nada que decidir.
- **Permisos reales del catálogo** (ADR-009): `cargo.move`, `cargo.to_rezago`, `cargo.to_secuestro`,
  `cargo.revert`, `cargo.update`, `cargo.read`, `alert.read`, `alert.manage`.
  **NO existen permisos `movement.*`** ni `alert.ack`: los movimientos se gobiernan con los permisos
  de `cargo`. No se inventan permisos nuevos.
- **Reutilizable de PHASE 4** (no reimplementar):
  - `src/modules/locations/capacity.calculator.ts` — `isUnitCompatible` (BR-035), `occupiedInUnit`
    (BR-033), `checkCapacity` (BR-005/BR-036 con `allowOverOccupation` +10% default), `roundQuantity`.
  - `src/modules/locations/location-domain.service.ts` — validación de estado de ubicación (P4-T3).
  - `src/common/errors/domain.exceptions.ts` — `locationInactive` (BR-004, con la nota explícita
    "reused by PHASE 5"), `capacityExceeded` (BR-005/BR-036), `unitIncompatible` (BR-035),
    `distributionExceedsTotal` (BR-034), `cargoLocationConflict` (BR-032), `cargoLocationNotFound`,
    `businessRuleViolation` (422 con `details.rule`).
  - **Falta** el factory de `INVALID_TRANSITION` (409, BR-016) — hay que agregarlo.
- `GET /cargos/:id` ya devuelve `movementsCount` y `observations[]`. `POST /cargos/:id/observations`
  y `/notes` ya existen (PHASE 3, permiso `cargo.update`).
- `src/modules/movements` **no existe** (verificado): la FASE 5 arranca de cero en ese módulo.
- Backend `develop` = `b9f8517`; Frontend `develop` = `06184d1`. Ambos limpios.
- Guards globales deny-by-default: toda ruta nueva necesita `@RequirePermissions(...)`.
- Contrato de errores: `INVALID_TRANSITION` es **409 canónico** (ERROR-HANDLING.md §4.3, ID-002
  resuelta 2026-09-23).

## Tareas
- [x] P5-T1 Máquina de estados (`StateMachineValidator`): transiciones válidas §7 / VALIDATION §4.4.1
  + §4.4.3 incluida `PARTIALLY_UNLOADED`; BR-016/BR-045; `fromStatus` se lee con lock, nunca del
  request; transición inválida → `409 INVALID_TRANSITION` con la transición en `details`.
- [x] P5-T2 `POST /api/v1/cargos/:id/movements` (API.md §5.6): observación obligatoria (BR-006/007),
  historial reconstruible (BR-008), `IN_TRANSIT` transitorio, carga inexistente → BR-003.
  Incluye `GET /cargos/:id/movements` (§5.5) con paginación y filtros `kind`/`userId`/`from`/`to`.
- [x] P5-T3 Validaciones de ubicación/capacidad integradas (BR-004/005/033/035) por unidad
  compatible, reutilizando los helpers de PHASE 4. Mover a ubicación inactiva → rechazo con tests;
  unidades incompatibles → `UNIT_INCOMPATIBLE`.
- [x] P5-T4 UI del flujo: `ObservationDialog` + `ConfirmDialog` + `MovementTimeline` básico
  (captura de cantidad/porcentaje en movimientos parciales, BR-037); el operador completa un
  movimiento con observación y cantidad/porcentaje en ≤3 pasos. **Hecho** — base `ui/dialog`,
  `ConfirmDialog`, `ObservationDialog` (contador 500, DP-UI-02), `MovementTimeline` y `MoveCargoForm`
  en `56ae05b`; merge `--no-ff` a `develop` en **`783553c`**.
- [x] P5-T5 Reversión ADMIN (BR-012, ADR-010/011): `kind=REVERSION`, `cargo.revert`, observación
  obligatoria, `reversionOfId` apunta al original, restaura el estado previo, conserva el historial,
  revertir una reversión prohibido. **Hecho** — `POST /api/v1/movements/:id/revert`
  (`cargo.revert`), `MovementTransactionService.executeReversion` (`58ebad2`). La reversión solo
  acepta movimientos con inverso exacto: los escritos en un solo segmento (`distribution.adjust` /
  `distribution.close`) devuelven 422 BR-012 porque su `quantityMoved` es el valor absoluto del
  segmento y el inverso se cancela solo (`40a6d7f`).
- [x] P5-T6 Transacciones de segmentos `CargoLocation` (BR-039): alta/baja/ajuste de segmento
  escribiendo `Movement` + `Observation` obligatoria en la misma transacción. **Hecho** —
  `executeSegmentCreate/Adjust/Close` en `MovementTransactionService` (`9f5e9c7`).
- [x] P5-T7 Movimientos parciales y descarga parcial (BR-037/038/042): parcial NO mueve el 100%;
  residual derivado `totalQuantity − Σ CargoLocation ACTIVE` (BR-042) expuesto como
  `inTruckAmount`/`inTruckUnit` en el detalle de carga; suma distribuida ≤ total (BR-034).
  **Hecho** — residual ya existía en `CargoService.toResponse`; se agregó cobertura e2e y el
  alta directa a sector (`locationId` en `POST /cargos`).
- [x] P5-T8 Alerta de capacidad (`AlertType.CAPACITY`, §9): detección sobre `occupiedCapacity`
  ≥ umbral configurable, sin auto-movimiento, umbral parametrizable por ubicación. **Hecho** —
  `CapacityAlertService` evaluado dentro de `refreshOccupancy`; alerta location-scoped, umbral en
  `Location.properties` (default 0.90), `HIGH` desde el umbral y `CRITICAL` desde 100%.

## Criterios de aceptación (PHASES.md §9 / P5-H1..H8)
- Cada transición §7 testeada; transiciones inválidas rechazadas (409 `INVALID_TRANSITION`).
- Movimiento sin observación rechazado; historial reconstruible; carga inexistente rechazada.
- Mover a ubicación inactiva o a capacidad completa rechazado, con tests; unidades incompatibles
  rechazadas.
- El operador completa un movimiento con observación y cantidad/porcentaje en ≤3 pasos.
- Reversión vista en el historial, con observación obligatoria.
- Movimiento parcial NO mueve el 100%; descarga parcial recalcula el residual; suma ≤ total.

## Alcance de esta corrida
- **5A (backend, P5-T1..P5-T3)**: máquina de estados + endpoint de movimientos + validaciones.
- **5B (backend, P5-T6..P5-T7)**: transacciones de segmentos + parciales + residual derivado.
- **5C (backend, P5-T5)**: reversión ADMIN.
- **5D (frontend, P5-T4)**: timeline + diálogos.
- **5E (backend, P5-T8)**: alerta de capacidad — **hecho**, decisión de alcance tomada con el humano
  (implementación mínima en PHASE 5; endpoint, job, notificaciones y settings quedan en PHASE 9).

## Decisiones tomadas en 5E (con el humano)
1. **D12 resuelta por migración**: `Alert.cargoId` pasa a nullable y se agrega `locationId` nullable
   con FK `RESTRICT`, CHECK XOR `ck_alerts_scope` e índice único parcial
   `uq_alerts_open_location_type`. Se descartó la convención vía `Alert.metadata` porque no puede
   deduplicar ni representar "una alerta por ubicación".
2. **D11 resuelta por convención de código**: `capacity <= 0` ⇒ sin límite, ya commiteada en FASE 4
   y 5A–5D. No se agrega `UNLIMITED` al enum `QuantityUnit`.
3. **Umbral**: `Location.properties.capacityAlertThreshold` como fracción en `(0,1)`, default `0.90`.
4. **Sin auto-resolve**: `API.md:323` + OQ-026 fijan la resolución como decisión humana con
   observación y roles; la alerta queda `OPEN` hasta que la resuelva el endpoint de PHASE 9.

## Pendiente canónico (requiere decisión del humano, no bloquea código)
- **D11 documental**: 8+ lugares de `cargoops-docs` afirman `capacityUnit = 'UNLIMITED'`
  (`DTOs.md` L169-170, `VALIDATION.md` L164, `OPENAPI.md` L299, `UI-GUIDELINES.md` L191,
  `DESIGN-TOKENS.md` L384, `JOBS.md` L167/L177 — este último con un SQL que no compila contra el
  schema real). El código ya eligió `capacity <= 0`. Corregir los docs es una decisión del humano.
- **Higiene**: `odd/tasks/cargoops-fase5-movements.md` declara P5-T8 y una cadena de PRs #5–#14
  "CUMPLIDAS" con commits que no existen en estos repos (línea descartada en el rebuild). Debe
  retirarse o marcarse para que nadie lo lea como baseline.

## Estado
- [x] P5-T1
- [x] P5-T2
- [x] P5-T3
- [x] P5-T4
- [x] P5-T5
- [x] P5-T6
- [x] P5-T7
- [x] P5-T8

## Progreso y evidencia (orquestador)
- **5A (P5-T1..P5-T3)** — merge `--no-ff` a `develop` en `04b4380`. Endpoints
  `POST/GET /api/v1/cargos/:id/movements`. Cubierto por la suite vigente.
- **5B (P5-T6, P5-T7)** — ruta: **delegado directo (un writer)**; disparador: 2+ archivos no
  triviales. Commit de unidad de trabajo `9f5e9c7` (`feat(distribution): move segment writes into
  the locked transaction`), merge `--no-ff` a `develop` en **`5d3c9da`**. `main` intacto; sin push.
  - Nuevos métodos públicos en `MovementTransactionService`: `executeSegmentCreate` (kind
    `INGRESS`), `executeSegmentAdjust` (`MOVE`), `executeSegmentClose` (`EXIT`), +
    `executeInitialLocationWithin` para el alta directa. Reutilizan `acquireLock`,
    `acquireLocationLocks` y `refreshOccupancy`; escriben `Observation` + `Movement` 1:1 en la
    misma transacción con backlink.
  - `PATCH /api/v1/cargos/:id/locations/:cargoLocationId` (`cargo.move`); DTO
    `UpdateCargoLocationDto` nuevo. `CargoLocationService` delega create/close/update al seam.
  - Alta directa a sector: `locationId` opcional en `POST /cargos` → `STORED` con un único
    `CargoLocation` ACTIVE y **sin** `Movement` (API.md §5.2 / DTOs.md §4.4 / OQ-10; arista
    `REGISTERED → STORED` con `kinds: []`). BR-042 exigido en el alta directa.
  - Residual `inTruckAmount`/`inTruckUnit` (`totalQuantity − Σ ACTIVE` en la unidad declarada) ya
    estaba en `CargoService.toResponse`; se agregó cobertura e2e.
- **Gates 5B observados por el orquestador** (en `cargoops-backend`): `npx prisma validate` ok;
  `npm run lint` ok; `npm run format:check` ok; `npm run build` ok; `npm run test` **260/260** (19
  archivos); `npm run test:e2e` **69/69** (6 archivos). Baseline previo: 236/236 y 64/64.
- **5C (P5-T5)** — ruta: **delegado directo (un writer)**; disparador: 2+ archivos no triviales.
  Commits `58ebad2` (`feat(movements): add ADMIN movement reversion (BR-012, ADR-011)`) y corrección
  del orquestador `40a6d7f`; merge `--no-ff` a `develop` en **`58f65de`**. `main` intacto; sin push.
  - `POST /api/v1/movements/:id/revert` con `cargo.revert` (ya estaba en `ADMIN_PERMISSIONS`; sin
    cambio de seed). Controller en `cargo`, transacción en `movements` (MODULES.md §5.8).
  - `executeReversion` valida todo antes de escribir: es una reversión → 422; ya revertido → 422;
    no es el último reversible → 422; observación en blanco → 422 (BR-006); estado restaurado
    `IN_TRANSIT`/`DELETED` → 409. Id desconocido o carga soft-deleted → 404.
  - Al original solo se le escriben `reversedById`/`reversedAt` (ADR-011: su dato queda intacto);
    audit `REVERT` con `reversionOfId`/`reversedById`.
  - **Bug detectado en el gate y corregido antes del merge**: el writer lo reportó como
    "limitación documentada", pero la traza muestra que para `distribution.adjust`/`distribution.close`
    el deshacer es un **no-op silencioso** — `quantityMoved` guarda el valor absoluto del segmento y
    `fromCargoLocationId === toCargoLocationId`, así que restarlo y volver a sumarlo a la misma fila
    se cancela, mientras la reversión se escribía igual y gastaba la única oportunidad de revertir.
    `40a6d7f` rechaza esos movimientos con 422 BR-012, amparado en la palabra "reversible" de
    VALIDATION §4.4.1. `distribution.create` (origen nulo) sigue revertible, con test que lo prueba
    para descartar un rechazo excesivo.
- **Gates 5C observados por el orquestador** (en `cargoops-backend`): `npx prisma validate` ok;
  `npm run lint` ok; `npm run format` + `format:check` ok; `npm run build` ok; `npm run test`
  **272/272** (19 archivos); `npm run test:e2e` **75/75** (6 archivos). Baseline previo: 270/270 y
  74/74 (260/260 y 69/69 antes de 5C).
- **Tamaño**: 5B 1645 + 5C 940 líneas autoradas (advisory ~400). Sin PR abierto; revisar si 5C se
  convierte en PR (posible `size:exception` o slicing).
- **5D (P5-T4, frontend)** — ruta: **delegado directo (un writer)**; disparador: 2+ archivos no
  triviales sobre una UI que no existía. La primera delegación fue abortada por el runtime, pero el
  writer alcanzó a dejar el trabajo completo y consistente en el índice; el orquestador lo verificó
  leyendo los archivos y corriendo los gates, y lo partió en dos unidades de trabajo:
  - `2d668ad` (`fix(locations): load cargo distribution inputs in ngOnInit`): `CargoDistributionComponent`
    leía `input.required()` desde el constructor → NG0950 en runtime. Era un bug latente de FASE 4
    que quedaba invisible porque no existía spec de la página de detalle; la spec nueva lo expuso.
  - `56ae05b` (`feat(cargo): add movement timeline, dialogs and move flow`), merge `--no-ff` a
    `develop` en **`783553c`**. `main` intacto; sin push.
  - Base `shared/ui/dialog`: `role="dialog"` + `aria-modal` + `aria-labelledby`, focus trap cíclico,
    `Esc` no cancela mientras `submitting`, foco inicial en `[data-dialog-initial-focus]` y
    restauración al invocador; el scrim **no** cierra (DP-UI-05).
  - `ObservationDialog`: observación obligatoria con trim, rechazo de solo-espacios, contador 500
    (DP-UI-02, nunca truncar), mapeo de sobres BR-006/BR-007 al campo y banner de alto impacto en el
    mismo diálogo (DP-UI-03).
  - `MovementTimeline`: `GET /cargos/:id/movements` con estados cargando/vacío/error, paginación y
    "Revertir" solo en la última entrada y solo con `cargo.revert`.
  - `MoveCargoForm`: espejo de la matriz confirmada de VALIDATION §4.4.1 + §4.4.3 y de BR-044
    (`STATUS_LOCATION_TYPES`) para pre-filtrar destinos; parcial por cantidad o porcentaje (BR-037).
  - `CargoService.movements/move/revert` y los modelos `Movement`/`MoveCargoRequest`/`RevertMovementRequest`,
    verificados contra la proyección `MovementResponse` real del backend.
- **Gates 5D observados por el orquestador** (en `cargoops-frontend`): `npm run build` ok;
  `ng test` **154/154** en **24 archivos** (baseline previo: 18 archivos de spec en `06184d1`; se
  agregaron 6). Sin lint/format scripts en el frontend (deuda conocida); no se agregaron.
- **5E (P5-T8, backend)** — ruta: **delegado directo (un writer)**, interrumpido dos veces por el
  runtime; el orquestador terminó el slice inline (chunk acotado y ya entendido). Tres unidades de
  trabajo en `feature/p5-capacity`, merge `--no-ff` a `develop` en **`9f81a53`**:
  - `83122f8` (`feat(db): scope alerts to locations or cargo`): schema + migración
    `20261019000000_alert_location_scope`.
  - `6d481ad` (`feat(alerts): detect location capacity breaches`): servicio, módulo, fake in-memory
    y 17 specs unitarios.
  - `61897d8` (`feat(movements): raise capacity alerts inside the occupancy transaction`): hook en
    `refreshOccupancy` + 5 specs e2e.
  - El hook vive en `refreshOccupancy`, no en cada call site: los 7 caminos (move, alta directa,
    segmento, ajuste, cierre, reversión) pasan por ahí, así que ninguna escritura aceptada puede
    cambiar la ocupación sin evaluar.
- **Gates 5E observados por el orquestador** (en `cargoops-backend`, sobre los bytes de `develop`):
  `npx prisma validate` ok; `npm run format:check` ok; `npm run lint` ok; `npm run build` ok;
  `npm run test` **289/289** (20 archivos); `npm run test:e2e` **80/80** (6 archivos).
  Baseline previo: 272/272 y 75/75.
- **Migración aplicada y verificada** (2026-10-19, con autorización del humano): Docker Desktop
  levantado y `postgres:16-alpine` — la misma imagen que documenta `OPENAPI.md` L317 para el CI — en
  `127.0.0.1:5432`, contenedor `cargoops-postgres`. `npx prisma migrate deploy` aplicó las 3
  migraciones (`init`, `add_refresh_tokens`, `alert_location_scope`); `migrate status` = up to date;
  `prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma` = **no difference
  detected** (sin drift). El seed corre verde contra el schema nuevo (25 permisos, 3 roles, 17
  ubicaciones, 15 cargas).
  Verificación funcional de las constraints contra la base real, no sólo DDL: ambos `NULL` → viola
  `ck_alerts_scope`; ambos seteados → viola `ck_alerts_scope`; alerta solo-location `OPEN` → inserta;
  duplicado `OPEN` con el mismo `(location_id, type)` → viola `uq_alerts_open_location_type`.
  Nota para Prisma 7: `--from-url` ya no existe en `migrate diff`; el flag es
  `--from-config-datasource`.
  El contenedor `cargoops-postgres` queda corriendo para las próximas fases
  (`docker stop cargoops-postgres` para apagarlo).
- **Tamaño**: 5E ~750 líneas en el merge (11 archivos), dentro del advisory.
- **RDD**: off clone-local; sin review nativo ejecutado.
