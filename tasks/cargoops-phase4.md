# FASE 4 — Locations (EPIC-005)

## Objetivo
Implementar PHASE 4 según `PHASES.md` §8: módulo `locations` (CRUD), capacidad derivada
(`occupiedCapacity` = suma de `CargoLocation` activos en unidad compatible, BR-033/035),
consultas de distribución M:N (BR-040) y UI de distribución/ocupación.

## Alcance autorizado
- Backend: `src/modules/locations` (nuevo), seed de distribución M:N de ejemplo.
- Frontend: `src/app/features/locations`, `DistributionPanel`, `LocationOccupancyCard`.
- Git flow: `feature/p4-*` desde `develop`, merges `--no-ff` a `develop`. Sin push/remote. Sin atribución AI.

## Estado de partida (verificado por el orquestador)
- **Prisma YA tiene los modelos** (FASE 1 R-03): `CargoLocation` (L302), `Location` (L334), `MapElement` (L384).
  Por lo tanto **no hay que crear entidades ni migración**: P4-T1 y P4-T2 se reducen a módulo + lógica.
  - `Location`: `code` único (`uq_locations_code`), `type`, `status`, `capacity`, `capacityUnit`,
    `occupiedCapacity`/`availableCapacity` (nullable), `allowOverOccupation`, `deletedAt` (soft delete).
  - `CargoLocation`: `cargoId`, `locationId`, `quantity`, `quantityUnit`, `percentage`, `enteredAt`,
    `exitedAt`, `status` (`CargoLocationStatus`, default `ACTIVE`), `notes`.
    Índice único parcial `uq_cargo_locations_active` = `(cargo_id, location_id) WHERE status='ACTIVE'`
    ya existe como SQL raw.
- Seed ya crea **17 locations** (idempotente, verificado).
- Permisos reales del catálogo: **`location.read`**, **`location.manage`** (no hay `location.create`/`update`/`delete`).
- Backend `develop` = `69a939d`; Frontend `develop` = `de4a39f`. Ambos limpios.
- Guards globales deny-by-default: toda ruta nueva necesita `@RequirePermissions(...)` o `@Public()`.

## Tareas
- [x] P4-T1 Módulo `locations`: service/repository Prisma + DTOs + controller (CRUD con `location.manage`, lectura con `location.read`). Soft delete. (Entidad/migración YA existen.)
- [x] P4-T2 Capacidad derivada: `occupiedCapacity` = Σ `CargoLocation` ACTIVE por unidad compatible (BR-033/035); `availableCapacity`; `allowOverOccupation` + validación de sobre-ocupación (BR-041/OQ-043).
- [x] P4-T3 Validación de estado de ubicación (BR-013/ADR-011 + reglas de §4.4) reutilizable por PHASE 5.
- [x] P4-T4 Seed de distribución de ejemplo + verificar ocupación derivada sobre los 17 locations.
- [x] P4-T5 `CargoLocation`: alta/baja de distribución (mover carga a/desde ubicación) respetando `uq_cargo_locations_active` y `percentage` (BR-032/OQ-045).
- [x] P4-T6 Consultas: ubicaciones de una carga, cargas de una ubicación, capacidad de una ubicación, ocupación agregada (BR-033/035/040).
- [x] P4-T7 UI: `DistributionPanel` + `LocationOccupancyCard` (capacidad/ocupada/disponible/%), estados loading/empty.

## Criterios de aceptación (PHASES.md §8)
- Módulo locations con CRUD; Viewer/Operator consultan, ADMIN (con `location.manage`) modifica.
- Ocupación **derivada**, nunca contador manual; sobre-ocupación rechazada salvo `allowOverOccupation`.
- Consultas exponen ubicaciones, cantidad, porcentaje, fechas de ingreso/salida e historial.
- Capacidad/ocupada/disponible por unidad compatible.
- UI refleja la distribución M:N con estados loading/empty.

## Alcance de esta corrida
- **4A (backend, P4-T1..P4-T6)**: módulo locations + capacidad derivada + consultas + tests.
- [x] P4-T7 (FASE 4B — frontend) — completado en corrida posterior (ver "Evidencia (FASE 4B — frontend)").

## Estado
- [x] P4-T1
- [x] P4-T2
- [x] P4-T3
- [x] P4-T4
- [x] P4-T5
- [x] P4-T6
- [x] P4-T7

## Evidencia (FASE 4A — backend)

Rama: `feature/p4-locations-backend` → merge `--no-ff` a `develop`.

### Commits (work units, Conventional Commits, sin atribución AI)
| Hash | Mensaje |
| --- | --- |
| `a8f3ac5` | feat(locations): add location registry with derived occupancy |
| `f292519` | feat(cargo): add cargo location distribution endpoints |
| `e073a43` | test(locations): cover location and distribution HTTP flows |
| `fd53953` | feat(seed): seed P4 distribution segments with derived occupancy |
| `b9f8517` | Merge branch 'feature/p4-locations-backend' into develop |

### Gates (observados)
| Gate | Resultado |
| --- | --- |
| `npx prisma validate` | `The schema at prisma\schema.prisma is valid 🚀` |
| `npm run db:seed` (x2) | Idéntico en ambas corridas: `Seed complete: 25 permissions, 3 roles, 17 locations, 15 cargos, 4 active distribution segments` |
| `npm run lint` | 0 errores, 0 warnings |
| `npm run format:check` | `All matched files use Prettier code style!` |
| `npm run build` | `nest build` limpio |
| `npm test` | 17 archivos / 103 tests passed |
| `npm run test:e2e` | 5 archivos / 54 tests passed |

### Ocupación derivada (verificada en DB tras seed)
| Location | capacity | occupied_capacity | available_capacity |
| --- | --- | --- | --- |
| `sector-04` | 100.00 AREA | 80.00 | 20.00 |
| `sector-03` | 0.00 AREA (unbounded) | 20.00 | (null) |
| resto (15) | — | 0.00 | (null) |

### Alcance no cubierto en esta corrida
- P4-T7 (UI `DistributionPanel` + `LocationOccupancyCard`) — FASE 4B, pendiente.
- Backend: solo `POST`/`DELETE` de `CargoLocation` (el PATCH y la integración con `Observation` de movimientos, junto con BR-042, quedan para PHASE 5).

## Evidencia (FASE 4B — frontend)

Rama: `feature/p4-locations-ui` desde `develop` @ `de4a39f` → merge `--no-ff` a `develop` @ `06184d1`.

### Commits (work units, Conventional Commits, sin atribución AI)
| Hash | Mensaje |
| --- | --- |
| `5d53568` | feat(locations): add location models, formatting utils and API client |
| `34530cd` | feat(locations): add locations list store |
| `4397da7` | feat(locations): add location occupancy card |
| `3b8dd98` | feat(locations): add distribution panel |
| `2ff563e` | feat(locations): wire cargo distribution into the cargo detail page |
| `7c0458b` | feat(locations): add locations list and detail pages |
| `06184d1` | Merge branch 'feature/p4-locations-ui' into develop |

### Componentes y piezas creadas
| Pieza | Ruta |
| --- | --- |
| Modelos + permisos | `src/app/core/models/location.model.ts` |
| `LocationService` (lectura + escritura) | `src/app/features/locations/services/location.service.ts` |
| `LocationsStore` | `src/app/features/locations/state/locations.store.ts` |
| `LocationOccupancyCardComponent` | `src/app/features/locations/components/location-occupancy-card/` |
| `DistributionPanelComponent` | `src/app/features/locations/components/distribution-panel/` |
| `CargoDistributionComponent` | `src/app/features/locations/components/cargo-distribution/` |
| Páginas lista/detalle | `src/app/pages/locations/` |
| Rutas `/locations` y `/locations/:id` (tras `authGuard`) + link en dashboard | `src/app/app.routes.ts`, `src/app/pages/dashboard/dashboard.component.html` |

### Gates (observados, sobre `develop` tras el merge)
| Gate | Resultado |
| --- | --- |
| `npx prettier --check .` | `All matched files use Prettier code style!` |
| `npx ng build` | `Application bundle generation complete` (sin errores) |
| `npx ng test --watch=false` | 18 archivos / 91 tests passed (baseline 14/60) |

### Notas
- Verificación contra backend en vivo: **NO** (no se levantó backend; los tests usan `HttpTestingController` sobre los shapes DTO de API.md/DTOs.md).
- Permisos: las acciones de escritura (`Distribuir`/`Egresar`, `location.manage`) se ocultan salvo que la sesión tenga `cargo.move` / `location.manage`; el backend sigue siendo la autoridad (BR-009).
- Android/iOS no aplica; solo web.
- Se mantienen los workarounds de Smart App Control (vitest/vite 7, alias `@rollup/wasm-node`, plugin `test/ts-decorator-metadata.plugin.ts`).
