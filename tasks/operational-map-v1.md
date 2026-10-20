# Mapa operativo v1 — scope corregido
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


**Estado:** especificado por las fuentes canónicas; sin construir.
**Reescrito:** 2026-10-02. La versión anterior de este doc estaba equivocada en 5 de 6
preguntas (ver «Corrección» abajo). Mantiene el mismo nombre de archivo a propósito: es el
mismo ítem, no uno nuevo.

## Objetivo

Construir la vista de mapa operativo de v1: lectura, selección y hover sobre el plano del
predio, con ocupación por ubicación y deep-link al detalle. Es la Fase 6-7 "Mapa + Dashboard"
del roadmap (`TEAM-ROLES.md` §6).

## El error de la versión anterior

Este doc se escribió sin leer la documentación del mapa. Declaró cinco ítems como «real design
work». Los cinco ya estaban especificados:

| # | Pregunta que este doc daba por abierta | Realidad | Fuente canónica |
| --- | --- | --- | --- |
| M01 | ¿De dónde sale el plano: asset en repo o configurable? | **No es asset.** El asset estático se evaluó y **se rechazó**. El plano se genera declarativamente en Angular desde coordenadas estructuradas en PostgreSQL | `ADR/ADR-006-SVG-Map-Engine.md`, alternativa 1: «Rechazada: viola BR-020 y MASTER-SPEC §3 — el layout es configurable» |
| M02 | ¿Cómo se modelan las ubicaciones sobre el plano? | `LocationVisual { id, x, y, width, height, rotation, zIndex, fill, stroke, labelPosition }` + `properties`, persistido vía `Map`/`MapElement` | `MAP-ENGINE.md` §5.1, §5.7; `DATABASE.md` (Location → 0..1 MapElement) |
| M03 | ¿Qué hace hover y qué hace selección? | Hover con tooltip de estado/capacidad; selección abre el detalle. **Y no hay drag en v1** | `MAP-UX.md` §4, §4.1; OQ-024 resuelta |
| M04 | ¿Deep-link bidireccional? | Mapa → detalle, sí. El inverso no está especificado | `ROUTING.md` L100 (`LocationCard` → «Ver detalle») |
| M05 | ¿Endpoint nuevo de ocupación? | **No hace falta.** Ya existe en el contrato | `API.md` §7.1–7.3; `DATABASE.md` L189–192 |
| M06 | ¿Entra en v1 o se saca de `MASTER-SPEC`? | **Entra en v1.** Lo que sale de v1 es el *editor* (OQ-015), no el mapa | `ROUTING.md` L46; `TEAM-ROLES.md` §6 |

La premisa del doc anterior —«la spec menciona un motor SVG pero no define de dónde llega el
asset»— era falsa. `ADR-006` define exactamente eso.

**Consecuencia práctica:** el mapa no está especificado en ningún lado. Está **sin construir**.
Estas tareas son de implementación, no de diseño.

## Estado real del código (verificado 2026-10-02)

| Capa | Estado |
| --- | --- |
| `prisma/schema.prisma` | **Listo.** `Map` (L364-383) y `MapElement` (L385-410) con la geometría completa; `Location` con `capacity`, `capacityUnit`, `occupiedCapacity`, `availableCapacity` (L333-336) |
| Migraciones | **Listo.** `20260924185728_init` incluye los modelos de mapa |
| Permisos | **Listo.** `map.read` y `map.edit` ya sembrados en `seed-rbac.ts` L71-76 |
| 17 ubicaciones | **Listo.** `seed.ts` L1 las siembra; L7 difiere la geometría «al mapa phase» |
| `src/maps` | **Ausente.** 11 módulos existen, `maps` no |
| `OperationalMap` | **Ausente.** `features/` tiene solo `cargos` y `locations` |
| Assets `.svg` | **Cero, y es lo correcto** según ADR-006 |

## Decisiones tomadas (no requieren más input)

**Rutas del SPA — idioma.** Los docs (`MASTER-SPEC` L339-340 y `ROUTING.md` L41-49, L97) usan
español; el código usa inglés. Decidido el 2026-10-02: **los documentos se corrigen a inglés**,
siguiendo el principio de que el código es la autoridad. El mapa entra como `/operational-map`.

**Residuales de `MAP-ENGINE.md` §9** — todos marcados «sin OQ asignada», o sea sin decisión de
negocio adjunta; son del Architect:

- **M2** (unidades del `Map`): se mantienen **unidades abstractas normalizadas**, no metros
  reales. Es la postura de `MAP-ENGINE.md` §5.7 y de `DATABASE.md` D6.
- **M5** (umbral de virtualización): **sin virtualización en v1**. El presupuesto documentado
  es ~17 ubicaciones + cientos de cargas, dentro del margen de SVG. Se mide contra señal real
  cuando exista (`PERFORMANCE.md` P4).
- **M7** (render tolerante): **tolerante**. Las piezas fuera del `viewBox` se recortan; no
  rompen el render. Coherente con `API.md` §13.4.
- **M8** (labels flotantes): se acepta como **limitación de v1**. Sin lógica de colisión.

**Residuales de `MAP-UX.md`**:

- **DP-MX-01** (zoom 0.4x–3.0x, niveles de densidad): se adoptan como **provisionales**,
  sujetas a ajuste con datos reales de `PERFORMANCE.md` W2.
- **DP-MX-02** (clustering): **contador clickeable por ubicación en v1**, sin mini-grid
  expandible. Al volumen de v1 no aporta y suma superficie de UI.
- **DP-MX-07** (animación de `IN_TRANSIT`): **se anima respetando `prefers-reduced-motion`**.
  No es una preferencia de producto, es el criterio de accesibilidad de `MAP-UX.md` §6.2.

**Fuera de alcance de v1** (decidido por OQ-015/OQ-024, no es decisión de este doc): el editor
de planos y el drag de cargas. `PATCH /maps/:id` se documenta pero no se implementa en esta
fase; el movimiento de carga se hace por diálogo transaccional con observación obligatoria
(BR-006/007).

## Alcance autorizado

Backend (`cargoops-backend`), frontend (`cargoops-frontend`) y docs (`cargoops-docs`).

## Checklist

- [x] **D1** Corregir las rutas del SPA a inglés en toda la documentación. **Hecho** —
      `cargoops-docs` PR #5, squash `671441b`, 8 archivos, 74/74. Verificado post-merge: 0 rutas
      y 0 `features/*` en español. El mapa queda como `/operational-map` y `features/operational-map`.
- [x] **B1** Módulo backend `src/maps`: `GET /maps`, `GET /maps/:id`, permiso `map.read`,
      envelope de errores estándar, 401 / 404 `MAP_NOT_FOUND`. **Hecho** — `cargoops-backend`
      PR #22, squash `cbdf100`, 11 archivos, +1813 (1359 fuente + 454 tests). 437 → 462 unit,
      252 → 278 e2e. `PATCH` deliberadamente ausente por OQ-015.
- [x] **B2** Geometría en seed: `Map` «PREDIO» + un `MapElement` por cada una de las 17
      ubicaciones. **Hecho** — `cargoops-backend` PR #23, squash `173e1c9`, 5 archivos,
      +1158/-12. 462 → 482 unit, 278 → 289 e2e. Lienzo 1600×900, `gridSize` 20.
- [x] **F1** Ruta `/operational-map` + reintroducir la entrada del sidebar (que en PR #15 se
      removió justamente porque no había vista detrás). **Hecho** — `cargoops-frontend` PR #16,
      squash `068eac0`. La promesa del sidebar ahora tiene vista detrás.
- [x] **F2** `OperationalMap`: SVG con `viewBox`, capas, memoización con Signals/`computed`,
      lazy-load. **Hecho** — mismo PR #16. 22 archivos, +3242/-9. 206 → 299 tests, 22 → 29
      archivos. `operational-map-page` es chunk lazy propio, 11.51 kB. `sortByReadingOrder`
      resuelve la condición de lectura que B2 no podía.
- [x] **F3** `MapToolbar` (zoom/pan), `MapLegend`, y la pieza por ubicación.
      **Hecho** — `cargoops-frontend` PR #17, squash `7689aac`. Zoom clavado a 0.4x–3.0x
      (DP-MX-01, provisional hasta datos reales de PERFORMANCE W2).
- [x] **F4** Interacciones v1: hover con tooltip de estado/capacidad, selección. **Sin drag.**
      **Hecho** — mismo PR #17. La paridad teclado/puntero es **estructural**, no una
      convención: `focus` escribe el mismo `hoveredPieceId` que el puntero y ambas superficies
      renderizan `describeLocation`. Un test afirma que el texto de ambos caminos es **igual**.
- [x] **F5** Ocupación: % sobre la unidad efectiva (BR-041), desglose por unidad, umbrales
      `<70` / `70–90` / `>90` (OQ-046) y estado `SOBRECUPACIÓN` (OQ-043/BR-036).
      **Hecho** — `cargoops-frontend` PR #20 → squash `f16115c`. 19 archivos, +3858/-81.
      **Sólo frontend**, por decisión del owner: el backend ya tenía todo. Corregí una
      conclusión mía previa — la ocupación **es alcanzable** y es *opt-in* por
      `GET /locations?includeCapacity=true`. `null` significa "no computado", nunca "vacío"
      (`locations.response.ts:32`): yo había verificado `null` sin pedir el flag. El
      desglose por unidad (`segmentsByUnit`) vive en `GET /locations/:id/capacity`, **no**
      en el item de lista, y se carga **on demand** sobre la pieza hover/focus, memoizado
      por id — no 17 peticiones al cargar.
- [ ] **F6** Accesibilidad WCAG 2.2 AA: navegación por teclado, roles ARIA, y alternativa
      textual (listado de ubicaciones con ocupación). El mapa nunca es la única vía.
- [ ] **F7** Deep-link mapa → `/locations/:id`.
- [ ] **Q1** E2E del mapa, incluyendo la alternativa textual (`TEAM-ROLES.md` §5.8 lista
      «plano» entre los casos críticos).

## Criterios de aceptación

- `MASTER-SPEC` y el código dicen lo mismo sobre el mapa: está en v1 y existe.
- El sidebar tiene entrada de mapa **solo** si hay vista detrás.
- El deep-link de `MASTER-SPEC` L340 existe, con el nombre de ruta vigente.
- El mapa es operable por teclado y tiene alternativa textual (WCAG 2.2 AA).
- Ningún dato de ocupación se recalcula en el frontend: viene del backend.

## Checks aplicables

Backend: `npm run lint`, `npm run typecheck`, `npm test`, `npm run build`.
Frontend: `npm run lint`, `npm run format:check`, `npm test`, `npm run build`.

**Baseline frontend: 206 tests en 22 archivos.** Si una feature del mapa mueve ese número, es
porque agrega cobertura.

**Baseline backend: 437 unit / 252 e2e.** Recordar que `npm test` **no** corre los e2e: los
e2e son `*.e2e-spec.ts` y `vitest.config.ts` incluye `**/*.spec.ts`. Solo `npm run test:e2e`
ejercita la raíz de composición.

## TDD

No hay configuración de TDD estricto en ninguno de los tres repos; se trabaja en modo
estándar, con los checks de arriba como puerta. Test runner: `npm test` (vitest).

## Entrega

Pronóstico: **2000–3000 líneas** de código authored, muy por encima del presupuesto de ~400 por
unidad de entrega.

**Estrategia: `exception-ok`** (decidido 2026-10-03 al cruzar el presupuesto con B1). Un módulo
NestJS cohesivo partido produce estados intermedios que no corren, y los tests no pueden
separarse del código que cubren. La feature sigue siendo una sola unidad conceptual; se entrega
en work units cohesivos, cada uno mergeado con `size:exception`.

Work units: D1 → B1 → B2 → F1/F2 → F3/F4 → F5/F6 → F7 → Q1.

## Progreso

- [x] **D1** — `cargoops-docs` PR #5 → squash `671441b`. Verificación post-merge: inventario
  estricto en 0 para las 10 rutas SPA en español, el segmento `movimientos` y los 15
  `features/*` en español. `/roles` se dejó intacto a propósito (palabra idéntica en español e
  inglés; los 10 hits que lo contaban como español eran prosa tipo `configuración/roles` sin
  barra inicial). Sin checks: el repo de docs no tiene CI. Cero BOM.

- [x] **F1/F2** — `cargoops-frontend` PR #16 → squash `068eac0`. 22 archivos, +3242/-9.
  206 → 299 tests en 22 → 29 archivos, cero bajas. El baseline **no se creyó**: el writer lo
  midió sobre un worktree limpio de `main`. Verificado por mí antes de mergear:
  **cero fuga de scope** de F3/F4/F5 (`toolbar`, `zoom`, `legend`, `tooltip`, `occupancy`,
  `@Output`, `draggable`, `cdkDrag` → 0 hits cada uno), `viewBox` derivado del lienzo del API
  (`buildMapViewBox`, sin constante), y el orden de lectura consumido de verdad en el
  `computed` de `layers`, no implementado y abandonado.

  **El subagent reportó los cuatro comandos verdes y el CI "pending" — y el CI estaba en rojo.**
  El test `sends an unauthenticated visitor from /operational-map to /login` fallaba con
  `expected [] to have a length of 1`. Mismo commit, mismo resultado distinto: no era una
  aserción mal escrita, era no-determinismo.

  La causa era el arnés, no el producto: `drainPendingRequests` se autoconfinaba con un
  **presupuesto de 20 ticks**. Resolver la cadena lazy `app.routes → main-layout →
  operational-map` lleva una cantidad impredecible de ticks — muchos más en un runner
  cargado que en una máquina rápida. Un presupuesto contado en ticks es un "rendirse después
  de N intentos" disfrazado: falla en la máquina lenta y pasa en la local, el peor modo de
  fallo posible para un test que fija una propiedad de seguridad. Reboundado a deadline de
  reloj, verificado verde 4 veces seguidas y 3 más bajo **96 quemadores de CPU en 32 núcleos
  lógicos** (sobre-suscripción 3×), que es la saturación que reprodujo el fallo original.
  Corregido y amalgamado en el mismo commit: la unidad no estaba entregada mientras su propio
  test se pusiera en rojo.
  propio sobre los 17 rectángulos: **0 solapamientos en 136 pares, 0 fuera de canvas, 0 fuera de
  grilla**. CI `success` con provenance (`run.headSha` = `pr.headRefOid` = `5882e7d`).
  `Map.status = ACTIVE` porque `maps.service.ts` **no** filtra por estado: sembrar `DRAFT`
  publicaría un plano que se declara no publicado. `createdById` resuelto por **orden de
  siembra** (`seedRbac` → ubicaciones → `seedMap`) con el admin dev, sin UUID literal. Idempotencia
  probada corriendo `db:seed` tres veces: 17 → 0 → 0. Paleta verbatim de
  `MAP-VISUAL-GUIDELINES.md` §5.1/§5.3, nada inventado.
  El run anterior fallaba en **precisión de citas**: tres `file:line` erróneas en comentarios
  (`MAP-UX.md L186` era la tabla de zoom, no el orden de tabulación). En un archivo cuyo valor
  es la trazabilidad, una línea mal citada es un defecto real.

- [x] **B1** — `cargoops-backend` PR #22 → squash `cbdf100`. Verificado de forma independiente:
  cero mutadores (solo GET), `map.read` en ambos endpoints vía `@RequirePermissions`
  denegado-por-defecto, casing camelCase pese al mapeo snake_case de Prisma, y soft-delete
  bien leído: `Map` no tiene `deletedAt` (se retira por `MapStatus.ARCHIVED`) mientras
  `MapElement` sí filtra por `deletedAt`. CI `success` con provenance verificada
  (`run.headSha` = `pr.headRefOid` = `be74836`). Mergeado con `size:exception` aprobado por el
  owner: un módulo NestJS partido deja estados intermedios que no corren.

**Próximo paso:** F6 — accesibilidad WCAG 2.2 AA.

- [x] **F5** — `cargoops-frontend` PR #20 → squash `f16115c`. 19 archivos, +3858/-81.
  **416 → 551 tests, 34 → 36 archivos**, verificado por mí sobre el commit de merge, no sobre
  la rama. CI `37239842365` `success` con provenance (`run.headSha` = `pr.headRefOid`).

  **El backend no necesitó nada.** `capacityUnit` ya es la unidad efectiva (BR-041, resuelta
  en servidor: override o default del `LocationType`). El límite de sobreocupación es
  `capacity * (1 + overOccupationLimitPercent/100)`, no un `+10` hardcodeado aunque ése sea el
  default almacenado. `selectOccupied` ya suma sólo el bucket compatible (BR-033/035).

  **Los tokens no existían.** `MASTER-SPEC.md:361` exige textualmente el namespace
  `--map-occupancy-*`; había 16 tokens `--map-*` y ninguno era de ocupación. Los diez se
  agregaron **verbatim** de `DESIGN-TOKENS.md` L335-342 (bandas y umbrales) y L357-358
  (hatch, `rgba(15,23,42,0.18)` y `0.10`). Sólo dentro de `:root`, **0 selectores** — verificado
  con un chequeo propio, no con un grep que sólo imprimiera el nombre de archivo.

  **La ocupación se pinta en un `<rect>` aparte**, sin atributo de presentación. Un `fill` de
  stylesheet sobre la pieza le robaría el paint a la API, que es el atributo que lleva los datos
  auditados. Por eso la banda es un overlay y no un color de la pieza.

  **Dos incumplimientos que el writer declaró y yo no había pedido.** Eligió no aterrizar el
  label de ocupación y dejó un anillo en vez del badge `SOBRECUPACIÓN`. Ambos son
  **"obligatorio"/"se agrega"** en `MAP-VISUAL-GUIDELINES.md` L72 y L71. Los dos se
  corrigieron. El label **extiende el `<text>` existente** — hay exactamente uno por pieza, y un
  test lo fija (`toHaveLength(1)` sobre 17 labels): L66-68 ponen nombre y ocupación en el
  **mismo** string, así que el motivo del "competiría con el nombre del sector" no se sostenía.
  Cuatro formatos, y L74/L75 **pisan** a L66-68: `UNLIMITED` → `"Sin límite"` en
  `muted-strong`; unidades incompatibles → sólo la ocupación por unidad y el **porcentaje
  ausente, no un guion** (un guion afirmaría que hay un porcentaje que no se pudo leer).
  El badge es un `<text>` con `stroke` + `paint-order: stroke fill`, no un chip `<rect>`: un
  chip tendría que medir el avance tipográfico de un string en una fuente que el código no
  carga.

  **Cortes de banda fijados por test**, porque `capacity-indicator.ts` corta distinto:
  `90 → warning`, `90.01 → danger`, `95 → danger`.

  ### Tres cosas que F5 NO cierra

  1. **`capacity-indicator.ts` contradice a OQ-046 y sigue así.** Corta en
     `percent >= 70 → 'warn'` y reserva `danger` para *sobre capacidad* (BR-036), así que una
     ubicación al **95% lee `warn` ahí y `danger` en el mapa**. No se tocó: es un componente
     ya publicado y está fuera de alcance. Requiere una decisión de quien tiene las dos
     superficies.
  2. **La alerta `CAPACITY` de L71/L73 no está implementada** — necesita una lectura de alertas
     que la página del mapa no hace. Reportada, no improvisada.
  3. **Solapamiento residual de labels a 0.8x**: dos sectores *sobreocupados* adyacentes se
     pisan hasta **28 unidades** (20 a 0.4x en forma contador). Medido con anchos de avance
     reales de `segoeui.ttf`/`arial.ttf`, no estimado. Los dos arreglos posibles **cambian el
     documento** — subir el límite de L157 contradice el límite enunciado, y aplicar el
     "gana el marcador de alerta" de L161 es la lógica de colisión que `MAP-ENGINE.md` §5.7
     M8 difiere. Encoger el texto es lo único que §5.7 L31 prohíbe. Los números quedan
     pineados en un test: es deuda heredada, no algo que se redescubra. La decisión es de
     quien tiene §5.7.

## F3/F4 entregado — y cuatro contradicciones en los documentos canónicos

PR #17 → squash `7689aac`. 27 archivos, +3323/-96. **299 → 416 tests, 29 → 34 archivos.**
Chunk de producción 27.68 kB (los 102 kB que reporté antes eran del dev server, no de prod).

**El writer revirtió por su cuenta el cambio de `tsconfig.spec.json`.** El razonamiento del run
anterior era correcto sobre *qué* afirmar — un hex leído del fuente que se manda prueba algo; un
constante string dentro del spec no prueba nada — pero `"types": ["node"]` es de alcance repo:
le da `process`, `Buffer` y `setTimeout` devolviendo `NodeJS.Timeout` a specs cuyo runtime es
el navegador. Ahora los dos specs que leen el fuente llevan `/// <reference types="node" />`,
que incluye `@types/node` solo para ese archivo. **Verificado por mí: blob ID de
`tsconfig.spec.json` idéntico al de `main`.** Y es *load-bearing*: sin la referencia aparecen
seis `TS2591`.

**Verifiqué `tokens.scss` propiedad por propiedad contra `main`:** 16 tokens nuevos (14
`--map-*` de paint, `--map-selection-ring`, `--space-56`), **cero declaraciones eliminadas** (las
3 líneas del diff son un bloque de comentario reescrito), y **cero reglas de selector nuevas** —
solo `:root`. Eso último es lo que importa: nada puede robarle el paint a la API.

**La colisión de color es real:** `--color-warning-hover` y `--map-zone-galpon-stroke` son
ambos `#d97706`. Verificado. La mitigación — duplicar el grosor del trazo (1px → 2px hover,
4px seleccionado) con `non-scaling-stroke` en vez de confiar en el tono — es la correcta.

### Corrección: de las cuatro "contradicciones", dos no existían

Las reporté después de que las encontrara el writer de F3/F4. Al verificar cada cita antes de
actuar, **dos no se sostenían** — y una de mis propias afirmaciones tampoco.

1. **REAL, y más grave de lo que dije: `MAP-ENGINE.md` §5.6 L129 se contradice a sí mismo.** El
   bloque ilustrativo muestra `<svg role="img">`, pero **L147 de la misma sección** exige
   *"piezas con `role="button"` y `tabindex`"*. Bajo un `role="img"` global el subárbol colapsa
   en una imagen y los botones quedan inalcanzables. O sea: la ilustración era la mitad incorrecta
   de una contradicción interna, no un outlier frente a otros documentos.
   La autoridad es explícita: `MAP-VISUAL-GUIDELINES.md` L185 dice *"el SVG no se expone como
   `role="img"` global (se pierde el recorrido por elementos); el área del mapa usa `role="group"`"*.
   **Corregido** en `c696d52`.
2. **NO EXISTÍA. `MAP-UX.md` §6.2 es consistente.** Yo dije que L212 describía las piezas como
   `role="group"` mientras L213 exigía una pieza enfocable. Leídas completas: **L211** habla de los
   `<g>` **de capa**; **L212** dice *"todas las ubicaciones y marcadores son focables (`tabindex`
   implícito por rol interactivo)"* — que es exactamente `role="button"`. Capas y piezas son
   elementos distintos; no hay contradicción. **Lo peor: el error está replicado en el código
   mergeado** — `operational-map.html:30-35` cita esa contradicción inexistente como justificación
   de `role="button"`. La conclusión es correcta; el razonamiento que la sostiene es falso.
3. **REAL — `aria-describedby` → leyenda.** La leyenda es un disclosure colapsable: su id solo
   existe abierta, y un `aria-describedby` colgante se anuncia como nada. **Corregido** en
   `c696d52` manteniendo el requisito y agregando la condición que lo vuelve seguro, en el mismo
   lugar (L186 está citado por `operational-map.scss:83`, así que agregar una línea lo rompía).
4. **REAL — la colisión `#D97706`.** `--color-warning-hover` y `--map-zone-galpon-stroke` son el
   mismo valor. Documentada en `DESIGN-TOKENS.md` §11 con la mitigación (grosor de trazo, no tono).

**Y yo tenía mal el `labelPosition: "TOP"`.** Lo reporté como drift comparando el ejemplo
ilustrativo de `MAP-ENGINE.md` §5.1 contra el texto normativo de al lado, sin comprobar que ambos
dijeran lo mismo. **`DTOs.md` L212 — línea byte-idéntica en `main` y en el commit — valida
`@IsIn(['TOP','BOTTOM','LEFT','RIGHT','CENTER'])`**: `TOP` es un valor válido, y §5.1 no se
contradecía. El único drift real de ese ejemplo era `zIndex: 1` → `3`. Corregido sólo eso, con el
motivo agregado en la línea en vez de cambiar el valor en silencio.

**Lección, y es la segunda vez en este trabajo.** Conté "cuatro documentos contra uno" sin verificar
que los cuatro hablaran del **mismo elemento**. `ACCESSIBILITY` y `COMPONENTS` hablan de
`MapLocation`; `MAP-UX` habla de las capas; el outlier real era `MAP-ENGINE`. Un conteo de
documentos que no incluye "sobre qué elemento" no es una verificación: es aritmética.

## Defectos de citation en el frontend — 4 abiertos, ninguno en un PR de docs

`cargoops-docs` PR #6 → squash `c696d52`, 6 archivos +11/-7. Repo sin CI, así que el writer
construyó la puerta: 74 archivos, 125 citas por línea resueltas, 1401 referencias de sección, y
**un auto-test del propio checker** (MAP-UX §6.2 resuelve, §99.99 no) para probar que la puerta
puede fallar. Eso sí es una puerta.

**Verifiqué yo la parte que sostiene el PR entero:** cada cita por número de línea hacia estos docs
viene de **código enviado, nunca de otro documento**. Por eso `MAP-ENGINE.md`, `API.md`,
`MAP-VISUAL-GUIDELINES.md` y `STATE-MANAGEMENT.md` tienen que ser **neutros en conteo de líneas**,
y lo son (227/415/243/210, sin cambios). Sólo crecieron `DTOs.md` (393→395) y `DESIGN-TOKENS.md`
(477→479), y verifiqué las dos inserciones: en `DTOs.md` después de L212 — la única cita por línea
que existe hacia ese archivo, desde `seed-map.spec.ts:182` — y en `DESIGN-TOKENS.md` en L361,
después de L333. Ninguna rompe nada. Las 125 siguen resolviendo al mismo texto.

Los cuatro defectos, todos en `cargoops-frontend`, ninguno arreglable desde el repo de docs.
**Los 4 de cita están cerrados** en PR #18 → `7d8e0ae` y PR #19 → `c14c7ee`; el 3 (los `<g>`) es
de comportamiento y queda para F6:

1. ~~**`operational-map.html:9` quedó viejo por este PR.**~~ **Corregido en `7d8e0ae`.** El texto
   nuevo dice lo que de verdad pasó: *no eran dos opiniones, `MAP-ENGINE.md` se contradecía a sí
   mismo* — L129 ilustraba `role="img"` mientras L147, **la misma sección**, exige `role="button"`
   + `tabindex` en piezas enfocables, y un `role="img"` global vuelve eso insatisfacible.
2. ~~**`operational-map.html:30-35` cita una contradicción que no existe.**~~ **Corregido en
   `7d8e0ae`.** El razonamiento nuevo es el real: §6.2 es consistente porque **capas ≠ piezas**
   (L211 los `<g>` de capa, L212 las piezas, L213 el anillo de foco). Registrado como **open item**
   que L211 pide `role="group"` + `aria-label` en las capas y los dos `<g>` no lo tienen.
3. **Las capas `<g>` no llevan `role="group"` ni `aria-label`** (`operational-map.html:53` y `:70`),
   que `MAP-UX.md` §6.2 L211 exige y `seed-map.ts:283` ya cita como canónico. Gap real de WCAG
   1.3.1 — **pertenece a F6**, no a una corrección de citas. No lo metí en un PR de texto.
4. ~~**`map-legend.spec.ts` cita `MAP-VISUAL-GUIDELINES §6.2 L216`.**~~ **Corregido en `7d8e0ae`** a
   `MAP-UX.md` L216.

**El primer PR destapó dos citas más rotas, de la misma familia, corregidas en `c14c7ee`:**

- **BR-004 atribuido al documento equivocado.** `map-legend.spec.ts` y `utils/map-legend.ts`
  citaban `MAP-VISUAL-GUIDELINES §5.3 L87` para "MAINTENANCE bloquea movimientos". L87 habla de
  *labels sobre fills suaves*. La autoridad es **`docs/product/BUSINESS-RULES.md` L105**: *"el
  destino de un movimiento debe tener `status ACTIVE`"*. El texto nuevo nombra el error de
  categoría: **un documento sobre color no tiene autoridad sobre una regla de negocio.**
- **Una cita que no tenía destino.** Tres sitios atribuían a `§3 L29` que `OTRO` no tiene token
  `--map-*`. L29 son umbrales de ocupación, y **`OTRO` aparece una sola vez en todo el documento**,
  en L17, como lista de tipos: no hay línea que lo diga. **El claim es verdadero por ausencia.**
  No lo repuntamos a L17: eso fabricaría una cita que el documento nunca hizo. El texto
  nuevo declara que el claim descansa en una ausencia, checkeable en `tokens.scss`.
- **Bonus del writer:** la misma frase decía *"`Map.type` es `'OTRO'`"*, y `MapType` es
  **`'PREDIO'`** a secas. `OTRO` es un `LocationType`. Corregido dentro de la reescritura.

Ambos PR son **solo-comentario**: verifiqué el diff línea por línea, ni una etiqueta, ni un
atributo, ni una línea ejecutable se movió. 416/416 en 34 archivos antes y después, que es lo que
tiene que dar un PR de texto.

**Un gate que no existía, y era una mentira.** El `grep -v '^\s*[0-9]*:[-+]\s*<!--'` que le pasé
espera un prefijo `NNNN:` que `git diff` sólo emite en las cabeceras `@@`: **el filtro es un no-op**
y no puede dar ningún veredicto. El writer lo refutó en vez de obedecer y decir que pasaba. Lo
reemplazó por comparar el código con los comentarios eliminados — colapsando whitespace, porque
quitar un bloque multilínea desplaza todas las líneas siguientes y finge una violación — y después
lo reforzar con SHA256 sobre los 27 archivos emitidos: idénticos byte a byte salvo el `timestamp`
de `ngsw.json`. Su control negativo detectó 5 bugs en su propio arnés, uno de los cuales era **una
prueba vacía**: leía el lado "después" con `git show :path`, que es el **índice** — igual a `HEAD`
para trabajo sin stagear — así que comparaba `HEAD` contra sí mismo y habría aprobado cualquier
edición. Un arnés que no puede fallar no es una verificación.

**Y cuatro veces asserted hoy sin verificar.** `§4.4` donde era `§4.3`; `docs/business/` donde era
`docs/product/`; `docs/validation/` donde era `docs/backend/`; y `map-legend.ts:101` como una cita
cuando el archivo con ese nombre son dos y el que tiene L101 es otro. La causa del último: mi
grep imprimía **sólo el nombre de archivo**, así que no podía distinguir
`components/map-legend/map-legend.ts` de `src/app/utils/map-legend.ts`, y asumí. Un grep que no
puede desambiguar no informa: no dice nada.

**Y una puerta que nadie había pedido:** ningún comando local type-chequea los specs. `ng test`
usa esbuild, `oxlint --type-aware` sólo reporta reglas de lint, y `ng build` usa
`tsconfig.app.json`, que excluye los specs. O sea: **un entorno de tipos roto en un spec pasa
las cuatro verificaciones.** `npx tsc -p tsconfig.spec.json --noEmit` sale limpio (verificado,
`exit=0`) y debería entrar al CI.

## F3/F4 en curso — HISTÓRICO, ya resuelto (entregado como `7689aac`)

> Esto describe el estado del 2026-10-04 temprano, cuando el subagent fue interrumpido.
> **Ya no es el estado actual** — ver "F3/F4 entregado" arriba. Se conserva porque el
> aprendizaje sobre las 4 interrupciones y los falsos negativos de diagnóstico sigue valiendo.

Rama `feat/operational-map-chrome` sobre `main` @ `068eac0`. **27 rutas en el índice, nada
commiteado, sin PR, rama NO pusheada.** El subagent fue interrumpido antes de verificar.

**Lo que hay.** 15 archivos nuevos y 12 modificados. Componentes `map-toolbar/` y
`map-legend/` (`.ts`/`.html`/`.scss`/`.spec.ts` cada uno). Utilidades nuevas con sus specs:
`map-viewport.ts` (zoom/pan, 0.4x–3.0x), `map-transit.ts` (animación `IN_TRANSIT` con
`prefers-reduced-motion`), `map-piece-detail.ts`, `map-legend.ts`. Se integra el toolbar y la
leyenda en `operational-map.*` y en la página.

**Cuatro cambios caen FUERA del directorio del mapa. Son los primeros puntos a revisar mañana:**

1. **`tsconfig.spec.json`** agrega `"node"` a `types`. El comentario del propio cambio lo
   justifica: un spec lee el archivo fuente del componente desde disco para probar que **no**
   contiene un literal hex, porque una constante string dentro del spec no probaría nada sobre
   el archivo que se manda. El razonamiento es bueno, pero **altera el entorno de tipos de
   todos los specs del repo** para acomodar una técnica de un solo test. Merece un juicio
   explícito, no pasar por alto.
2. **`src/styles/tokens.scss` +48 líneas.** Es lo más grande fuera del mapa. Hay que confirmar
   que solo agrega tokens `--map-*` nuevos y **no** sobreescribe tokens existentes: F1/F2 ya
   había dejado documentada la tensión de que una regla de stylesheet le gana al atributo de
   presentación SVG y le roba el paint a la API.
3. **`location-labels.ts` +16** agrega `declaredCapacityLabel`.
4. **`capacity-indicator.ts`**: 1 línea, delega en ese helper nuevo. Es un refactor sobre una
   feature ya entregada (`locations`), no sobre el mapa. Comportamiento aparentemente igual,
   pero toca código en producción fuera del alcance.

**Lo que NO hay que perder de vista:** ninguna verificación. Cero resultados de test, cero
lint, cero build, y **el CI nunca corrió** porque la rama no está pusheada. El precedente de
esta sesión obliga: el run anterior reportó cuatro comandos verdes y el CI estaba rojo.

**Al retomar:** partir de esta rama, NO de `main`. Leer lo escrito, juzgarlo, corregir lo que
esté mal — empezando por los cuatro puntos de alcance — correr las cuatro verificaciones,
pushear, abrir PR y **mirar el CI hasta el final**. Un work unit no está entregado mientras su
propio test esté en rojo.

## Notas de ejecución

**El conteo inicial era incorrecto.** El inventario que le pasé el writer daba 44 ocurrencias;
el real era 52. La causa: mi primer conteo contaba *líneas*, no ocurrencias, así que una línea
con `/cargas` y `/cargas/nueva` contaba una sola vez. El archivo y los sitios eran los mismos,
así que no hubo desvío de alcance, pero la cifra que reporté no era la buena.

**Mapear a ciegas rompe el mapeo.** La primera pasada del writer aplicó solo las 7 rutas del
briefing y dejó `/auditoria`, `/configuracion` y `/usuarios` en español — no por descuido sino
porque **nunca le di esos mapeos**. La autoridad correcta ya estaba en el repo: los
encabezados de `API.md` §8-9 (`Dashboard, Alerts, Audit` / `Trucks, Users, Roles,
Notifications, Settings, Health`). Segunda pasada los cerró.

**Una ruta puede quedar medio traducida.** `/cargas/:id/movimientos/:movId` sobrevivió a la
primera pasada porque `movimientos` no estaba en ninguna tabla de mapeo. Un mapeo parcial
produce rutas híbridas, que son peores que no tocar nada: parecen correctas en revisión.

**Reemplazo por substring simple corrompe el documento.** `capacidad/cargas` matchea `/cargas`,
`cargas/ubicaciones` matchea `/ubicaciones`, `editar/eliminar/planos` matchea `/planos`,
`dashboard/alertas` matchea `/alertas`. Las 7 rutas resultaron inseguras bajo búsqueda laxa.
Todo el renombrado se hizo con formas delimitadas por backtick.

**Pendiente menor, fuera de D1:** un store llamado `plano-editor.store` dentro de
`features/maps/state/` está documentado con nombre en español mientras sus hermanos están en
inglés. El archivo no existe todavía; se resuelve cuando se construya el feature de mapas.

**~~B1 extiende el contrato de `GET /maps/:id`~~ — CORREGIDO: este drift NO existía. Lo registré
mal y lo verifiqué levantando el proyecto el 2026-10-04.** La respuesta real de
`GET /maps/:id` es exactamente `{ "data": { "id", "name", "elements": [...] } }` — verificado
contra la API corriendo, no leyendo el código. **No lleva `width`, `height` ni `gridSize`.**

Lo que yo había anotado era una inferencia no verificada: asumí que el detalle cargaba el lienzo
porque "un render SVG necesita el `viewBox`". Falso. El lienzo viaja en el **listado**
(`MapSummaryDto`: `code`, `name`, `width`, `height`, `gridSize`, `status`), y por eso la
pantalla hace **tres** lecturas y no dos: `MapSummaryDto` para el lienzo y el estado,
`MapDetailDto` para los elementos, y locations para los nombres y la capacidad. El split es
deliberado en el servidor ("a client that picked a map from the list already holds them").

**`API.md` §7.2 está correcto tal como está. No hay nada que escribir de vuelta.** Un ítem de
documentación que iba a abrir era trabajo inventado sobre un drift que no existe.

**Drift de `elementType`, sin resolver a propósito.** `DTOs.md` §4.7 valida
`['LOCATION','LABEL','POLYGON','PATH']`; el comentario en `schema.prisma` L389 dice
`LOCATION | LABEL | SHAPE | ZONE`. Como `elementType` es `String` y no un enum, la base no
restringe nada, así que no hay conflicto de ejecución — pero los dos documentos no coinciden.
Solo afecta al `PATCH`, que está fuera de v1. Reportado, no tocado.

**El orden de lectura del plano no es el orden de la API.** Los 17 elementos van todos en
`zIndex: 3`, así que el desempate de pintado es el `id`, y `GET /maps/:id` los devuelve en orden
uuid. `MAP-UX.md` L211 quiere que el orden de tabulación siga el orden de lectura del plano. Un
consumidor tiene que ordenarlos explícitamente — un `seed` no puede resolverlo. **Esto es una
condición de F2, no una omisión de B2.**

### Cómo levantar el entorno (verificado 2026-10-04, no estaba escrito en ningún lado)

Hay **cuatro** repos, no tres: `cargoops-infrastructure` es el que tiene el compose de dev.

```
# 1. infra (desde cargoops-infrastructure\)
docker compose -f docker-compose.dev.yml up -d postgres redis
#    minio está en perfil opcional y su registro es inalcanzable (nota L43-44 del compose): no hace falta

# 2. API (desde cargoops-backend\)
npm run start:dev                      # puerto 3000

# 3. web (desde cargoops-frontend\)
npm start                              # puerto 4200
```

El volumen `pg_data` persiste entre corridas: la base y las 17 filas de B2 ya estaban.
Contenedores: `cargoops-postgres-dev` (5432) y `cargoops-redis-dev` (6379), ambos con healthcheck.

**Login: el endpoint pide `username`, no `email`.** `POST /api/v1/auth/login` con
`{ "email": ... }` devuelve `400 VALIDATION_ERROR` pidiendo `username`. Seed: `admin` /
`operator` / `viewer`, password `DevPass123!` (`prisma/seed-rbac.ts:172`).

**El frontend habla cross-origin directo, sin proxy.** `src/app/core/api.config.ts` fija
`API_BASE_URL = 'http://localhost:3000/api/v1'`, así que depende del CORS del backend.
Verificado: `app.setup.ts:22` usa allow-list `CORS_ORIGINS ?? 'http://localhost:4200'`,
credentialed y **nunca wildcard** (`SECURITY.md` §5.1). Comprobado con `curl.exe`:
`4200` recibe `Access-Control-Allow-Origin` + `Allow-Credentials: true`, el preflight `OPTIONS`
devuelve `204` con los métodos y headers correctos, y un origin no permitido **no** recibe la
cabecera (el `200` igual, lo bloquea el navegador). `curl.exe` y no `Invoke-WebRequest`: este
último falla con "modo no interactivo" en este entorno.

**Los envelopes no son arrays.** `GET /maps` devuelve `{ data: { items: [...], meta: {...} } }`;
`GET /maps/:id` devuelve `{ data: { id, name, elements } }`. Leer `$r.data` como array da un
elemento que es el sobre entero, y `$items[0].id` sale vacío.

**Las columnas son snake_case aunque el modelo sea camelCase.** `maps.grid_size`,
`map_elements.location_id`, `users.active` — es el `@map()` de Prisma. Consultar `"gridSize"`
falla con `column "gridsize" does not exist` y sugiere `grid_size`.

**Cuidado al truncar uuids para diagnosticar.** Todos los `locationId` arrancan con `01a0d857`
porque son **uuidv7** generados en la misma ventana de milisegundos, y los 8 primeros
caracteres son el prefijo de timestamp. Leí los 17 como idénticos y casi reporto un bug que no
existe. Contá los ids completos antes de declarar una colisión.

**En PowerShell, `\"` no es escape.** Lo pasa literal y destroza los identificadores
entrecomillados (`unterminated quoted identifier`). Usar cadenas con comillas simples:
`'select ... "gridSize" ...'`.

### Un drift que inventé y una lección

Registré que "B1 extiende `API.md` §7.2 con `width`/`height`/`gridSize`" y dejé anotado
abrir un ítem de documentación para escribirlo de vuelta. **Era falso**, y lo acepté porque lo
escribí sin verificar. Levantando el proyecto se ve: el detalle devuelve exactamente
`{ id, name, elements }` — el lienzo viaja en el **listado**, no en el detalle. Ese ítem de
documentación era trabajo sobre un drift inexistente.

La lección no es "verificar más", ya lo decía: es que **una inferencia razonada no es una
verificación**, aunque suene inevitable. Lo dije porque "el SVG necesita el `viewBox`", y de
ahí salté a "el detalle debe traer el lienzo". Ninguna de las dos pasos está en el código.
Un ítem de documentación que se abre sobre una inferencia no verificada es trabajo
inventado, y cuesta más que no hacerlo.

**El galpón no tiene perímetro todavía.** El stroke `#CBD5E1` de los sectores es correcto según
`MAP-ENGINE.md` §5.1 L57, pero significa que el galpón no tiene contorno hasta que exista el
outline de zona, que `MAP-ENGINE.md` §5.5 L114-116 difiere a v1.1.

**~~El bloque JSON ilustrativo de `MAP-ENGINE.md` §5.1~~ — RESUELTO en `c696d52`.** Reportaba
también un drift en `labelPosition: "TOP"` que **no era drift**: `DTOs.md` L212 valida ese valor.
Sólo se corrigió `zIndex: 1` → `3`, que sí contradecía §5.2 L71. La mitad falsa de esta entrada
es la misma clase de error que la de arriba: comparar dos textos sin comprobar que hablan de lo
mismo.

**`API.md` §7.2 usa `#F59E0B` como fill de ejemplo**, y `MAP-VISUAL-GUIDELINES.md` DP-MV-01 lo
marca como color que falla el contraste 3:1. El seed no lo usa: usa los `--map-*` auditados.
