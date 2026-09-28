# CargoOps — FASE 3: Cargo Management (EPIC-003)

> Feature document (ODD). Repos: `cargoops-backend` (NestJS) y `cargoops-frontend` (Angular). Docs canónicos: `cargoops-docs/docs/backend/{API,DTOs,VALIDATION}.md`, `MASTER-SPEC.md` §4/§7/§10, `PHASES.md` FASE 3.
> TDD: **no configurado** → checks funcionales (lint/format/build/unit/e2e). RDD: **off (clone-local)** en backend → sin review nativo.
> Delivery: **chained PRs stacked-to-main** (decisión 10, aprobada en sesión previa; chain strategy cacheada).

## Objetivo

Implementar el módulo de dominio **Cargo Management**: alta, consulta (listado con filtros + paginación), actualización, soft delete/restore y notas de cargas; CRUD de trucks; UI frontend básica (CargoTable/Search/Filters/Detail). Cerrar EPIC-003.

## Contexto / estado actual (mapeo 2026-09-25)

- **Schema ya canónico desde FASE 1**: `Cargo`, `Truck`, `CargoLocation`, `Observation`, `Movement`, `Alert` modelados con migración init (soft delete `deletedAt` en Cargo/Truck per ADR-011; índice único case-insensitive `uq_cargo_code_upper` via raw SQL — BR-002). **No hay migración nueva en FASE 3** salvo que el desarrollo lo exija.
- **Backend**: módulos `auth` (guards globales JWT+Permissions deny-by-default), `audit` (AuditService.record), `common` (AppException, SuccessEnvelopeInterceptor, AllExceptionsFilter, requestId), `prisma`, `health`. `app.setup.ts` configura prefijo `/api/v1`, helmet, CORS credentialed, ValidationPipe.
- **Catálogo permisos (25, sembrados FASE 2)**: `cargo.read/create/update/move/delete_soft/restore/revert/export_pdf/to_rezago/to_secuestro`, `truck.read/create/update`, etc. Bundles: VIEWER (read-only BR-010), OPERATOR (Viewer + create/update/move/to_rezago/audit propio BR-011), ADMIN (todo + delete_soft/restore/to_secuestro BR-012).
- **Contratos (API.md/DTOs.md, W5)**: `POST/GET /api/v1/cargos`, `GET/PATCH /api/v1/cargos/:id`, `POST /api/v1/cargos/:id/notes`, `GET/POST/PATCH /api/v1/trucks(|/:id)`. Envelope `{ data }` + `meta` paginado. Errores: 400 `VALIDATION_ERROR`, 401/403, 404 `CARGO_NOT_FOUND`/`TRUCK_NOT_FOUND`, 409 `TRUCK_PLATE_DUPLICATE`/`CARGO_CODE_DUPLICATE`, 422 `BUSINESS_RULE_VIOLATION` (BR-006 obs vacía).
- **Frontend**: Angular standalone/zoneless; rutas lazy (`main-layout`, `login`, `not-found`); `auth` core (guard, interceptor single-flight, store signals); sin `features/` ni páginas de negocio todavía.

## Bloqueos canónicos (verificados 2026-09-25)

Todas las OQ de FASE 3 están **resueltas** (consolidación v0.5, 2026-09-24). PHASES.md quedó desactualizado con 🔴/🟡:
- OQ-001 → **BR-002** (regex código `^[A-Z0-9][A-Z0-9./-]{2,31}$` 3–32, uppercase, único case-insensitive).
- OQ-003 → **BR-050** (una carga → UN camión a la vez `truckId`; un camión → N cargas).
- OQ-004 → **BR-043** (egreso EXIT: FASE movimientos, no FASE 3).
- OQ-022 → **BR-006** (observación OBLIGATORIA siempre, incluido el alta).

## Alcance autorizado

Autorizado por el usuario ("continuar" tras cierre de FASE 2; ODD protocol paso 1 explicit change intent). Implementación multipaso sustancial → este feature doc + mirror Engram (paso 5 ODD).

**In scope (FASE 3, EPIC-003):**
- Backend: módulo `cargo` (alta, listado, detalle, update, soft delete, restore, notas) + módulo `trucks` (CRUD sin delete — no hay permiso canónico).
- Frontend: seed de cargas de ejemplo + páginas de cargos (tabla, búsqueda, filtros, detalle básico).
- Tests/checks funcionales, work-unit commits, chained PRs stacked-to-main.

**Out of scope (FASES posteriores):**
- Movements/estados de ubicación (F4), distribución CargoLocation (F5), alerts (F7), export PDF, mapas, dashboard.
- Soft delete de trucks (sin permiso; BR-013 canónico solo para User/Truck/Cargo/Location/MapElement — se deja documentado).
- Edición de `code` (inmutable v1, OQ-027) y de `truckId` (flujo movements, API.md §5.4).

## Tasks (checklist estable)

- [x] **P3-T1** Backend módulo `cargo`: estructura CargoModule (controller/service/dtos), DTOs Create/Update/CargoListQuery/Notes, respuesta CargoResponseDto, scope `notDeleted` (ADR-011), registro en AppModule. Checks: lint/format/build/unit. ✅ (PR-A, verificado 2026-09-25)
- [x] **P3-T2** Backend `POST /api/v1/cargos`: validación BR-001/002 (código normalizado uppercase, único), estado inicial service (REGISTERED; IN_TRUCK si truckId — BR-016 nota DTOs), **observación obligatoria en alta (BR-006/OQ-022)** → persiste Observation en misma transacción + audit CREATE + createdById desde token. Permiso `cargo.create`. Checks: unit + e2e + build. ✅ (PR-A)
- [x] **P3-T3** Backend consultas: `GET /api/v1/cargos` (paginación ListQueryDto + filtros code/search/status/truckId/createdById/from/to; envelope `{data:{items,meta}}`), `GET /api/v1/cargos/:id` (detalle + observations recientes + movementsCount; sin alertsSummary — decisión 34) — permiso `cargo.read`. Checks: unit + e2e. ✅ (PR-B) (spot-check parent 5/5 verde)
- [x] **P3-T4** Backend `PATCH /api/v1/cargos/:id` (whitelist: name/totalQuantity+totalUnit juntos/entryDate/estimatedExpiryDate/metadata; **sin code — BR-047 inmutable, sin truckId — decisión 26**; audit UPDATE) + `DELETE /api/v1/cargos/:id` (soft, `cargo.delete_soft` ADMIN, audit, sin tocar status) + `POST /api/v1/cargos/:id/restore` (`cargo.restore` ADMIN, idempotente, audit RESTORE) + `POST /api/v1/cargos/:id/notes` (NotesDto, `cargo.update`, 400 vacío/422 whitespace BR-006, audit UPDATE). Checks: unit + e2e. ✅ (PR-B) (spot-check parent 5/5 verde)
- [x] **P3-T5** Backend módulo `trucks`: `GET /trucks` (paginación + filtros plate/withCargo — decisión 41, cargosCount), `POST /trucks` (`truck.create`, BR-029 patente MERCOSUR según decisión 39, 409 `TRUCK_PLATE_DUPLICATE`), `GET /trucks/:id` (con cargas asignadas no borradas), `PATCH /trucks/:id` (`truck.update` — plate mutable, decisión 42). Sin DELETE (decisión 28). Checks: unit + e2e. ✅ (PR-C) (spot-check parent 5/5 verde)
- [x] **P3-T6** Frontend: seed cargas de ejemplo (backend `prisma/seed-cargo.ts`, idempotente) + páginas `/cargos` (CargoTable, CargoSearch, CargoFilters, listado paginado) y `/cargos/:id` (CargoDetail básico) bajo main-layout con auth guard; servicio cargo + modelo. Checks: lint/format/build/test frontend + e2e backend (si aplica). ✅ (PR-C seed + PR-D UI) (spot-check parent 5/5 backend + 4/4 frontend)

## Decisiones (continúa de decisión 20)

| # | Decisión | Rationale |
| --- | --- | --- |
| 21 | FASE 3 arranca sin bloqueos; PHASES.md desactualizado vs OPEN-QUESTIONS v0.5 | OQ-001/003/004/022 resueltas (2026-09-23/24) |
| 22 | Delivery: 4 PRs chained stacked-to-main (backend cargo core → backend consultas/update → backend trucks → frontend UI) | Sesión FASE 2: decisión 10; each slice < ~400 authored lines target |
| 23 | Estado inicial de Cargo lo fija el service, NO el DTO (BR-016) | DTOs.md §4.4 nota: REGISTERED; IN_TRUCK si truckId y sin segmentos |
| 24 | Alta crea Cargo + Observation en la MISMA transacción (BR-006) | OQ-022: observación obligatoria siempre |
| 25 | `code` inmutable v1: PATCH no lo acepta (OQ-027); si se registró mal → soft-delete + re-alta | DTOs.md §4.4 UpdateCargoDto vs OQ-027 (más reciente/específica) |
| 26 | PATCH no acepta `truckId` (flujo movements/segments, FASE 4) | API.md §5.4 |
| 27 | Soft delete: `DELETE /cargos/:id` (soft, audit, `cargo.delete_soft` ADMIN); restore: `POST /cargos/:id/restore` (`cargo.restore` ADMIN, audit) | BR-013/ADR-011; contratos no enumerados en MASTER-SPEC §10 → convención REST idempotente |
| 28 | Trucks: sin DELETE (catálogo ADR-009 no tiene `truck.delete`); soft delete de trucks queda documentado como gap | No inventar permisos fuera del catálogo sembrado |
| 29 | Notas: `POST /cargos/:id/notes` con NotesDto `{text, metadata?}`, `cargo.update`; Observation sin movementId | API.md §5.8 |
| 30 | Listado F3: filtros code/search/status/truckId/createdById/from/to; `locationId` y `alert` quedan para F4/F7 (dependen de segments/alerts) | No sobre-implementar; DTO canónico CargoListQueryDto completo queda documentado |
| 31 | 422 `BUSINESS_RULE_VIOLATION` con `{rule: 'BR-006'}` para observación vacía (class-validator → 400 shape, semántica BR → 422) | ERROR-HANDLING.md §3/§4.9 |
| 32 | `description` NO se acepta en Create/Update en FASE 3: DTOs.md la lista pero el storage canónico (DATABASE.md §5.3 / schema) no tiene columna; sin migración en F3 → gap documental pendiente (fix DTOs.md o migración futura) | No inventar storage; feature doc permite migración solo si el desarrollo la exige |
| 33 | BR-002 en el boundary: regex uppercase-only del DTO responde 400 a minúsculas; `normalizeCargoCode` (trim+uppercase) queda como defensa para callers internos; ambigüedad OQ-001 ("normalizado a mayúsculas") vs DTOs.md literal → se adopta el contrato literal, queda documentada | DTOs.md §4.4 es la fuente del contrato HTTP; e2e asserta el comportamiento |\n| 34 | Detalle SIN `alertsSummary`/`openAlerts` en FASE 3: no hay generador de alerts (F7) → count siempre 0 sería código muerto; `@Param('id', ParseUUIDPipe)` en detalle → 400 `VALIDATION_ERROR` para id malformado (evita 500 de Prisma), adición de contrato documentada | Feature doc P3-T3 vs realidad: DTOs.md §4.12 pone alertsSummary en el DTO base que decisión 30 excluye; writer flageó el riesgo |\n| 35 | `NotesDto.metadata` declarado pero NO persistido: `Observation` no tiene columna metadata (schema canónico). Mismo posture que decisión 32: sin migración inventada; gap canónico pendiente (fix DTOs.md o migración futura) | Writer verificó el modelo Observation; contrato DTOs.md vs storage real |\n| 36 | `estimatedExpiryDate` y `name` no son clearables en F3 (validators aceptan string/date, nunca `null` explícito → PATCH no puede resetear a NULL). Deferido con el fix de DTOs.md; no bloquea el contrato | Writer flag; requeriría `@ValidateIf` por campo |\n| 37 | BR-042 en PATCH deliberadamente estricto: cambiar solo `totalUnit` se rechaza aunque ya exista total. Re-send del par hace explícita la reinterpretación de unidad; se relaja solo si el usuario lo decide | Writer flag; coherencia con el alta (ambos juntos) |\n| 38 | Notas auditan `AuditAction.UPDATE` sobre entity `cargo` (no hay acción NOTE en el catálogo; OQ-021 establece UPDATE como acción genérica) | MASTER-SPEC OQ-021 |\n| 39 | Patente truck en F3: validación por DTOs.md §4.5 (`^[A-Z0-9-]+$`, max 20, normalize uppercase en service); la regex BR-029 de MASTER-SPEC (`^[A-Z]{2,3}\\d{2}[A-Z0-9]$`) NO matchea sus propios ejemplos (AB123CD/AB1234C fallan la ancla `$`) → gap documental BR-029 vs DTOs.md; se adopta la fuente de la capa de validación, la unicidad la da el unique DB (P2002 → 409 `TRUCK_PLATE_DUPLICATE`) | Verificado en OPEN-QUESTIONS.md ID-009 y MASTER-SPEC BR-029: ejemplo canónico inconsistente con su regex; no inventar regex nueva |\n| 40 | Length caps truck implementados según STORAGE (50/50/120 VarChar) no según DTOs.md §4.5 (60/60/160): over-length → 400 en vez de 500 de Postgres. Gap documental DTOs.md vs schema; sin migración en F3 | Mismo posture que decisión 32: nunca aceptar un valor que la DB no puede guardar |\n| 41 | `withCargo` (API.md §9.1 nombra el param sin definirlo): tri-state — `true` = ≥1 cargo no borrado, `false` = ninguno, ausente = sin filtro; `?withCargo=maybe` → 400; `cargosCount` con `_count` filtrado en la misma query (sin N+1). Semántica inventada por writer; si el producto espera otra cosa (ej. eager-load), cambiar service + 1 test | Gap documental API.md; decisión registrada para revisión en integración frontend |\n| 42 | Plate de truck es mutable vía PATCH (única identidad): a diferencia de cargo code (BR-047) los trucks no tienen flujo soft-delete/re-registro; audit UPDATE registra before/after → trazable | Writer flag; decisión razonable con trazabilidad de auditoría |
| 43 | Seed idempotente por skip-on-exists (no upsert): una carga solo es correcta junto con su observación BR-006; re-ejecutar imprime `0 created this run`. 3 trucks MERCOSUR + 5 cargas CARGO-001..005 con observations | Writer; la idempotencia es observable, no prometida |
| 44 | Bug preexistente NO corregido (fuera de scope): `login.page.ts` lee `error.error?.message` pero el envelope real es `{ error: { code, message, requestId } }` → el login siempre cae en mensaje genérico. Nuevo `http-error.ts` lo lee bien (hace la inconsistencia visible). Vale un PR propio | Writer flag; no tocar auth en un PR de cargos |
| 45 | Detalle de cargo muestra "Asignado"/"—" para camión: la API solo devuelve `truckId`, no la patente. Si la patente importa en detalle → extender read model del backend (gap pendiente, no bloquea P3-T6) | Writer flag; no inventar campo sin contrato |
| 46 | Breadcrumb "FASE 1 · Foundation" en el layout: deuda de fases anteriores, no introducida en F3 | Writer flag |
| 47 | Spec de seguridad de rutas usa drenado acotado (20 rondas) + aserción explícita de `POST /auth/refresh` y ausencia de reads a `/cargos` en el camino sin sesión; holgado, puede volverse flaky si se agrega un preloader/resolver profundo | Writer; monitorear en próximas fases |

## Delivery: slices / PRs

1. **PR-A** `feat/p3-cargo-core` (backend): P3-T1 + P3-T2 — módulo cargo + alta con observación (≈450-550 líneas).
2. **PR-B** `feat/p3-cargo-query` (backend): P3-T3 + P3-T4 — consultas + update/delete/restore/notes (≈500-600 líneas).
3. **PR-C** `feat/p3-trucks` (backend): P3-T5 — CRUD trucks (≈300-400 líneas).
4. **PR-D** `feat/p3-frontend-cargo` (frontend): P3-T6 — seed + UI (≈500-650 líneas).

Orden stacked-to-main; merge squash en orden; lección decisión 20: retarget hijos a main ANTES de borrar rama base; rebase `--onto origin/main <base-commit>` para diffs limpios.

## Chequeos aplicables (por work-unit commit)

- Backend: `npm run lint`, `npm run format:check`, `npm run build`, `npm test`, `npm run test:e2e` (Postgres local, `.atl`), reportar `<comando>: <resultado>` exacto.
- Frontend: `npm run lint`, `npm run format:check`, `npm run build`, `npm test` (si aplica), reportar exacto.
- Known environmental: superagent no envía User-Agent → tests e2e lo setean.

## Evidencia / progreso

Backend (`feat/p3-cargo-core`, `feat/p3-cargo-query`, `feat/p3-trucks`):
- `f0104b4` feat(cargo): cargo module + POST /api/v1/cargos create (P3-T1/T2) — PR-A (#4)
- `b396c02` feat(cargo): list and detail query endpoints with pagination (P3-T3)
- `f0c73b2` test(cargo): e2e coverage for list/detail query endpoints (P3-T3)
- `b471a3d` feat(cargo): PATCH, soft delete/restore and notes endpoints (P3-T4)
- `8bc9417` test(cargo): e2e coverage for mutations (P3-T4) — PR-B (#5)
- `1304a29` feat(trucks): trucks module CRUD without delete (P3-T5)
- `1304a29~1`→`1304a29` test(trucks) e2e (P3-T5) — PR-C (#6)
- `5d25cc5` feat(seed): example cargos and trucks seed with observations (P3-T6a) — viaja en PR-C (#6)

Frontend (`feat/p3-frontend-cargo`):
- `093cadc` feat(cargos): cargo list/detail pages, service, model and nav link (P3-T6b) — PR-D (cargoops-frontend #3)

PRs abiertos: backend #4 (PR-A), #5 (PR-B), #6 (PR-C, incluye seed); frontend #3 (PR-D). Orden de merge: #4 → #5 → #6 (backend), luego #3 (frontend); lección decisión 20 al mergear (retarget hijos a main antes de borrar base).

## Próximo paso

- ✅ FASE 3 implementada completa (P3-T1..T6) y **MERGEADA**:
  - Backend main: `4b5ed64` (#4, PR-A) → `f48291c` (#5, PR-B) → `eeecb37` (#6, PR-C + seed). Retarget + rebase `--onto origin/main` aplicados (decisión 20); CI verde en cada PR.
  - Frontend main: `9066ccc` (#3, PR-D).
- Pendientes registrados (deudas/gaps de decisiones 28/32/35/36/39/40/41/44/45/46): ver tabla de decisiones; varios requieren fix de docs canónicos (DTOs.md/API.md/MASTER-SPEC) o PRs propios.
- Al cerrar FASE 3: mirrors Engram + session summary; FASE 4 según PHASES.md a decisión del usuario.