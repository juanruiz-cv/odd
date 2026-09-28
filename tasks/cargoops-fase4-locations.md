# FASE 4 — Locations (EPIC-004)

## Objetivo
Implementar el módulo de ubicaciones y distribución de CargoOps: CRUD de `Location` por ADMIN, derivación de capacidad/ocupación (BR-033/035/036/041), validaciones de negocio BR-004/005, seed de distribución CargoLocation y consultas de distribución (BR-040), más UI de listado/ocupación/distribución en frontend. Entregado en PRs chained stacked-to-main.

## Problema
FASE 3 entregó cargas y trucks, pero una carga aún no puede distribuirse en ubicaciones: no existe módulo `locations`, ni cálculo de capacidad, ni consultas de distribución ni UI. La distribución M:N (CargoLocation) ya está modelada en el schema (F1) pero sin servicio ni endpoints.

## Por qué
PHASES.md §8 — Locations (EPIC-004): hitos P4-T1..P4-T7. Sin locations no existe el mapa operativo, la ocupación, ni movements (F5 consume BR-004/005). Es la fase previa obligatoria de Movements.

## Scope
- Backend: módulo `src/locations` (CRUD, capacidad, consultas de distribución), migración `overOccupationLimitPercent` en Location, seed de distribución CargoLocation, tests unit + e2e.
- Frontend: `features/locations` (páginas + LocationCard/CapacityIndicator) y componentes de distribución (DistributionPanel/LocationOccupancyCard) en la UI de cargos.
- NO incluye: movements (F5), mapa SVG (F6), dashboard (F7), settings globales (defaults por tipo quedan como constantes del servicio hasta que exista settings).

## Contexto ya existente (verificado por mapper — no reimplementar)
- Schema F1 completo: `Location`, `CargoLocation`, `Map`, `MapElement`, `Movement`; enums `LocationType` (PLAZOLETA/GALPON/SECTOR/SCANNER/BALANZA/REZAGO/SECUESTRO/OTRO), `LocationStatus` (ACTIVE/INACTIVE/MAINTENANCE), `CargoLocationStatus` (ACTIVE/EXITED), `QuantityUnit` (sin UNLIMITED). `AuditAction.CAPACITY_CHANGE` ✅.
- Seed `prisma/seed.ts` YA crea las 17 ubicaciones (upsert por code): LOC-SEC-01..12 (SECTOR 100 AREA), LOC-PLAZOLETA (12 UNITS), LOC-SCANNER (1 UNITS), LOC-BALANZA (1 UNITS), LOC-REZAGO (20 UNITS), LOC-SECUESTRO (10 UNITS), todas ACTIVE, allowOverOccupation false.
- RBAC: `location.read` (3 roles) y `location.manage` (ADMIN) sembrados. OPERATOR no tiene write de locations (corresponde a ADR-009).
- No existe `src/locations`, `src/movements`, `src/maps` ni módulo settings. `app.module.ts` registra prisma/health/auth/cargo/trucks.
- Convenciones: PermissionsGuard por handler (`@RequirePermissions`), servicios sin autorización, excepciones por módulo extendiendo AppException, envelope `{data}` vía interceptor, paginación `$transaction([findMany,count])`, whitelist de sort, soft delete scope const, mappers puros en `*.response.ts`, `audit.record` fuera del try/catch con `pickFields`.

## Tasks

### F4-T1 — Módulo locations CRUD (P4-T1) + migración BR-036
- [x] Migración: agregar `overOccupationLimitPercent Decimal @default(10) @db.Decimal(5,2)` a `Location` (BR-036 configurable +10%).
- [x] `src/locations/`: DTOs (CreateLocationDto/UpdateLocationDto/LocationQueryDto), excepciones (LOCATION_NOT_FOUND, LOCATION_CODE_DUPLICATE, LOCATION_HAS_CARGO), service con CRUD + auditoría CAPACITY_CHANGE en cambios de capacidad/unidad, controller `@RequirePermissions('location.read'|'location.manage')`, mappers `*.response.ts`.
- [x] Listado `GET /api/v1/locations` con filtros type/status/includeCapacity + paginación; detalle `GET /api/v1/locations/:id`.
- [x] POST/PATCH por ADMIN (location.manage); PATCH a INACTIVE/MAINTENANCE rechazado 422 LOCATION_HAS_CARGO si hay CargoLocation ACTIVE (BR-004 service assert).
- [x] e2e: CRUD, permisos (403 sin manage), duplicados, BR-004.
- ✅ **Commit `bfa9db4` en `feat/p4-locations-crud`** — writer verificado: lint 0/0, format ok, e2e 40/40 (locations), unit 170 (9 files), build 0; pre-commit hooks pasaron. Spot-check parent: e2e 40/40 re-corrido, migración `20260926014108_add_over_occupation_limit` presente, `LocationsModule` registrado.
- ✅ **Merge `592865b` (#7) → main** (squash incluye el fix CI + fixtures propias).

### F4-T2 — CapacityCalculator + consultas de capacidad (P4-T2/T3)
- [x] `CapacityCalculator` (service util): `occupiedCapacity`/`availableCapacity` derivados = Σ CargoLocation ACTIVE en unidad compatible (BR-033/035); sin conversión de unidades (BR-048); `allowOverOccupation` + `overOccupationLimitPercent` (BR-036, ADMIN-only vía permiso, auditoría CAPACITY_CHANGE).
- [x] Unidad efectiva por tipo (BR-041): resolver default (SECTOR/GALPON→AREA, PLAZOLETA/SCANNER/BALANZA→UNITS, resto→configurable; constante hasta settings).
- [x] `GET /api/v1/locations/:id/capacity`: capacity/occupied/available/occupancyPercent/segmentsByUnit (BR-033/040).
- [x] `includeCapacity=true` en listado/detalle.
- [x] Unit tests del calculator (compatible units, incompatibles rechazados, sobreocupación con/sin flag).
- ✅ **Commit `f8ae263` en `feat/p4-locations-capacity`** — writer verificado: lint 0/0, format ok, unit 207 (10 files), e2e locations 50/50, build 0; e2e completa 158/158 sin regresión. Spot-check parent: e2e 50/50 re-corrido.
- 🧠 **Decisión D-55** (writer, alineada a BR-036): `availableCapacity` ahora es la diferencia cruda (negativa en sobre-ocupación) en vez de `Math.max(0, ·)` — la UI no puede confundir "exactamente lleno" con "sobre-ocupado".
- ✅ **Merge `9916c8f` (#8) → main** (retarget de base + cherry-pick de T2 sobre main: la topología original con el fix daba CONFLICTING al retargetear — decisión 20).

### F4-T3 — Seed distribución + consultas distribución (P4-T5/T6)
- [x] Seed CargoLocation (skip-on-exists, patrón seed-cargo): asociar cargos de ejemplo del seed a ubicaciones §5 (p.ej. 029TERRA26 → Sector 3 + Sector 4; 032TERRA26 → Sector 4; 050TERRA26 → Sector 4), quantity + quantityUnit, ACTIVE.
- [x] `GET /api/v1/cargos/:id/locations` (perm cargo.read): DistributionSegmentDto paginado con location nested (facade en cargo module, MODULES.md).
- [x] `GET /api/v1/locations/:id/cargos` (location.read): items con cargoId/code/status/cargoLocationId/quantity/quantityUnit/percentage/enteredAt + openAlerts/permanenceDays.
- [x] e2e: distribución consultable desde ambos lados; segmentos ACTIVE/EXITED.
- ✅ **Commit `463dc92` en `feat/p4-distribution-queries`** — writer verificado: lint 0/0, format ok, unit 254 (12 files), e2e distribution 33/33 + locations 50/50 + full 191/191, seed idempotente (2ª vuelta `0 created`), build 0. Spot-check parent: e2e full 191/191 + unit 254/254 re-corridos.
- 🧠 **D-56** (writer): `onlyActive` default **true** en `GET /locations/:id/cargos` — API.md §6.4 lo documenta así ("Cargos presentes en la ubicación"); `?onlyActive=false` amplía al historial EXITED.
- 🧠 **D-57** (writer): `openAlerts` hardcodeado 0 (slot para F7 — alerts no existe aún), comentado en DTO.
- 🧠 **D-58** (writer): refactor DRY — helpers comunes `common/pagination/pagination-meta.ts` + `common/query/to-boolean-param.ts`; trucks/locations/cargo repointados (el commit toca archivos fuera de los endpoints nuevos; suite completa verde).
- Seed mapping (9 segmentos ACTIVE, todos AREA): CARGO-001→SEC-03(6)+SEC-04(5); CARGO-002→SEC-04(8); CARGO-003→SEC-05(9); CARGO-004→SEC-04(2)+SEC-05(2)+SEC-06(1); CARGO-005→SEC-06(4)+SEC-03(3). Σ por cargo < totalQuantity (BR-034).
- ✅ **Merge `dcf21b2` (#9) → main** (mismo retarget + cherry-pick; árbol idéntico al verificado; 1 flake local no reproducible — CI verde).

### Fix CI e2e (2026-09-27) — seed en el job e2e + fixtures propias de ADR-011
- 🐞 **Root cause**: el job `End-to-end (postgres)` de `.github/workflows/ci.yml` corría `prisma migrate deploy` pero NUNCA `prisma db seed`. Los specs e2e asumen el catálogo canónico sembrado (explícito en los comentarios: `distribution.e2e-spec.ts` aserta los segmentos demo CARGO-001→LOC-SEC-03/04; `locations.e2e-spec.ts` usaba una ubicación sembrada vía `anySeededLocationId`). En CI la DB arranca vacía → `findFirstOrThrow` P2025 → 2 fails en #7 (y el test de seed de distribución fallaría en #9 también). Local pasaba porque la DB dev tiene el seed — exactamente lo que escondió el bug.
- ✅ **Fix 1 — `ci(e2e): seed the database before the e2e suite`** (`cc63e42` en #7): agrega `npx prisma db seed` tras `migrate deploy` en el job e2e. Idempotente; replica el entorno dev canónico.
- ✅ **Fix 2 — `test(locations): own ADR-011 fixtures instead of borrowed seeded rows`** (`7c418c1` en #7): reemplaza `anySeededLocationId()` por `createScopedLocation(suffix)` que crea la ubicación vía el POST real (código con prefijo `CODE`, el afterAll la limpia). El test de lista ahora aserta presencia ANTES del soft-delete (no-vacuo). Hace el spec hermético — robusto aunque corras e2e sin seed.
- **Verificación CI-exacta**: simulación local con DB scratch fresca (`cargoops_ci_verify`): `migrate deploy` + `db seed` + `npm run test:e2e` → **191/191**. Gates locales en la punta de la cadena: lint 0 (80 files), format ok, unit 254/254, build 0.
- **Propagación**: merges locales `913efa0` (#8) y `b906fde` (#9) — sin conflictos (los cambios no solapan con F4-T2/T3). Las 3 ramas pusheadas; CI re-corrido por GH.
- PRs de la cadena: #7 → #8 → #9 (backend), frontend #4.

### F4-T4 — UI locations + distribución (P4-T4/T7)
- [x] `features/locations`: lista de ubicaciones (LocationCard: nombre/código/tipo/estado) + detalle con CapacityIndicator (capacidad/ocupada/disponible/%).
- [x] DistributionPanel en detalle de cargo: cargas de una ubicación / ubicaciones de una carga (consumir consultas F4-T3).
- [x] LocationOccupancyCard (capacity/occupied/available/allowOverOccupation).
- [x] Guard de rutas + tests unit de páginas/componentes.
- ✅ **Commit `d103a73` en `feat/p4-locations-ui`** — writer verificado: lint 0/0 (47 files), format ok, ng test 49 (10 files), build exit 0 con lazy chunks de locations. Spot-check parent: ng test 49/49 + build re-corridos; nav con "Ubicaciones" → /locations.
- 🧠 **D-59** (writer): fix WAI-ARIA — `aria-valuenow` clamped al max (100) porque la sobre-ocupación >100% es estado normal (BR-036); `aria-valuetext` mantiene el valor real. Test de regresión.
- 🧠 **D-60** (writer): fix bug preexistente F3 — nav link apuntaba a `/cargas` pero la ruta es `cargos` (caía al not-found).
- ✅ **Merge `f0e4166` (frontend #4) → main**.

## Cierre FASE 4 (2026-09-27)
- Cadena mergeada en orden decisión 20: `592865b` (#7) → `9916c8f` (#8) → `dcf21b2` (#9) backend; `f0e4166` (#4) frontend. Ramas de la cadena eliminadas (local + remoto).
- Verificación post-merge: backend main unit 254/254 + e2e full 191/191; frontend main ng test 49/49 + build OK.
- 🧠 **D-61**: flujo retarget con squash — si la rama hija tiene topología de merges del base, al retargetear da CONFLICTING; reconstruir = `checkout -B <rama> origin/main` + `cherry-pick <commit único>` (contenido idéntico, verificado por `git diff` vacío).
- Pendientes heredados F3 (sin tocar): bug login decisión 44, gaps documentales 32/35/36/39/40/41, patente en detalle 45, breadcrumb 46.

## Decisiones
- **D-48**: `overOccupationLimitPercent` se agrega como columna en Location con default 10 (BR-036/OQ-043: configurable por ubicación, no constante). Migración propia.
- **D-49**: Unidad efectiva por tipo (BR-041) resuelta en el servicio (capacidad/unidad default por LocationType); el schema guarda el valor efectivo (capacityUnit es required). Defaults como constantes hasta que exista settings.
- **D-50**: `occupiedArea` (DTOs.md §4.12) no tiene columna en schema → NO se implementa; gap documental para limpiar DTOs.md (mismo patrón decisión 32/35). `capacityPercent`/`occupancyPercent`/`segmentsByUnit` se computan en mappers.
- **D-51**: Límites de storage mandan sobre DTOs.md (patrón decisión 40): `code` MaxLength(50) (schema VarChar 50, DTOs.md dice 30), `description` MaxLength(255) (DTOs.md dice 500). Gap documental.
- **D-52**: CargoLocation único ACTIVE por (cargoId, locationId) ya está en schema (uq_cargo_locations_active) — el service debe protegerlo (P2002 → 409).
- **D-53**: BR-004 assert (rechazar mover a inactiva) se implementa como método de dominio del service locations (`assertCanReceive`), testeado acá, consumido por movements en F5 (P4-T3 según PHASES.md).
- **D-54**: Delivery chained PRs stacked-to-main (cacheado); >400 líneas aceptadas por PR (heurística, no bloqueante). CI verde requerido antes de cada merge (flujo decisión 20).

## Restricciones
- No modificar orden de fases (§18 PHASES.md). Movements/CargoLocation writes (POST/PATCH/DELETE /cargos/:id/locations) son FASE 5 — NO implementar ahora.
- No inventar reglas de negocio: aplicar literalmente BR-004/005/033/034/035/036/040/041/048.
- Conventional Commits, sin atribución IA; artifacts técnicos en inglés; copy UI es-AR neutral.

## Aceptación
- Backend: lint 0/0, format ok, build 0, unit y e2e verdes (nuevos tests de locations + regresión F3); CI verde por PR.
- Frontend: lint 0/0, format ok, build 0, ng test verdes; páginas y componentes con estados loading/empty.
- Seed: 17 ubicaciones + distribución CargoLocation de ejemplo (idempotente, skip-on-exists).
- Permisos: lectura 3 roles, escritura ADMIN-only (403 sin manage).

## Próximo paso
1. F4-T1: branch `feat/p4-locations-crud` desde main, PR #7.
2. Al cerrar FASE 4: mirrors Engram + session summary; FASE 5 (Movements) a decisión del usuario.