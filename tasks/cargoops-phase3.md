# FASE 3 — Cargo Management (EPIC-003/004)

## Objetivo
Implementar PHASE 3 según `PHASES.md` §7: módulo cargo (entidad + validaciones BR-001/002 + observación obligatoria), consultas con filtros/paginación, update + soft delete/restore, módulo trucks con asignación, y UI básica (CargoTable/CargoSearch/CargoFilters/CargoDetail).

## Alcance autorizado
- Backend: `src/modules/cargo`, `src/modules/trucks` (nuevos), seed de cargas de ejemplo.
- Frontend: `src/app/features/cargo`, `src/app/pages/cargos`, componentes UI.
- Git flow: `feature/p3-*` desde `develop`, merges `--no-ff` a `develop`. Sin push/remote. Sin atribución AI.

## Tareas
- [x] P3-T1 Módulo cargo: entidad (Cargo, CargoStatus §7), repository Prisma, DTOs. Soft delete (ADR-011).
- [x] P3-T2 `POST /api/v1/cargos` con BR-001/BR-002 (código canónico OQ-001) + observación obligatoria (BR-006/OQ-022).
- [x] P3-T3 Consultas: listado con filtros (code, status, ubicación vía CargoLocation BR-032, truckId) + paginación/envelope §10; detalle.
- [x] P3-T4 Update (campos permitidos por rol), soft delete/restore (BR-013, ADMIN), notas/observaciones sin movimiento (Observation.cargoId).
- [x] P3-T5 Módulo trucks: CRUD + asignación de cargas a camión (§4.2, OQ-003). Truck único por patente; ocupación consultable.
- [x] P3-T6 Seeds de cargas de ejemplo §5 + CargoTable/CargoSearch/CargoFilters/CargoDetail básicos.

## Criterios de aceptación (PHASES.md §7)
- Migración de Cargo lista; tipos generados; soft delete (ADR-011).
- Carga sin código rechazada (422); duplicados rechazados; observación obligatoria en el alta persistida con la carga.
- Filtros/paginación testeados; Viewer consulta, Operator crea (BR-010).
- Update con validaciones; delete→auditoría→restore conserva historial.
- Truck único por patente; asignación testeada; ocupación de camión consultable.
- UI lista con datos seed; estado vacío y loading correctos.
- Sin modificar MASTER-SPEC / OPEN-QUESTIONS.
- No AI attribution en commits.

## Restricciones técnicas
- No borrar workarounds de Smart App Control (`test/ts-decorator-metadata.plugin.ts`, alias `@rollup/wasm-node`, vitest/vite 7 pinned).
- No tocar volúmenes Docker `cargoops-dev_pg_data` / `cargoops-dev_redis_data`.
- Artefactos (código, commits, comentarios) en inglés.
- Guards globales deny-by-default ya activos: toda ruta nueva necesita `@RequirePermissions()` o `@Public()`.

## Plan de ejecución (split por riesgo de contexto)
- **3A (backend)**: P3-T1..P3-T5 + seed de cargas de ejemplo.
- **3B (frontend)**: P3-T6 UI.

## Rutas / permisos esperados
- `cargo.read`, `cargo.create`, `cargo.update`, `cargo.delete` (+ `cargo.to_rezago` / `cargo.to_secuestro` para transiciones de estado, fases posteriores).
- `trucks.read` / `trucks.manage` (según catálogo ADR-009 — verificar en DB antes de usar).

## Evidencia (FASE 3A — backend)

Rama `feature/p3-cargo-backend` creada desde `develop`, merge `--no-ff` a `develop`. Sin push/remote. Sin atribución AI.

### Commits en `feature/p3-cargo-backend`
- `870899a` feat(cargo): add cargo registry endpoints with soft delete and notes
- `3d47ef7` feat(trucks): add fleet registry with occupancy view
- `af03ea4` test(cargo): cover cargo and truck services and HTTP flows
- `4334183` feat(seed): seed MASTER-SPEC example cargos idempotently

### Merge
- `69a939d` Merge branch 'feature/p3-cargo-backend' (--no-ff, HEAD de `develop`). Working tree limpio.

### Endpoints implementados (prefijo global `/api/v1`)
| Método | Ruta | Permiso |
| --- | --- | --- |
| POST | `/cargos` | `cargo.create` |
| GET | `/cargos` | `cargo.read` |
| GET | `/cargos/:id` | `cargo.read` |
| PATCH | `/cargos/:id` | `cargo.update` |
| DELETE | `/cargos/:id` (soft) | `cargo.delete_soft` |
| POST | `/cargos/:id/restore` | `cargo.restore` |
| POST | `/cargos/:id/notes` | `cargo.update` |
| POST | `/cargos/:id/observations` (alias) | `cargo.update` |
| POST | `/trucks` | `truck.create` |
| GET | `/trucks` | `truck.read` |
| GET | `/trucks/:id` | `truck.read` |
| PATCH | `/trucks/:id` | `truck.update` |

Nota: el catálogo canónico usa `truck.*` (no `trucks.read`/`trucks.manage`). El soft delete de carga usa `cargo.delete_soft` y el restore `cargo.restore` (ver `src/modules/permissions/rbac.catalog.ts`). No existe `truck.delete_soft`, por eso no hay DELETE de camión.

### Gates observados (todos en verde)
- `npx prisma validate` → schema válido (🚀).
- `npm run db:seed` (x2) → idempotente: 25 permisos, 3 roles, 17 ubicaciones, 15 cargos en ambas corridas.
- `npm run lint` → exit 0.
- `npm run format:check` → "All matched files use Prettier code style!".
- `npm run build` → exit 0.
- `npm test` → 73 passed (15 files).
- `npm run test:e2e` → 37 passed (4 files).

### Archivos principales
- `src/modules/cargo/*` (controller, service, DTOs, spec), `src/modules/trucks/*`, `src/app.module.ts`.
- `prisma/seed.ts` (usuario sistema + 15 cargos de ejemplo §5, upsert por UUID determinista).
- `test/cargo.e2e-spec.ts`, `test/support/in-memory-prisma.ts`.

### Alcance restante
- P3-T6 (UI: CargoTable/CargoSearch/CargoFilters/CargoDetail) fuera de esta fase (FASE 3B).

## Evidencia (FASE 3B — Frontend UI)

Rama `feature/p3-cargo-ui` desde `develop` (`54e3d4a`). Merge `--no-ff` a `develop`. Sin push/remote. Sin atribución AI.

### Commits
| Hash | Subject |
| --- | --- |
| `4844454` | feat(cargo): add cargo models and API envelope types |
| `fa21054` | feat(cargo): add cargo API service |
| `b18d0c6` | feat(cargo): add cargo formatting utilities and status badge |
| `1baffc4` | feat(cargo): add cargo table component |
| `dd26118` | feat(cargo): add cargo search component |
| `cdd4f83` | feat(cargo): add cargo filters component |
| `fc021df` | feat(cargo): add cargo detail component |
| `dbac6f9` | feat(cargo): add cargos list store, pages and routes |

### Merge
- `8b7a17d` Merge branch 'feature/p3-cargo-ui' (`--no-ff`, 2 padres: `54e3d4a` + `dbac6f9`). Working tree limpio.

### Componentes y archivos
| Artefacto | Ruta |
| --- | --- |
| CargoTable | `src/app/features/cargo/components/cargo-table/` |
| CargoSearch | `src/app/features/cargo/components/cargo-search/` |
| CargoFilters | `src/app/features/cargo/components/cargo-filters/` |
| CargoDetail | `src/app/features/cargo/components/cargo-detail/` |
| CargoStatusBadge (extra, requerido para label+color) | `src/app/features/cargo/components/cargo-status-badge/` |
| CargoService | `src/app/features/cargo/services/cargo.service.ts` |
| CargosStore (list state) | `src/app/features/cargo/state/cargos.store.ts` |
| Formatters es-AR + labels | `src/app/features/cargo/utils/cargo-format.ts` |
| Modelos (Cargo, CargoStatus, envelope) | `src/app/core/models/cargo.model.ts`, `src/app/core/models/api.model.ts` |
| Páginas + rutas | `src/app/pages/cargos/`, `src/app/app.routes.ts` |

### Rutas
| Ruta | Guard | Carga |
| --- | --- | --- |
| `/cargos` | `authGuard` | lazy `CargosPageComponent` (chunk propio) |
| `/cargos/:id` | `authGuard` | lazy `CargoDetailPageComponent` (chunk propio; `:id` es UUID, `GET /cargos/:id` usa `ParseUUIDPipe`) |

### Gates observados (sobre el HEAD de la rama, todos exit 0)
- `npx prettier --check .` → "All matched files use Prettier code style!".
- `npx ng build` → build completo; chunks lazy `cargos-page-component` y `cargo-detail-page-component` presentes.
- `npx ng test --watch=false` → 13 archivos / 54 tests passed (baseline previo: 7 archivos / 19 tests).
- Re-verificados post-merge sobre `develop` (`8b7a17d`): los tres gates vuelven a dar exit 0.

### Verificación contra backend en vivo
- `live_backend_verified: false`. No hay backend escuchando en localhost (sin listeners en 3000/4000/4200/8080), no existe `proxy.config` en `cargoops-frontend` ni docker-compose en el repo, por lo que la UI no puede alcanzar `/api/v1` sin agregar configuración fuera del alcance autorizado. El contrato se replicó 1:1 desde `cargo.controller.ts` / `cargo.service.ts` / DTOs del backend y quedó fijado en specs que verifican `envelope { data }`, gramática de query (`page`, `limit`, `code`, `status`, `truckId`), `PATCH` sin `truckId` y las URLs de `DELETE`/`restore`/`notes`.

### Descubrimientos — AMBOS RESUELTOS (ver sección de cierre más abajo)
- ~~`AuthService.login/refresh/me` NO desenvuelve el envelope `{ data }`~~ → **RESUELTO** en `feature/fix-auth-envelope`.
- ~~EOL: `core.autocrlf=true` re-checkoutea a CRLF y rompe `prettier --check .`~~ → **RESUELTO** de raíz en `feature/p3-line-endings`.

### Alcance diferido
- `CargoFilters.availableLocations` (depende de PHASE 4 `features/locations`).
- Filtros del listado en query params / deep-link (ROUTING §5.4).
- `/cargos/new`, `/cargos/:id/edit`, export PDF, `MovementTimeline`, `DistributionPanel`.

## Correcciones post-verificación (orquestador, 2026-10-15)

Ambas surgieron al re-verificar FASE 3 con comandos propios, no de los reportes de los subagentes.

### FIX-A — `AuthService` no desenvuelve el envelope `{ data }` (frontend) — RESUELTO
- **Impacto**: severo. `login()`/`refresh()` leían `res.accessToken` del envelope → `undefined` → `store.setSession(undefined, undefined)` → la sesión nunca se establecía y `authGuard` devolvía a `/login` (loop de login contra el backend real).
- **Por qué pasó**: `services/auth/` no tenía spec; los specs existentes mockeaban la respuesta ya desenvuelta, y `refresh-token.interceptor.spec` generaba `'Bearer undefined'`.
- **Evidencia de la causa**: `SuccessResponseInterceptor` es `APP_INTERCEPTOR` global (envuelve todo en `{ data }`); `AuthController` usa `@Res({ passthrough: true })` → sí se envuelve; no existe interceptor que desenvuelva; `CargoService` sí lo hacía con `map(r => r.data)`.
- **Fix**: rama `feature/fix-auth-envelope` → commit `5c46423` → merge `--no-ff` **`7e1507e`** en `develop`.
  `login`/`refresh`/`logout` usan `ApiEnvelope<T>` + `map(r => r.data)`; tipos de retorno corregidos; nuevo `auth.service.spec.ts` (6 tests) que falla sin el fix (probado revirtiendo solo `auth.service.ts`: 7 tests rojos en 3 archivos).
- **Pendiente menor**: `AuthService.me()` tiene el mismo defecto latente y no se usa en ningún lado (el backend tampoco expone `GET /auth/me` — el controller solo tiene login/refresh/logout).

### FIX-B — Política de line endings (frontend) — RESUELTO
- **Problema**: `cargoops-frontend` no tenía `.gitattributes`; con `core.autocrlf=true` cada checkout/merge reescribía los archivos tocados a CRLF y `prettier --check .` fallaba **solo por EOL** (4 archivos tras el fix A, 34 tras el merge de 3B). El gate era frágil y no reproducible.
- **Fix**: rama `feature/p3-line-endings` → commit `ec0243d` → merge `--no-ff` **`de4a39f`** en `develop`. Se agregó `.gitattributes` con `* text=auto eol=lf` (mismo criterio que `cargoops-backend`).
- **Verificado durable**: tras `git reset --hard`, `git ls-files --eol` reporta `w/lf` + `attr/text=auto eol=lf` y `prettier --check .` pasa.
- **Nota**: `cargoops-infra` tampoco tiene `.gitattributes` (deuda menor, no bloqueante).

### Deuda menor detectada (no bloqueante, no corregida)
- `cargoops-frontend/package.json` no tiene scripts `lint`/`format`/`format:check` (a diferencia del backend); el gate se corre a mano con `npx prettier`.
- `cargoops-backend/README.md` menciona `oxlint` pero el linter real es ESLint (drift de documentación, FASE 1). Fuera del scope de `npm run format:check` (que solo cubre `src/**/*.ts` y `test/**/*.ts`).
- `cargoops-infra` sin `.gitattributes`.

## Verificación final del orquestador (develop, 2026-10-15)
- **Backend** `develop` @ `69a939d`: `npm test` **73/73** (15 files) · `npm run test:e2e` **37/37** (4 files) · `npm run format:check` exit 0 · seed idempotente ×2 → tabla `cargo` con **15 filas**, índice funcional `uq_cargo_code ON UPPER(code)` presente (BR-002).
- **Frontend** `develop` @ `de4a39f`: `npx prettier --check .` exit 0 · `npx ng build` exit 0 · `npx ng test --watch=false` **60/60** (14 files).
- Árboles de trabajo limpios en ambos. `main` sin avanzar (sin release).

## Estado
- [x] P3-T1
- [x] P3-T2
- [x] P3-T3
- [x] P3-T4
- [x] P3-T5
- [x] P3-T6 (FASE 3B — frontend UI)
- [x] FIX-A envelope AuthService
- [x] FIX-B política de line endings (frontend)
