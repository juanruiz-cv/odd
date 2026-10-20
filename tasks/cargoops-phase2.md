# FASE 2 — Authentication + RBAC (EPIC-002)

## Objetivo
Implementar PHASE 2 según PHASES.md §6: auth JWT (access+refresh), roles/permisos (Viewer, Operator, Admin), guards backend, auth frontend, auditoría.

## Alcance autorizado
- Backend: auth, users, roles, permissions, audit (módulos)
- Frontend: login, guards, interceptors, services/auth, state
- Git flow: feature/* desde develop, merges --no-ff a develop, sin push/remote

## Tareas (P2-H1–P2-H5)
- P2-T1 Auth JWT (access+refresh, logout, rate limit, hashing)
- P2-T2 Seed roles/permisos (Viewer/Operator/Admin, Permission codes §8)
- P2-T3 Guards backend (JwtAuthGuard, PermissionsGuard) en rutas prueba
- P2-T4 Frontend auth (login, auth guard UX, interceptor refresh, logout)
- P2-T5 Auditoría LOGIN/LOGOUT + helmet/CORS/sanitización

## Criterios
- Login/refresh/logout con tests; refresh rota tokens; rate limit
- Matriz §8 1:1; BR-010/011/012
- 401/403 correctos (BR-009)
- UX guards + interceptor refresh
- AuditLog LOGIN/LOGOUT; headers seguridad
- Sin modificar MASTER-SPEC/OPEN-QUESTIONS
- No AI attribution en commits
- Verificar todo antes de marcar done

## Estado
[x] P2-T1
[x] P2-T2
[x] P2-T3
[x] P2-T4
[x] P2-T5
[x] P2-FIX guards globales deny-by-default (2026-10-15)

## Verificación de cierre (orquestador, 2026-10-15)
- Backend `develop`: merge FASE 2 `c5f9a15` (commits 5acda6f, 5096480, b66d468, e553a7e, 40a5d1f).
- Frontend `develop`: merge `54e3d4a` (commits ec70f3f, c8736f1, d6c8b71).
- Gates re-ejecutados: backend `npm test` 64/64 (14 files), `npm run test:e2e` 20/20 (3 files), build 0, lint 0, format 0; frontend `ng build` 0, `ng test` 19/19 (7 files).
- DB: 25 permissions, 3 roles (ADMIN/OPERATOR/VIEWER), 48 role_permissions (25/15/8). Nombres canónicos («Admin / Supervisor» §MASTER-SPEC L257).

## P2-FIX — Guards globales deny-by-default (desviación del spec, RESUELTA)
- **Desviación detectada por el orquestador**: `JwtAuthGuard`/`PermissionsGuard` estaban aplicados por ruta con `@UseGuards` (fail-open); `AUTHORIZATION.md` §5.4 exige guards **globales** con **deny-by-default** (todo handler sin permiso declarado → 403, con `@Public()` para rutas públicas).
- **Fix**: `feature/p2-global-guards` → commit `03b08c4` → merge `--no-ff` `61b622d` en `develop`.
- Registrados como `APP_GUARD` en `src/app.module.ts` en orden Throttler → Jwt → Permissions.
- Rutas públicas confirmadas contra `API.md` §4: `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout` (leen cookie `refresh_token`), `GET /health`.
- Rutas protegidas: `GET /users` → `@RequirePermissions('users.manage')`.
- Gates re-ejecutados tras el fix: lint 0 · format:check 0 · build 0 · test 64/64 · e2e 20/20 (incluye `test/guards.e2e-spec.ts`, 5 tests).
