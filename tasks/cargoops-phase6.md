# FASE 6 — Operational Map (EPIC-007)

## Objetivo
Implementar PHASE 6 según `PHASES.md` §10: datos del mapa (BR-020), motor SVG de solo lectura
(ADR-006), interacciones, accesibilidad WCAG 2.2 AA core y performance con ~50 ubicaciones.

## Estado de partida (verificado por el orquestador)
- `Map` y `MapElement` **ya existen** en `prisma/schema.prisma` (L363 y L385) y en la base
  (migración `init`, sin drift). No hace falta migración.
- **No existe** `src/modules/maps` ni la feature `features/map` en el frontend.
- `map.read` / `map.edit` ya están en el catálogo RBAC (`rbac.catalog.ts:44-45`) y asignados a
  VIEWER/OPERATOR/ADMIN. **No se tocan permisos.**
- El seed **no crea ningún `Map` ni `MapElement`** → 6A debe sembrar el plano y sus 17 elementos.
- `Map.createdById` es NOT NULL con FK RESTRICT a `users`: se usa el usuario `seed-system`.
- `GET /maps` no tiene DTO de query; se replica el patrón de `LocationListQueryDto`
  (page/limit + meta con `totalItems`/`totalPages`/`hasNext`).
- `PATCH /maps/:id` **no se implementa**: OQ-015 lo excluye de v1 (`DTOs.md:215`, `API.md:410`).

## Slices
- **6A (backend, P6-T1)**: módulo `maps` de lectura (`GET /maps`, `GET /maps/:id`), proyección
  numérica de `Decimal`, seed del plano + 17 elementos, tests unit + e2e. **COMPLETADO Y MERGEADO.**
- **6B (frontend, P6-T2)**: motor SVG de solo lectura (`OperationalMap`, `MapLocation`, `MapLegend`,
  `MapToolbar`), servicio/modelos y ruta. **COMPLETADO Y MERGEADO.**
- **6C (frontend, P6-T3)**: viewport (zoom 0.4x–3.0x + reset + pan), hover/focus, selección
  bidireccional, `LocationCard` en panel lateral y el label de ocupación obligatorio.
  **COMPLETADO Y MERGEADO.**
- **6D (frontend, P6-T4/T5)**: accesibilidad (navegación por teclado completa, listado alternativo
  equivalente, `prefers-reduced-motion`) y performance (capas, memoización, lazy render).
  **COMPLETADO Y MERGEADO.**

## Plan de 6C (decidido por el orquestador antes de escribir)
Contrato de `COMPONENTS.md` §6.7/§6.8/§6.10 + `MAP-UX.md` §4.1/§4.3/§6.
1. **Viewport en `MapStore`**: signals `zoom` (1), `panX`/`panY` (0); `ZOOM_MIN = 0.4`,
   `ZOOM_MAX = 3`, `ZOOM_STEP = 0.2` (rango con ratificación pendiente en DP-MX-01, que es un
   residual local sin decisión de negocio). API `zoomIn()`, `zoomOut()`, `resetView()`, `setZoom()`,
   `panBy()` con clamping. Se hace en el store porque `MapToolbar` y `OperationalMap` son hermanos.
2. **`MapToolbar` suma sus outputs del contrato**: `zoomIn()`, `zoomOut()`, `resetView()`,
   `toggleLegend()`, `toggleAccessibleList()` (los dos últimos ya existen como señales propias),
   e inputs `zoomLevel`/`canZoomIn`/`canZoomOut`. Sigue sin fetchear.
3. **Zoom**: botones, rueda del mouse con `Ctrl`, `+`/`-` del teclado, doble click, reset con `0`.
   `preventDefault()` en la rueda para no scrollear la página.
4. **Pan**: arrastre con mouse (pointer events + `setPointerCapture`), flechas del teclado.
5. **Transform**: `<g class="operational-map__viewport" [attr.transform]="viewportTransform()">`
   envolviendo los `MapLocation`; el título del plano queda **fuera** del viewport.
6. **Hover/focus/selección**: `MapLocation` suma `isSelected`/`isHovered` y emite
   `hoverChange`/`selectedChange`. El grupo se vuelve focable (`tabindex="0"`, `focus()`/`blur()`
   para que "focus = hover" sea real) y Enter/Space selecciona. Sin movimiento de foco entre nodos.
7. **Resaltado cruzado**: la lista del panel resalta la ubicación bajo hover/focus del mapa y
   hace scroll suave; click en la fila selecciona (mapa ⇄ panel).
8. **`LocationCard`**: nombre, código, tipo, estado y `CapacityIndicator` con la **unidad efectiva**
   (BR-041), todo desde la `Location` que ya tiene el store. El panel es la alternativa accesible:
   operable sin el SVG y con la misma información.
9. **`CapacityIndicator`** (COMPONENTS §6.12): `role="meter"` + `aria-valuenow/min/max` +
   `aria-valuetext` con unidad ("80 de 100 m³"), label textual además de la barra, estados
   unlimited (`capacity <= 0` → "Ilimitado" sin barra, consistente con D11) y sin datos.
10. **Requirement entrante de 6B**: el label de ocupación `"35/100 m³ · 35 %"` que exige
    `MAP-VISUAL-GUIDELINES.md:72` pasa a renderizarse en el **panel**, donde puede verse completo
    sin saturar el canvas. En el canvas el label sigue siendo el nombre (densidad, §5.7).
11. **Sin animación** (D-P6-3) y **sin drag & drop de cargas** (D-P6-1).

## Plan de 6D (decidido por el orquestador antes de escribir)
Último slice de la FASE 6. Cubre **P6-T4 (accesibilidad)** y **P6-T5 (performance)**, según
`PHASES.md:122-123`, `ACCESSIBILITY.md` §5.2/§5.3/§5.6/§5.7/§5.10 y `PERFORMANCE.md` §5.3/§5.4.

### Alcance real (lo que YA está hecho y NO se repite)
6B/6C ya entregaron buena parte de P6-T4: `role="group"` + `aria-label` en el SVG, `role="img"` +
label por ubicación, foco real (`focus`/`blur` = hover), Enter/Space, anillo `#D97706`, navegación
por teclado del viewport (`+`/`-`/`0`/flechas), listado alternativo operable sin el SVG con
`aria-current`, y sin estado transmitido solo por color. **6D es lo que falta, no un rewrite.**

### 1. `prefers-reduced-motion` (hueco real y verificado)
`grep` sobre todo `src/` da **cero** ocurrencias de `prefers-reduced-motion`. `ACCESSIBILITY.md:122`
lo exige como parte del checklist manual y `MAP-UX.md:215` lo liga a DP-MX-07 (residual local).
Hoy el único movimiento es `scroll-behavior: smooth` del listado (6C). Se agrega el bloque
`@media (prefers-reduced-motion: reduce)` que lo neutraliza. Barato, verificable y hoy ausente.

### 2. Anuncio de selección con `aria-live="polite"`
`ACCESSIBILITY.md:101` es explícito: *"anunciar solo la **selección** (no el hover) con
`aria-live="polite"`"*. Hoy no existe. Se agrega una región viva que anuncia la ubicación
seleccionada (nombre + ocupación). **El hover no se anuncia** — eso es ruido para un lector de
pantalla, y la doc lo prohíbe expresamente.

### 3. `aria-describedby` del SVG → ayuda de teclado
Actualmente la ayuda ("Usá las teclas más, menos y cero...") está **dentro** del `aria-label` del
SVG. Eso infla el nombre accesible y hace que el lector lo lea entero en cada entrada. Se separa en
un `<desc>`/párrafo oculto referenciado por `aria-describedby`, dejando el `aria-label` corto.

### 4. Capas semánticas del render (MAP-ENGINE §5.2 / PERFORMANCE §5.4)
Los elementos ya se ordenan por `zIndex` y el viewport es un único `<g>` (6C) — o sea, *"zoom/pan solo
transforman el contenedor"* ya se cumple. Lo que falta es hacer **explícita** la separación en capas
con keys estables, para que el contrato de render sea auditable y no accidental.

### 5. Memoización: prohibición de recrear la colección
MAP-ENGINE §5.2: *"Prohibido recrear la colección completa en cada hover/zoom."* Hoy `rows()` es un
`computed` que se recalcula si cambia `map()` o `locationById()` — **no** depende de zoom/pan/
hover/selection, así que ya es correcto. 6D lo **cubre con un test** para que no se rompa por
accidente: assertar que cambiar zoom/hover/selection **no** vuelve a ejecutar el mapeo de elementos.

### 6. `ChangeDetectionStrategy.OnPush` + verificación de que el foco no se roba
OnPush ya está en todos los componentes del mapa. Lo que 6D verifica es un riesgo real de 6C:
que al re-renderizar por selección **el elemento con foco no pierda el foco** (bug clásico cuando
el `@for` re-crea nodos por un `track` inestable). El `track row.element.id` ya es estable; se
testea.

### Decisión tomada: P6-T5 NO se valida con métricas reales
`PHASES.md:123` dice *"sin degradación perceptible con ~50 ubicaciones (métrica en PHASE 12)"* y
`PERFORMANCE.md:105` lo confirma: la validación con percentiles p50/p95/p99 y Lighthouse es el
**gate de QA de la FASE 12**, no de la 6. `PERFORMANCE.md:102` además exige medir en **staging con
datos representativos**, nunca en local. Por lo tanto 6D entrega **el código y los tests
estructurales** de performance (capas, memoización, no-recreación) y **NO** un reporte de métricas
inventado. Las métricas quedan como deuda explícita para FASE 12. **No se fabrica evidencia.**

### Fuera de alcance de 6D (deuda, no de esta fase)
- Contadores agregados por ubicación y umbrales de zoom de densidad (`MAP-UX.md:182-187`): eso es
  dinámica de **cargas** sobre el mapa, y el backend todavía no expone cargas por ubicación en
  `GET /maps`. Es PHASE 7.
- Buscador/filtros del panel (`MAP-UX.md:200-201`): PHASE 7.
- Las métricas reales p50/p95/p99 y Lighthouse: FASE 12 (ver arriba).

## Decisiones sobre contradicciones documentales (D-P6-*)
Los docs se contradicen entre sí. El código sigue estas decisiones y **no se editan los docs**.

1. **D-P6-1 — Sin drag & drop.** `ADR-006:18` lista arrastrar cargas como interacción v1, pero
   `MAP-UX.md:14,161,313` y OQ-024 lo excluyen. Se sigue MAP-UX: mover cargas es por formulario.
2. **D-P6-2 — Umbrales de ocupación `<70 / 70–90 / >90`** (OQ-046, `MAP-VISUAL-GUIDELINES.md:66-68`).
   `MAP-UX.md:78-79` ("parcial 1–99 / lleno ≥100") está desactualizado.
3. **D-P6-3 — Sin animación.** `MAP-VISUAL-GUIDELINES.md:32,100` prohíbe pulse/flicker; `MAP-UX.md`
   pedía pulso para `IN_TRANSIT`. Se sigue el brand: estático + borde discontinuo + icono.
4. **D-P6-4 — Anillo de selección `#D97706`** (DP-MV-01), no `#F59E0B` (2.15:1, falla 3:1).
5. **D-P6-5 — Ningún estado se transmite sólo por borde o color.** Siempre icono/patrón/badge.
6. **D-P6-6 — Tokens `--map-occupancy-*`** (`MAP-VISUAL-GUIDELINES.md`), no `--map-sector-*`.
7. **D-P6-7 — `PATCH /maps` fuera de v1** (OQ-015). No se implementa.
8. **D-P6-8 — Sin nodo camión en v1.** BR-042 + `MAP-UX.md:104` + `MAP-ENGINE.md:175` coinciden en
   que el camión no es una ubicación; `DP-MV-04` queda diferido (no lo exige P6-H2..H5).
9. **D-P6-9 — `GET /maps` pagina con `page`/`limit`** (API-CONVENTIONS §4), aunque `API.md:280` no
   liste los parámetros. Default 25, max 100.
10. **D-P6-10 — `INACTIVE`/`MAINTENANCE` sin tokens propios**: se usa el fallback propuesto en
    `MAP-VISUAL-GUIDELINES.md:76,232` (surface-muted + hatch denso + badge), nunca sólo color.

## Tareas
- [x] P6-T1 Datos del mapa: `GET /maps` + `GET /maps/:id` con elementos; seed del plano (17). **6A**
- [x] P6-T2 Motor SVG de solo lectura: OperationalMap, MapLocation, MapLegend, MapToolbar. **6B**
- [x] P6-T3 Interacciones: zoom, pan, hover, selección + CapacityIndicator. **6C**
- [x] P6-T4 Accesibilidad: teclado, ARIA, listado alternativo equivalente. **6D**
- [x] P6-T5 Performance: capas, memoización, lazy render. **6D**

12. **Versionar los untracked de `odd/tasks/` - RESUELTO (2026-10-20).** Habia 10 archivos untracked (el tracker de PHASE 6 entre ellos: vivia solo en disco). Se versionaron en **dos commits separados por linea de trabajo**, para no mezclar procedencias: `1d9656b` (`docs(odd): version the rebuild line feature documents`, 7 archivos, 1460 inserciones - phase2..6 + rebuild-from-zero + handoff/2026-10-07) y `af0b829` (`docs(odd): version the Track line documents with a provenance header`, 4 archivos, 1013 inserciones - pre-rebuild + handoff/2026-10-01). Los 4 de Track llevan un header `DOCUMENTO DE LA LINEA TRACK - NO ES BASELINE DEL REBUILD` porque citan PRs/commits **no verificables en este clon**. Repo `odd` quedo **limpio, cero untracked**. Sin pushear.
13. **Pendiente D - `MASTER-SPEC.md:109` (`CapacityType`/`UNLIMITED`) - RESUELTO (2026-10-20).** Commit `43e0bd3` en `cargoops-docs` (`docs(master-spec): close the CapacityType/UNLIMITED drift in the canonical enums`). La seccion 4.3 se **contradecia dentro de una misma linea**: declaraba `QuantityUnit` con 6 valores y despues decia "`CapacityType` queda alineado: ... `UNLIMITED` ...", reintroduciendo el septimo. **Ninguno de los dos existe en el codigo**: `prisma/schema.prisma` declara `QuantityUnit` con exactamente 6 valores y no tiene `CapacityType`; la capacidad ilimitada es `capacity <= 0` (`capacity.calculator.ts:138`, `capacity-alert.service.ts:107`, con tests que citan D11). Los otros 6 archivos del corpus **ya lo decian bien** (`DATABASE.md:372`/`:540`, `DTOs.md:170`, `COMPONENTS.md:154`, `I18N.md:96`) - **MASTER-SPEC era el unico que quedo atras**. Ahora es consistente con ellos. Efecto colateral: `WriteAllLines` agrego el newline final que el archivo no tenia (1 byte, sin impacto semantico). **D11 cerrada en su origen.** Sin pushear.
14. **Pendiente F - anuncio de seleccion con unidades - RESUELTO (2026-10-20).** Commit `8550909` en `cargoops-frontend`, rama `fix/map-announcement-percent-anchor` (2 archivos, +8/-2). El anuncio terminaba con un `35 %` suelto, que un lector de pantalla lee como numero flotante sin referencia: `Sector l1 seleccionada - 35 m2 de 100 m2 35 %`. Ahora dice `... ocupacion 35 %`. **No se toco el formateador**: `formatLocationPercent` ya devuelve `35 %` correcto, y la unidad repetida en `35 m2 de 100 m2` es intencional (quitarla volveria ambiguos los dos numeros). Lo unico que faltaba era el sustantivo. El spec ya no asserta el `35 %` pelado (el texto nuevo lo contiene por substring, pasaria sin ejercitar el fix); ahora asserta `ocupacion 35 %` y que el texto no termine con la forma sin anclar. **Verificado por mutacion**: revertir la linea del componente hace fallar el test con `expected ... to contain "ocupacion 35 %"`. Gates: lint 0/0, format:check limpio, build OK, **267/267** en 34 archivos. **Mergeada `--no-ff` a `develop`** como `ab3d6ee`. Gates re-corridos FROM LA RAMA antes del merge y DE NUEVO post-merge sobre `develop`: lint 0/0, format:check limpio, build OK, **267/267** en 34 archivos. Verificacion de que el merge trajo el archivo correcto por **hash de blob** (develop == working tree, `566c4466`) - el `Select-String` de la consola dio un falso negativo por el encoding de acentos, no por el contenido.
15. **Push de `cargoops-docs` - HECHO (2026-10-20), autorizado por el usuario.** 2 commits publicados a `origin/docs/resolve-spec-code-drift-2`: `d8f63d5` (drift D11 + tokens de mapa) y `43e0bd3` (cierre de `CapacityType`/`UNLIMITED` en MASTER-SPEC 4.3). 18 archivos, +56/-52. Verificado contra el remoto real: `ls-remote` devuelve `43e0bd39b0c70b20027ada0f4a1bb3a55fd78eeb`, **identico al HEAD local**. `origin/main` **intacto** (`c696d52`). **La rama es nueva en el remoto** (no existia), asi que no se sobreescribio trabajo ajeno. **PR sin abrir** - requiere autorizacion explicita aparte.

## Punto de retorno (para retomar)

> **Última actualización: 2026-10-20 (cierre de sesión — FASE 6 CERRADA).**

### Estado de los repos (verificado, no inferido)
| Repo | Rama | HEAD | Árbol | Remotos |
|---|---|---|---|---|
| `cargoops-backend` | `develop` | `271c74e` | **limpio** | ninguno |
| `cargoops-frontend` | `develop` | `ab3d6ee` | **limpio** | ninguno |
| `cargoops-infrastructure` | `develop` | `4c78c11` | limpio | ninguno |
| `cargoops-docs` | `docs/resolve-spec-code-drift-2` | `43e0bd3` | limpio | **SI** - **PUSHEADA** (`origin/docs/resolve-spec-code-drift-2` == `43e0bd3`), PR sin abrir |
| `odd` | `main` | `699b6df` | **limpio** (todo versionado) | **SI** (`origin` = juanruiz-cv/odd) |

- **`main` intacto** en todos (frontend `72b40c2`, backend y infra sin tocar desde el scaffold).
- **Ninguna rama sin mergear a `develop`** en backend ni frontend (`git branch --no-merged develop` vacío).
- **Sin push, sin PR, sin remotos configurados.** Nada se publicó.
- Feature branches ya consumidas: `feature/p6-map-backend`, `feature/p6-map-engine`,
  `feature/p6-map-interactions` (todas mergeadas con `--no-ff`).

### Retomar exactamente acá
1. **FASE 6 ESTÁ CERRADA**: 6A/6B/6C/6D mergeadas. No hay nada a medias en ningún repo.
2. **La FASE 6 completa se cierra con 6D.** La siguiente fase es **PHASE 7**: dinámica de **cargas**
   sobre el mapa (contadores agregados por ubicación, umbrales de zoom de densidad — `MAP-UX.md:182-187`
   — y buscador/filtros del panel — `MAP-UX.md:200-201`), que el backend todavía no soporta en
   `GET /maps`. Punto de partida: `cargoops-frontend` `develop` @ `15533df`.
3. **Postgres local está detenido** (`cargoops-postgres`, Exited 0). Si la FASE 7 arranca solo en
   frontend **no hace falta levantarlo**. Para verificar seed/migraciones: `docker start cargoops-postgres`
   (Docker Desktop tiene que estar arriba).
4. **El orquestador debe releer este archivo** + el topic Engram `odd/cargoops-phase6/tasks` antes de
   planificar la FASE 7; después reconciliar contra el estado real de los repos (los hashes de arriba).

### Gates de referencia (no deben regresar)
- **Backend**: `npx prisma validate` · `npm run lint` · `npm run format:check` · `npm run build` ·
  `npm run test` **297/297** (21 archivos) · `npm run test:e2e` **85/85** (7 archivos).
- **Frontend**: `npm run build` · `npx ng test --watch=false` **267/267** (34 archivos) ·
  `npm run format:check` **limpio** (agregado el 2026-10-20).
  - **Prettier**: `.prettierrc` es la **única** fuente de verdad (commit `15533df`). Cuidado:
    Prettier resuelve `package.json` **antes** que `.prettierrc`, así que un bloque inline
    `"prettier"` en `package.json` **anula** el `.prettierrc`. Verificar con
    `npx prettier --find-config-path <archivo>`.
  - **`.gitattributes`** con `* text=auto eol=lf` neutraliza el `core.autocrlf=true` del host y
    evita que un checkout/merge rompa `format:check` por fin de línea (ver obs Engram 24).
  - **ESLint sigue SIN existir en el frontend** — decisión pendiente del usuario (ver pendientes).

### Pendientes que NO son de la FASE 6 (arrastrados, necesitan decisión tuya)

1. **Retirar `odd/tasks/cargoops-fase5-movements.md` - RESUELTO (2026-10-20).** El archivo fue **sellado** con un encabezado `ARTEFACTO RETIRADO - NO ES BASELINE` y despues **borrado** del repo `odd` en el commit `abe5067` (`docs(odd): remove the retired fase5-movements artifact`, 98 lineas eliminadas). Antes de borrar se **corrigio el propio header**: afirmaba "este repo no tiene remotos ni PRs" refiriendose a TODOS los repos, pero `cargoops-docs` (`juanruiz-cv/cargoops-docs`) y `odd` (`juanruiz-cv/odd`) **SI tienen remoto en GitHub y PRs mergeados** (docs #1-#6, odd #16/#17). El header ahora habla solo de los tres repos de codigo, que es donde el artefacto alega PRs. El argumento central sobrevive: **los 10 commits que cita no existen en ningun repo**. El borrado esta en `main` local de `odd`, **sin pushear** - espera autorizacion.
   con un encabezado prominente `ARTEFACTO RETIRADO — NO ES BASELINE` que documenta la evidencia de
   las afirmaciones "CUMPLIDA" fabricadas. **El archivo sigue existiendo físicamente**, pendiente de
   tu decisión de borrarlo (no se borró nada sin autorización).
2. **Contradicción `capacityUnit = 'UNLIMITED'` — RESUELTA EN LOS DOCS (2026-10-20).** Decisión:
   **PERSISTIDO (2026-10-20):** commit `d8f63d5` en la rama `docs/resolve-spec-code-drift-2` de `cargoops-docs` (17 archivos, +54/-50). Antes de este commit el trabajo estaba **solo en el working tree, sin persistir** — riesgo real detectado al auditar. La rama parte de `main` (`c696d52`) siguiendo el flujo de PRs del repo (#1-#6). **NO pusheado, sin PR** — espera decision del usuario. Verificado con auditoria read-only: los 17 archivos son trabajo NUEVO genuino (bucket A vacio = nada ya en main; bucket C vacio = cero borrados huerfanos, no pisa nada de main).
   **el código es la fuente de verdad.** Convención real (ya implementada, sin cambios): `capacity <= 0`
   significa ilimitado; `QuantityUnit` tiene exactamente UNITS/PALLETS/TONS/CUBIC_METERS/AREA/PERCENT
   (no existe `UNLIMITED`); los derivados `availableCapacity`/`capacityPercent` devuelven `null` cuando
   es ilimitado. Se corrigieron **22 archivos** bajo `cargoops-docs/docs/**` en dos pasadas. Claves:
   `backend/DTOs.md`, `backend/VALIDATION.md`, `backend/OPENAPI.md`, `backend/JOBS.md` (SQL
   `l.capacity_unit <> 'UNLIMITED'` → `l.capacity > 0`, ahora PostgreSQL válido),
   `brand/UI-GUIDELINES.md`, `brand/DESIGN-TOKENS.md`, `brand/MAP-VISUAL-GUIDELINES.md`,
   `frontend/DESIGN-SYSTEM.md`, `frontend/COMPONENTS.md`, `frontend/I18N.md`,
   `product/BUSINESS-RULES.md`, `product/PRODUCT-BACKLOG.md`, `product/USER-STORIES.md`,
   `architecture/DATABASE.md` (fila **D11 ahora marcada RESUELTA 2026-10-20**), `STANDARDS.md`,
   `architecture/ADR/ADR-004-PostgreSQL.md`. **Encuentro bonus:** `ux/MAP-UX.md:77-79` todavía describía
   un modelo de ocupación de dos bandas ("parcial 1–99 / lleno ≥100") que contradecía su propio archivo;
   corregido a las bandas canónicas **normal <70 / warning 70–90 / danger >90**. **SIN EDITAR
   (reservado al humano):** `MASTER-SPEC.md:109` todavía dice que `CapacityType` incluye `UNLIMITED`
   — es un registro canónico de decisiones, se dejó intacto deliberadamente y queda como referencia
   pendiente conocida. **Pregunta abierta, sin cambiar:** `brand/DESIGN-TOKENS.md:151` describe
   `--color-warning` como "capacidad ≥90%", que puede malinterpretarse como contradicción con las bandas
   de ocupación; en realidad es el color de **alerta de capacidad** (umbral de alerta 90%), otra cosa
   distinta a la banda *visual* de ocupación. Queda para tu confirmación.
3. **Token gap del mapa — RESUELTO EN LOS DOCS (2026-10-20).** Los docs ahora nombran los tokens
   exactamente como el código: `--map-occupancy-{ok,partial,full}-{fill,stroke}` (los **hex** nunca
   dejaron de coincidir), más los antes indocumentados `--map-surface-muted` (#E2E8F0),
   `--map-stroke` (#CBD5E1), `--map-selection` (#D97706) y `--map-pattern-hatch{,-light,-dense}`,
   agregados a `DESIGN-TOKENS.md` §11. `brand/MAP-VISUAL-GUIDELINES.md` actualizado a los mismos nombres.
4. **CORRECCIÓN — el repo de infra no tenía git perdido.** La nota anterior ("`cargoops-infra` no
   tiene `.git`") era **INCORRECTA**: la carpeta se llama en realidad **`cargoops-infrastructure`** y
   **sí tiene `.git`** (`develop` @ `4c78c11`). Existía además una carpeta huérfana vacía `cargoops-infra`
   que ya fue **eliminada**. No hay deuda de scaffolding. *(Limpieza adicional hecha en la raíz del
   proyecto, movida a un backup temporal, no borrada: `revtest.txt`, `revtest2.txt` — logs de test
   obsoletos que mostraban engañosamente un fallo ya corregido — y `_scaffold21` / `_scaffold_tmp`
   (~472 MB de scaffolds viejos sin trackear).)*
5. **Riesgo de PR gigante**: cuando se autorice push/PR, aplica la estrategia `ask-on-risk` cacheada.
   Los acumulados (5B 1645 + 5C 940 + 5D ~3600 + 5E ~750 + 6A 877 + 6B + 6C 2481 + **6D 435 líneas,
   10 archivos, todos bajo `src/app`**) superan largamente el heurístico de ~400; habrá que cortar en
   slices o pedir `size:exception`.
6. **Deuda de la feature del mapa**: muestra el primer plano por nombre, sin selector múltiple de planos
   (asume un solo plano; sigue vigente).
7. **Actualizado — todavía sin validar en navegador real ni backend vivo, PERO ahora posible.** La
   suite automatizada **nunca** ejercitó un navegador real (jsdom no implementa layout SVG ⇒ rasterizado,
   legibilidad de hatches a escala real y *feel* de pan/zoom sin validar) ni hizo una llamada real a
   `/api/v1/maps` (todo con `HttpTestingController`). **Novedad:** el fix del proxy dev-server (ver
   pendiente 8) **hace posible** una comprobación manual en navegador; el usuario inició sesión por la UI
   y funciona. La verificación formal en navegador real queda pendiente.
8. **NUEVO — fix dev del proxy sin commitear (espera tu decisión).** En `cargoops-frontend` el árbol de
   trabajo tiene `angular.json` (modificado: agrega `serve.options.proxyConfig = "proxy.conf.json"`) y
   `proxy.conf.json` (sin trackear: proxea `/api/v1` → `http://localhost:3000`). Corrigen un bug **real**:
   el frontend usa `apiBaseUrl: '/api/v1'` (ruta relativa, correcta para producción detrás de un reverse
   proxy), así que en dev el navegador pegaba a `localhost:4200/api/v1/...` y recibía 404. No existía
   `proxyConfig`. Verificado corregido: `POST http://localhost:4200/api/v1/auth/login` devuelve 200 a
   través del proxy. **Todavía no decidiste si commitearlos.**
9. **NUEVO — helper dev sin trackear en `cargoops-backend` (espera tu decisión).** `prisma/create-dev-user.ts`
   crea/resetea un usuario de login local porque el seed solo crea `seed-system` (documentado como que
   nunca inicia sesión) y el módulo de auth no tiene endpoint de registro. Pendiente de commit-o-borrar.
10. **RESUELTO (2026-10-20) — lint/format del frontend: Prettier sí, ESLint todavía no.** El usuario
    eligió resolver **solo Prettier** en esta pasada. Commit `15533df` en `cargoops-frontend` `develop`
    (29 archivos, +216/−205):
    - **Trampa real encontrada**: había **dos** configs de Prettier compitiendo y divergían. Prettier
      resuelve `package.json` **ANTES** que `.prettierrc`, así que el bloque inline `"prettier"` del
      `package.json` **ganaba** y el `.prettierrc` en disco **nunca se leía** (config muerta). Esa era
      la razón por la que el `<svg>` de `map-legend` se formateaba distinto a lo esperado.
    - **Unificado en `.prettierrc`** como única fuente de verdad (patrón del backend), con
      `printWidth 100 · singleQuote · trailingComma all · semi true` + override `parser: angular` para
      `*.html`. Se eliminó el bloque inline del `package.json`. Prettier 3 ya infiere el parser Angular
      para `.html`, así que el override es por explicitud, no por necesidad.
    - **Agregados**: `.prettierignore` (`dist · node_modules · coverage · .angular · package-lock.json`)
      y scripts `format` / `format:check` (glob `src/**/*.{ts,html,css}`).
    - **26 de 130 archivos (20%)** estaban fuera de formato, repartidos en `cargo`, `map`, `pages` y
      `shared/ui` — deuda de todo el proyecto, no de un módulo. Formateados. Verificado que son cambios
      **cosméticos puros** (`git diff -w` vs `git diff`: 214 vs 205 inserciones).
    - **Gates verificados desde el commit**: `format:check` limpio · build OK sin `anyComponentStyle` ·
      `ng test` **267/267 en 34 archivos**.
    - **Verificación crítica de EOL**: `core.autocrlf=true` en el host es una trampa conocida (rompió el
      gate en una sesión anterior). El `.gitattributes` con `* text=auto eol=lf` **ya está aplicado**:
      `git ls-files --eol` muestra `attr/text=auto eol=lf` y **0 archivos `w/crlf`**. `--end-of-line auto`
      pasa igual que el default ⇒ sin problema de EOL latente.
11. **RESUELTO (2026-10-20) - ESLint en el frontend.** Instalado y con las 47 violaciones de la primera corrida arregladas. Decision de alcance del usuario: **renombrar los 5 `no-output-native`** (no apagarlos). Dos commits:
    - **`50233fd`** (`fix(frontend): proxy dev-server API calls to the backend`) - 2 archivos, +11. El fix del proxy, commiteado **primero** por ser un bugfix funcional independiente del chore de lint (decision del usuario: dos unidades separadas para que un `git bisect` futuro no mezcle red con calidad).
    - **`72ef7a2`** (`chore(frontend): add eslint and fix its first findings`) - 46 archivos, +1454/-113 (`package-lock.json` aporta 1282 de esas lineas por las 88 dependencias nuevas).
    - **Versionado elegido**: `eslint ^10` + `@eslint/js ^10` + `typescript-eslint ^8` + `angular-eslint ^21.4.0`. **`angular-eslint@22` NO sirve**: exige `@angular-devkit/core >= 22.0.0` y el proyecto esta en Angular 21, habria roto peers. Verificado antes de instalar con `npm view angular-eslint@21.4.0 dependencies` (`@angular-devkit/core >= 21.0.0 < 22.0.0`, coincide exacto).
    - **Config** `eslint.config.mjs`: espeja la del backend - parser type-aware (`projectService`), set de reglas lean, **`recommendedTypeChecked` completo NO** (mismo motivo que el backend: su familia `no-unsafe-*` fuerza casts sobre superficie `any` de terceros). Mas `tsRecommended` + `templateRecommended` + `templateAccessibility` de angular-eslint y `processInlineTemplates`.
    - **Las 47 violaciones, clasificadas** (41 archivos): 37 `consistent-type-imports` (auto-fix con `--fix`, ninguna edicion a mano); **4 bugs reales de promesas** - `router.navigate()` huerfano en `login.component.ts:37`, `dashboard.component.ts:18-19` (`no-misused-promises`) y `refresh-token.interceptor.ts:61` (dentro de un `catchError`, el peor de los tres: ya manejaba un error y encima navegaba sin manejar el resultado) - arreglados con `void` + comentario que documenta la intencion, **no** con `eslint-disable`; 5 `no-output-native`; 1 `no-console` (`main.ts`, `console.error` -> `console.debug` segun la intencion documentada de la regla).
    - **Renames de `@Output`** (colisionaban con eventos DOM nativos): `cancel` -> `cancelled` (confirm-dialog, observation-dialog, move-cargo-form), `submit` -> `submitted` (observation-dialog), `select` -> `locationSelected` (map-location-list). Actualizados 8 bindings en templates y 14 referencias en specs. **NO tocados** (son eventos DOM nativos legitimos): los `(submit)="$event.preventDefault(); ..."` de `cargo-detail.component.html:106`, `move-cargo-form.component.ts:39` y `cargo-distribution.component.html:18`; `DialogBaseComponent.cancelRequest` (ya bien nombrado); el `cancelled` de `HttpTestingController` en `map-page.component.spec.ts`.
    - **Script agregado**: `"lint": "eslint \"src/**/*.ts\" \"src/**/*.html\""` (sin `--fix`, consistente con el estilo de `format`/`format:check`).
    - **Gates verificados por el orquestador desde el arbol commiteado**: `npm run lint` **0 errores / 0 warnings** (exit 0), `format:check` limpio, `build` OK sin `anyComponentStyle`, `ng test` **267/267 en 34 archivos** (sin regresion).
    - **Verificacion independiente del orquestador** (no el reporte del writer): los 5 renames presentes, los 8 bindings actualizados, **cero** referencias viejas a `.cancel`/`.select`/`.submit`, `(submit)` de `<form>` nativos intactos, y la config de ESLint **sin una sola regla debilitada**.
    - **Separacion entre commits verificada**: `git show --name-only 72ef7a2` no toca `angular.json` ni `proxy.conf.json` - superposicion cero.

## Criterios de aceptación (PHASES.md §10)
- `GET /maps` y locations devuelven estructura editable-ready.
- Render correcto de las 17 ubicaciones desde datos reales; leyenda presente.
- Navegación fluida; selección muestra detalle/capacidad.
- WCAG 2.2 AA core (focus visible, contraste, screen reader).
- Sin degradación perceptible con ~50 ubicaciones.

## Contrato de `GET /maps` (6A)
- `GET /api/v1/maps` → `{ items: [{ id, name, code, type, width, height, gridSize, status,
  version, updatedAt }], meta: { page, limit, totalItems, totalPages, hasNext } }`, permiso `map.read`.
- `GET /api/v1/maps/:id` → `{ id, name, code, type, width, height, gridSize, status, version,
  updatedAt, elements: [{ id, mapId, locationId, elementType, x, y, width, height, rotation,
  zIndex, fill, stroke, labelPosition, properties }] }`, permiso `map.read`, 404 `MAP_NOT_FOUND`.
- `Decimal` se proyecta a `number`; `properties` es JSON. Elementos ordenados por `zIndex` asc y
  sólo `deletedAt = null`.

## Evidencia

### 6A — backend (P6-T1) — COMPLETADO
- Rama `feature/p6-map-backend`, commits `1077d79` (`feat(maps): expose read-only map endpoints`) y
  `d2dfef1` (`feat(seed): seed the operational map with 17 location elements`).
- Merge `--no-ff` a `develop` → **`271c74e`**. Árbol limpio, sin diff contra la rama.
- Archivos: `src/modules/maps/{maps.module,maps.service,maps.controller}.ts`,
  `dto/{map-list-query,map-detail-query}.dto.ts`, `maps.service.spec.ts` (8 tests),
  `test/maps.e2e-spec.ts` (5 tests), `domain.exceptions.ts` (+`mapNotFound`), `app.module.ts`,
  `prisma/seed.ts`, `test/support/in-memory-prisma.ts` (+`map`/`mapElement`).
- Gates observados por el orquestador: `npx prisma validate` ok; `format:check` ok; `lint` ok;
  `build` ok; unit **297/297** (21 archivos, baseline 289/20); e2e **85/85** (7 archivos,
  baseline 80/6).
- Seed real contra Postgres: `Seed complete: 25 permissions, 3 roles, 17 locations, 15 cargos,
  4 active distribution segments, 1 maps, 17 map elements`; segunda corrida idéntica
  (idempotente). Verificado por SQL: 1 fila en `maps` (`predio`, ACTIVE, 1000x600, grid 10) y
  17 filas en `map_elements`, todas `LOCATION`, `z_index=3`, `label_position=CENTER`,
  `fill`/`stroke` NULL (los colores los deriva el frontend), coordenadas exactas al layout.
- `Decimal → number` probado de verdad: el unit spec siembra los Decimales como **strings** y
  asserta valores numéricos.
- Pendiente de 6A: ninguno. `PATCH /maps` no se implementó (D-P6-7).

### 6B — frontend (P6-T2) — COMPLETADO Y MERGEADO
- Rama `feature/p6-map-engine` (desde `develop` `783553c`), commits `50ee15f`
  (`feat(map): add map models, service and store`), `00a1c8e`
  (`feat(map): render the operational map read-only`) y `1b8c1da`
  (`fix(map): keep the location group in the accessibility tree`).
- Merge `--no-ff` a `develop` → **`f57a504`**. Árbol limpio, sin diff contra la rama. `main` intacto.
- Archivos: `core/models/map.model.ts`; `features/map/services/map.service.ts` (+spec);
  `features/map/state/map.store.ts` (+spec); `features/map/utils/map-format.ts` (+spec);
  `features/map/components/{map-location,map-legend,map-toolbar,operational-map}/`;
  `pages/map/map-page.component.*`; `app.routes.ts` (ruta lazy `map` + `authGuard`);
  `dashboard.component.html` (+ link "Mapa operativo").
- Gates observados por el orquestador: `npm run build` verde; `npx ng test --watch=false`
  **196/196** en **32 archivos** (baseline 154/154 en 24).
- D-P6-2 implementada como `occupancyBand()` (`ok <70` / `partial 70–90` / `full >90` / `none`),
  probada en los bordes 0, 69.9, 70, 90, 90.1, 100, >100 y `null`.
- D-P6-3 respetada: cero animaciones (el estado de carga es un bloque muted, sin shimmer).
- D-P6-5 respetada: cada banda y cada estado no-ACTIVE lleva hatch + texto visible.
- D-P6-10 verificada contra `MAP-VISUAL-GUIDELINES.md:76,232`: el fallback neutral + hatch denso +
  badge de texto es exactamente la propuesta DP-MV-03 (residual local de presentación, sin OQ).
- **Hallazgo de arquitectura — RESUELTO por decisión del usuario (2026-10-20):** se acepta el
  `<svg>` wrapper. Angular no permite que la plantilla de un componente hijo empiece con `<g>`
  (`NG8001: 'g' is not a known element`) porque el selector del componente nunca es parte del
  parse-tree del padre y no hereda el namespace SVG del `<svg>` que lo contiene. El wrapper no
  declara `viewBox` ni `width`/`height`: según la spec de SVG es un viewport 100%/100% que **no
  crea un sistema de coordenadas nuevo**, así el `<g>` interno sigue en unidades de usuario del
  mapa padre. `MapLocationComponent` conserva su contrato presentacional puro.
- **Bug de accesibilidad encontrado por el orquestador durante la revisión del diff y corregido
  en `1b8c1da`:** el wrapper `<svg class="map-location-host">` tenía `aria-hidden="true"`, lo que
  **anulaba** el `role="img"` + `aria-label` del `<g>` hijo y sacaba las 17 ubicaciones del árbol
  de accesibilidad — contradiciendo el criterio PHASES.md §10 "WCAG 2.2 AA core... screen reader".
  Se removió el `aria-hidden` del wrapper (los círculos/decoraciones internos ya no usan
  `aria-hidden`; el `role="img"` del grupo, presente desde `00a1c8e`, es la fuente accesible) y se
  agregó un spec de regresión que asserta `aria-hidden === null` en el wrapper y `role="img"` +
  `aria-label` con identidad, ocupación y estado en el grupo. El wrapper sigue sin aportar
  semántica propia: no tiene rol, así que no duplica el anuncio.
- **Nota vs docs (no es contradicción que el código deba seguir, es refinamiento):** `§5.2`
  (`MAP-VISUAL-GUIDELINES.md:72`) exige que el label de ocupación `"35/100 m³ · 35 %"` sea
  **obligatorio en el mapa**. 6B renderiza el nombre de la ubicación ("Sector 4") como label
  visible y lleva ocupación + estado en el `aria-label`; el detalle numérico completo vive en el
  panel de selección, que es 6C. **Queda como requisito entrante de 6C**: la ubicación con
  capacidad debe mostrar su label de ocupación con el par patrón + texto.
- **Token gap (diferido a 6C/6D):** el código consume `--map-occupancy-{ok,partial,full}-*` y
  `--map-surface-muted` / `--map-stroke`, pero `MAP-VISUAL-GUIDELINES.md` todavía nombra esos
  valores `--map-occupancy-{normal,warning,danger}-*` y `--map-zone-*`. Los **hex** coinciden con
  la tabla §5.2 y §5.1; lo que difiere es el **nombre del token**. `§5.8` (`:167`) exige que el
  valor viva en `DESIGN-TOKENS.md` §11; el rename no está hecho. Decidir si se alinean los nombres.
- **No observado (declarado por el writer)**: render real en navegador (jsdom no implementa layout
  SVG: no se valida el rasterizado ni la lectura de los hatches a escala real); ninguna llamada
  real a `/api/v1/maps` (todo con `HttpTestingController`); el store toma `items[0]` del listado
  (determinista porque el backend ordena por `name asc`) — si hubiera más de un plano, v1 muestra
  el primero por nombre.
- **Restricción para futuros specs del mapa**: el store hace `forkJoin` de `GET /maps` y
  `GET /locations`; si falla el primero, `forkJoin` cancela el segundo, así que un spec que quiera
  flushear la respuesta de locations después del error de maps muere con
  `Cannot flush a cancelled request`. Los specs existentes toman ambas refs, flushean primero el
  error de maps y assertan `locationsRequest.cancelled === true`.
- **Deuda de la feature**: el mapa muestra el primer plano por nombre, sin selector múltiple
  (diferible a 6C/6D).

### 6C — frontend (P6-T3) — COMPLETADO Y MERGEADO
- Rama `feature/p6-map-interactions` (desde `develop` `f57a504`), commits `9ac540d`
  (`feat(map): add viewport zoom and pan state`), `adc6475`
  (`feat(map): add hover, focus and selection to the canvas`) y `9f77300`
  (`feat(map): add the location panel and accessible list`). 27 archivos, +2481/−38.
- Merge `--no-ff` a `develop` → **`51a3912`**. Árbol limpio, sin diff contra la rama. `main` intacto.
- Archivos nuevos: `features/map/components/location-card/` y `map-location-list/`.
  Modificados: `map.store.ts` (+spec, viewport), `operational-map` (viewport `<g>` + interacciones),
  `map-location` (foco/hover/selección), `map-toolbar` (contrato de outputs), `pages/map/*`.
- Gates observados por el orquestador: `npm run build` verde (sin warning de `anyComponentStyle`);
  `npx ng test --watch=false` **259/259** en **34 archivos** (baseline 196/196 en 32).
- Viewport: `ZOOM_MIN = 0.4`, `ZOOM_MAX = 3`, `ZOOM_STEP = 0.2`, clamping en el store.
  `load()`/`reload()` **no** resetean el viewport (el usuario conserva la vista al reintentar).
- Interacciones: rueda (con `preventDefault`), doble click, arrastre con pointer capture,
  `+`/`-`/`0`/flechas por teclado. Todos los gestos tienen equivalente no-gestual (WCAG 2.5.1/2.1.1).
- **Tres defectos reales encontrados y corregidos por el writer (no eran "los tests pasan"):**
  1. `TS2445` en el spec: leer `fixture.componentInstance.store` directo rompía el *build* de
     `ng test` antes de correr un solo test. Reescrito para assertar estado protegido **por el DOM**
     (`.map-toolbar__zoom`), que es la convención del repo (`cargo-detail-page`).
  2. **`focus` ≠ `hover`**: conectar solo `mouseenter` habría dejado a los usuarios de teclado sin
     resaltado cruzado. Se cableó `focus()`/`blur()` de verdad.
  3. **Regresión de formato de porcentaje**: dos caminos distintos producían `72.5` vs `72,5`.
     Unificado en `formatLocationPercent` para ambos lugares.
- **Requirement entrante de 6B saldado**: el label de ocupación obligatorio
  (`MAP-VISUAL-GUIDELINES.md:72`) ahora se renderiza completo en `LocationCard` del panel, con la
  **unidad efectiva** (BR-041). En el canvas el label sigue siendo el nombre (densidad, §5.7).
- Sin animación (D-P6-3; la única excepción es `scroll-behavior: smooth` del listado, permitida).
  Sin drag & drop de cargas (D-P6-1). Anillo de selección `#D97706` (D-P6-4).
- **No observado (persiste de 6B)**: render real en navegador — jsdom no implementa layout SVG, así
  que el rasterizado, la legibilidad de los hatches a escala real y el *feel* del pan/zoom quedan
  sin verificar. Ninguna llamada real a `/api/v1/maps` (todo con `HttpTestingController`).
- **Deuda de la feature (sin cambios)**: el mapa muestra el primer plano por nombre, sin selector
  múltiple de planos.
- **Token gap (sigue diferido a 6D o a una decisión explícita)**: el código consume
  `--map-occupancy-{ok,partial,full}-*` y `--map-surface-muted`/`--map-stroke`, mientras
  `MAP-VISUAL-GUIDELINES.md` los nombra `--map-occupancy-{normal,warning,danger}-*` y `--map-zone-*`.
  Los **hex coinciden**; difiere el **nombre**. `§5.8:167` exige que el valor viva en
  `DESIGN-TOKENS.md` §11. Sigue sin resolverse y **no se edita ningún doc sin decisión del usuario**.

### 6D — frontend (P6-T4/T5) — COMPLETADO Y MERGEADO
- Rama `feature/p6-map-a11y-perf` (desde `develop` `51a3912`), commits:
  - `31c6abe` (`fix(map): honour prefers-reduced-motion in the accessible list`)
  - `3ddae13` (`feat(map): announce the selected location to screen readers`)
  - `5c6066b` (`feat(map): describe the keyboard controls instead of the accessible name`)
  - `3f61971` (`refactor(map): make the render layers explicit`)
- Merge `--no-ff` a `develop` → **`37d5ff0`**. 10 archivos, +435/−18, todos bajo `src/app`.
- **Verificación independiente (agente verificador fresco, no el writer)** observó:
  - `npm run build` → `Application bundle generation complete`, **sin warning de `anyComponentStyle`**.
  - `npx ng test --watch=false` → **267/267** en **34 archivos** (la rama partía de 259/259 en 34 sobre
    el `develop` viejo).
  - Re-verificación post-merge sobre `develop`: build verde, **267/267** en **34 archivos**.
- **Los 6 ítems planificados, cada uno CONFIRMADO en código con un test:**
  1. **`prefers-reduced-motion`**: bloque en `map-location-list.component.css:27` (`scroll-behavior:
     smooth`) + `:39-42` (media query que lo neutraliza a `auto`). Test en
     `map-location-list.component.spec.ts:239-253`.
  2. **`aria-live="polite"` que anuncia SOLO la selección**: `map-page.component.html:14`, computado en
     `map-page.component.ts:90-113` (lee únicamente `selectedLocationId`). Tests en
     `map-page.component.spec.ts:296-350`.
  3. **Ayuda de teclado movida de `aria-label` a `aria-describedby`**: `operational-map.component.html:21-22`
     + `<desc>` en `:39-41`, id en `operational-map.component.ts:84`. Test en
     `operational-map.component.spec.ts:322-350`.
  4. **Capas de render explícitas con keys estables**: `operational-map.component.html:86-101`, CSS en
     `operational-map.component.css:73-93`, `track row.element.id` en `html:91`. Tests en
     `operational-map.component.spec.ts:161-233`.
  5. **Memoización (las filas NO se recalculan con zoom/hover/selección)**: `rows` computed en
     `operational-map.component.ts:99-111`, hover/selección son signals separados. Test en
     `operational-map.component.spec.ts:239-311` (identidad de nodo DOM + marcador plantado + un caso
     **control** que cambia `map` y prueba que la sonda SÍ puede fallar — el test más fuerte del diff).
  6. **`OnPush` + foco no robado**: `operational-map.component.ts:44`, `map-page.component.ts:51`. Test en
     `operational-map.component.spec.ts:313-333` (asserta el `document.activeElement` real).
- **Limitaciones honestas (NO sobrevender estos ítems):**
  - El test del ítem 1 asserta que el **TEXTO** de la regla CSS existe, **NO** que el movimiento se
    suprima de verdad: jsdom no evalúa media queries. El comportamiento real es una comprobación
    manual/en navegador real.
  - Ítems 2 y 3: jsdom **no** computa el nombre accesible. Los tests prueban el **contrato del DOM**
    (el texto de la live-region sigue solo a la selección; `aria-describedby` resuelve a un `<desc>` no
    vacío) pero **no pueden** probar que un lector de pantalla lo anuncie.
  - Ítem 4: la capa `structure` se entrega intencionalmente vacía (el backend sirve solo elementos
    `LOCATION`) — **por diseño, no es un defecto**.
  - **P6-T5 NO se valida con métricas reales** (p50/p95/p99, Lighthouse): `PHASES.md:123` +
    `PERFORMANCE.md:105` ubican eso en el **gate de QA de la FASE 12**, y `PERFORMANCE.md:102` exige
    medir en **staging con datos representativos**. 6D entregó **código + tests estructurales** solamente;
    sin reporte de performance fabricado. Queda como **deuda explícita para la FASE 12**.


