# CargoOps — Cierre health-probes: QA, documentación y merge (feature: cargoops-health-probes-cierre)
> ## DOCUMENTO DE LA LINEA TRACK - NO ES BASELINE DEL REBUILD
>
> **Marcado el 2026-10-20 por el orquestador.** Este documento pertenece a la **linea de
> trabajo Track**, anterior al rebuild desde cero. **No describe el repositorio actual** de
> este proyecto.
>
> **Sobre las referencias que cita:** menciona PRs (`cargoops-backend` #19/#20/#23,
> `cargoops-docs` #5) y commits (`efd0286`, `4640d0b`, `173e1c9`, `7cfa607`...) que **no son
> verificables en este clon**: los tres repos de codigo no tienen remoto, y esos objetos no
> existen en el historial local. La linea Track trabajo en un clon con remoto; ademas su
> propio handoff advierte que en ella **se inventaron SHAs y PRs** (ver `handoff/2026-10-01.md`,
> "Aviso de integridad"). No tomar sus numeros como hechos sin re-verificarlos.
>
> **Valor que si conserva:** decisiones de producto, hallazgos de seguridad y conflictos
> documentales que explican por que ciertas cosas estan como estan (p. ej. D-99: el access
> token sobrevive al logout hasta 15 min; `/auth/me` borrado por no tener consumidores).
>
> **Uso permitido:** consulta historica unicamente. **Nunca** tratar sus checkboxes como
> cumplimiento del rebuild actual.


## Objetivo
Cerrar el fix de health probes (`4640d0b`) con gate de QA completo, documentación al día y merge a `main`. Auditar de paso los cambios sin commitear que existían en el árbol y evitar mergear trabajo que rompa la coherencia código↔docs.

## Problema
Tres hallazgos de la sesión refutaron la premisa inicial:
1. `localhost:4200` servía un bundle horneado viejo, no el código → el sidebar "roto" era un falso positivo.
2. `npm test` nunca corre los e2e (`*.e2e-spec.ts` no matchea `include: ['**/*.spec.ts']`) → "tests verdes" nunca probaron el arranque de la app.
3. Tres cambios sin commitear de sesiones previas, dos de los cuales introducen contradicciones con los docs canónicos.

## Alcance autorizado
- **Backend**: commit `4640d0b` (ya hecho), documentación (`README.md`, `CONTRIBUTING.md` nuevo), push, PR, CI, merge a `main`.
- **Docs**: readback de coherencia `API.md` §9.6 ↔ implementación. **Sin** editar.
- **NO**: revert de decisiones de negocio, cambios de schema, implementación del mapa, refactor de `CargoService`.

## Checklist

- [x] **T01** Auditar los 3 cambios sin commitear (backend ×2, docs ×1) — inline (orquestador)
- [x] **T02** Revertir el WIP de `src/cargo/*` (`name` en `CreateCargoDto`) — inline; decisión del usuario
- [x] **T03** Commitear el fix de health (`4640d0b`) con work-unit commit — inline
- [x] **T04** Gate de QA local: `lint` · `format:check` · `build` · `unit` · `e2e` — inline
- [x] **T05** `README.md` backend actualizado + `CONTRIBUTING.md` nuevo — delegated (writer, 2 archivos no triviales)
- [x] **T06** Readback coherencia `cargoops-docs` (`API.md` §9.6, `MASTER-SPEC` §4.1) — inline — **encontró drift real, Contrary a la suposición inicial**
- [x] **T07** Push de `fix/health-probes-di-boot` + apertura de PR (#19) — inline
- [x] **T08** QA de CI: workflow verde en el PR antes de mergear — delegated (fresh worker, `gh pr checks --watch`)
- [x] **T09** Merge a `main` + verificación post-merge — inline
- [x] **T10** Espejo Engram `odd/cargoops-health-probes-cierre/tasks` + cierre — inline

## T06 / T11 — `API.md` §9.6 SÍ tenía drift (corrección de una suposición mía)

Dije al inicio que §9.6 ya cuadraba con el código. **Era falso.** El ejemplo de readiness 200 seguía mostrando `"checks": { "database": "up", "redis": "up" }` y sin `timestamp`. Lo detectó el writer delegado al redactar el README; yo lo verifiqué contra `src/health/health.controller.ts` antes de actuar.

Corregido en `cargoops-docs` con el criterio ya establecido: la implementación enviada es la autoridad, y añadir una sonda de Redis exigiría primero la dependencia real, que no existe.

- Commit `1804f79` en `docs/resolve-spec-code-drift`, **PR #2**, squash merge `91fd6b6`.
- `cargoops-docs` **no tiene CI**: el gate fue verificar que el ejemplo parsea, no una corrida automática.
- **Autocorrección durante el edit**: metí una llave `}` de más en el ejemplo. Se detectó al parsear el JSON y comparar contra la línea de liveness, que ya tenía la forma correcta. Llaves balanceadas 3/3, `ConvertFrom-Json` OK.

## T08 / T09 — resultado del QA y del merge

| Repo | PR | Commits | Merge | Gate |
| --- | --- | --- | --- | --- |
| `cargoops-backend` | #19 | `4640d0b` + `4d11a38` | squash `efd0286` | CI verde: `checks` 46s, `e2e` 1m37s |
| `cargoops-docs` | #2 | `1804f79` | squash `91fd6b6` | sin CI; verificación explícita del JSON |

El job `e2e` de CI aprovisiona un `postgres:16-alpine` real y corre `prisma migrate deploy` + `db seed` antes de los tests: **252 aserciones contra base real**, no contra mocks. El worker verificó que las suites no pasan en vacío (15 archivos / 431 unit, 12 archivos / 252 e2e).

Estilo de merge: **squash**, que es el patrón dominante del repo (#9 a #17, `main` lineal); #18 fue la excepción con merge commit.

### Gotcha de refs encontrado durante el post-merge

`main` local en `cargoops-docs` estaba **10 commits atrasada**. El `git checkout main` no llegó a aplicarse y el `git pull --ff-only` corrió sobre `docs/resolve-spec-code-drift`, que **divergió** de `main` porque el squash reescribió el commit — de ahí un `fatal: Not possible to fast-forward` que parecía un conflicto de contenido pero no lo era. El diff de 48 líneas en `DTOs.md` queparisonaba contra un `main` obsoleto. Resuelto con `git stash push` → `checkout main` → `pull --ff-only` → `stash pop`, con el stash verificado antes de mover refs. El WIP de `DTOs.md` quedó restaurado y sin commitear, como se decidió.

## Decisiones tomadas por el usuario
- **Redis fuera del health check**: Redis no es dependencia del backend (sin `redis`/`ioredis`, sin `REDIS_URL`); la revocación de refresh vive en la tabla `refresh_tokens`.
- **Alinear con `API.md` §9.6**: `/health` = liveness puro (sin dependencias), `/health/ready` = readiness con 503 `SERVICE_UNAVAILABLE`.
- **Revertir el WIP de `src/cargo/*`**: quitaba `name` de `CreateCargoDto`, contradiciendo `MASTER-SPEC` §4.1 y `DTOs.md` §79/§88.

## Verificación (T04)
| Gate | Comando | Resultado |
| --- | --- | --- |
| Lint | `npm run lint` | 0 errores; ~18 warnings preexistentes, ninguno en `src/health` |
| Formato | `npm run format:check` | ✅ "All matched files use Prettier code style!" |
| Build | `npm run build` | ✅ Prisma client 7.10.0 generado + `nest build` |
| Unit | `npm test` | ✅ **431/431** en 15 archivos |
| E2E | `npm run test:e2e` | ✅ **252/252** en 12 archivos |

## Hallazgos que quedan ABIERTOS (no resueltos aquí)

1. ~~**`docs/backend/DTOs.md` sin commitear**~~ — **RESUELTO** (PR #3 en `cargoops-docs`, squash `43b344c`, 2026-10-02). El WIP que borraba la fila `description` se descartó: `MASTER-SPEC` §4.1 la exige y el DTO la implementa. Lo que estaba mal era el número del tope, no la existencia de la fila. Se alinearon **dos** filas de §4.6 contra el almacenamiento canónico: `code` 30 → 50 (columna `varchar(50)`, decisión D-51) y `description` 500 → 255 (columna `varchar(255)`). Los dos topes fallaban en sentidos **opuestos**: 500 pasaba la validación documentada y reventaba después con `value too long` de Postgres (un 500 opaco en vez de un 400 con mensaje); 30 rechazaba códigos que el modelo canónico sí puede guardar. Se agregó la nota *storage-wins* bajo la tabla para que nadie "corrija" los números de vuelta. Al terminar, `description` quedó consistente también con `CreateRoleDto`, que ya usaba 255. Gate manual por no haber CI: doc vs código extraído campo por campo (120=120, 50=50, 255=255, MATCH en los tres), 13 filas con conteo de pipes uniforme, encoding sin U+FFFD ni BOM.
2. **`GET /health/ready` sin timeout** — **RESUELTO** (backend PR #21, squash `c95ad0f`; documentado en `API.md` §9.6 por docs PR #4, squash `8800986`, 2026-10-02). Presupuesto de **2 s**, elegido por el dueño. El fallo era real: una base que deja de responder *sin cerrar el socket* deja la promise del driver pendiente para siempre, así que la sonda se colgaba en vez de responder 503 — y una sonda colgada es peor que una que falla, porque el caller queda esperando en vez de recibir la orden de dejar de mandarle tráfico. Una conexión rechazada sí rechaza sola, que es exactamente por qué ningún test existente lo detectaba. Implementado con `Promise.race` entre el `SELECT 1` y un timer, limpiado en `finally`: sin esa limpieza el timer pendiente mantiene vivo el event loop durante el presupuesto completo después de cada probe exitoso, lo que **cuelga** al worker de vitest en vez de fallarlo. Dos tests fijan `vi.getTimerCount()` en 0. El presupuesto acota la respuesta, no la statement (cancelarla exigiría una transacción interactiva y no aportaría nada). El log distingue `timed out` de `failed` porque son incidentes distintos. Gate: 437/437 unit (431 + 6), 252/252 e2e sin movimiento, typecheck 0, lint 0 errores, format limpio. **Mutation check**: quitar el `race` falla 4 de los 6 tests nuevos y revienta el suite de 305 ms a 20,3 s — los tests no son vacíos.
3. **`GET /health/ready` público y sin throttling** (`AUTHORIZATION.md` §5.4): usable como oráculo de disponibilidad de la DB.
4. **Invariante "un solo `PrismaService`" sin guarda**: `overrideProvider` de Nest itera todos los módulos, así que ningún test puede detectarlo. Queda un comentario.
5. **25 errores preexistentes de `tsc --noEmit`**, verificados por código: `TS2307: 11` (falta `supertest/types` en `test/*.e2e-spec.ts`), `TS2345: 9`, `TS2339: 2`, `TS2314: 2`, `TS2322: 1`. **Ninguno en `src/health`.**
6. **`cargoops-frontend/README.md` es un stub de 127 bytes** y el repo frontend no tiene CONTRIBUTING.

## Evidencia
- Merge `cargoops-backend`: PR **#19**, squash `efd0286` sobre `origin/main` (`9244519`). Commits de rama `4640d0b` (fix) + `4d11a38` (docs).
- Merge `cargoops-docs`: PR **#2**, squash `91fd6b6`. Commit de rama `1804f79`.
- Verificado sobre `main` después del merge, no solo con el estado de GitHub:
  - `cargoops-backend/main`: `README.md`, `CONTRIBUTING.md` y `src/health/health.controller.spec.ts` presentes; `health.controller.ts` sin ningún `@Inject(...)` real (solo las menciones en comentarios); working tree limpio.
  - `cargoops-docs/main`: §9.6 con `checks.database` + `timestamp`, JSON parsea, `redis` ausente.
- Contratos verificados en vivo contra `cargoops-api-dev`:
  - `GET /api/v1/health` → 200 `{"data":{"status":"ok","uptime":N,"timestamp":"ISO"}}`
  - `GET /api/v1/health/ready` → 200 `{"data":{"status":"ok","checks":{"database":"up"},"timestamp":"ISO"}}`
  - caso DB caída → 503 `{"error":{"code":"SERVICE_UNAVAILABLE","message":"Dependencias no disponibles","requestId":"…","details":{"database":"down"}}}`
- Orden de implementación registrado en Engram: `bugfix/health-module-di` (134), `api/health-probes` (137), `discovery/vitest-excludes-e2e` (139), `discovery/frontend-served-bundle-is-baked` (138, proyecto `cargoops-frontend`).
