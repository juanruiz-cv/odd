# CargoOps — Fase 0: Documentación (feature: cargoops-documentation)

## Objetivo
Generar la documentación completa de CargoOps (producto, arquitectura, frontend, backend, UX/UI, marca, QA, DevOps, roadmap, ADRs, backlog, DoD) a partir del "PROMPT MAESTRO — FASE 1: DOCUMENTATION FIRST" (61 secciones). Sin implementación de código.

## Alcance autorizado
- Crear 60+ archivos Markdown bajo `cargoops-docs/`. → **Creados: 73 archivos .md (0 código, 0 vacíos).**
- Definir decisiones canónicas en `docs/MASTER-SPEC.md` (fuente de verdad). → ✅
- NO: código de producción, dependencias, instalaciones, repos git, frontend/backend. → ✅ Verificado.

## Contexto clave
- Workspace: `C:\Users\juan-\Desktop\IA\Track` (sin git — sin work-unit commits; pendiente crear repos en FASE 1).
- Proyecto Engram: `track`.
- Stack objetivo (draft): Angular 20+ / NestJS / PostgreSQL / Prisma / Redis + BullMQ / S3 / JWT / Modular Monolith / SVG map engine.
- Idioma de contenido de docs: español profesional/neutral; filenames en inglés (decisión 2026-09-23, revisable).

## Checklist (routes) — ESTADO FINAL
- [x] T00 Explorar workspace + proyecto Engram — inline
- [x] T01 Escribir MASTER-SPEC.md + seed OPEN-QUESTIONS.md — inline (orquestador)
- [x] T02-T11 Writers W1..W10 — delegated (writer trigger: múltiples archivos no triviales por grupo). 1er intento abortado por reinicio de server (34 archivos creados); relanzado con scope preciso de faltantes → 73/73.
- [x] T12 Gatekeeper: glob (73 .md / 0 no-md / 0 vacíos), spot-check (README, OPEN-QUESTIONS), sin placeholders (matches "Todo" = español, no stubs).
- [x] T13 README.md + OPEN-QUESTIONS.md consolidados (creados por writers; leídos y verificados por orquestador).
- [x] T14 Reporte final + memoria (engram mirrors + decisiones) + cierre.

## AMPLIACIÓN 0.2 — secciones 62-70 (distribución M:N de cargas y ubicaciones, 2026-09-23)
Cambio de modelo: Cargo↔Location pasa de N:1 a MANY-TO-MANY vía `CargoLocation`; capacidad por unidad (no booleano ocupado); distribución multi-ubicación; movimientos parciales; descarga parcial; consistencia de cantidades.

- [x] T15 Corregir MASTER-SPEC.md (v0.2): entidad CargoLocation, relaciones M:N, enums QuantityUnit/CargoLocationStatus, decisión 8, seeds de distribución, BR-032..040, máquina de estados, alerta CAPACITY, endpoints de distribución, componentes, mapa §69, manifest, §24 — inline (orquestador)
- [x] T16 Actualizar OPEN-QUESTIONS.md: OQ-002 resuelta, OQ-009 refinada, nuevas OQ-041..045 — inline (orquestador)
- [x] T15 Corregir MASTER-SPEC.md (v0.2): entidad CargoLocation, relaciones M:N, enums QuantityUnit/CargoLocationStatus, decisión 8, seeds de distribución, BR-032..040, máquina de estados, alerta CAPACITY, endpoints de distribución, componentes, mapa §69, manifest, §24 — inline (orquestador)
- [x] T16 Actualizar OPEN-QUESTIONS.md: OQ-002 resuelta, OQ-009 refinada, nuevas OQ-041..047 — inline (orquestador)
- [x] T17 W-DB (architecture): ARCHITECTURE, DATABASE, MAP-ENGINE, PERFORMANCE — delegated ✅ (reports + pendientes OQ) 
- [x] T18 W-API (backend): API, API-CONVENTIONS, DTOs, VALIDATION, ERROR-HANDLING, MODULES, JOBS — delegated ✅
- [x] T19 W-FE (frontend): FRONTEND-ARCHITECTURE, COMPONENTS, STATE-MANAGEMENT, DESIGN-SYSTEM, ROUTING — delegated ✅
- [x] T20 W-UX (ux): USER-FLOWS, SCREENS, MAP-UX — delegated ✅
- [x] T21 W-BR (product): PRD, PRODUCT-BACKLOG, USER-STORIES, USE-CASES, BUSINESS-RULES — delegated ✅
- [x] T22 W-QA (qa): TEST-PLAN, TEST-CASES, E2E-SCENARIOS, ACCEPTANCE-CRITERIA, QA-STRATEGY — delegated ✅
- [x] T23 W-BRAND (brand): MAP-VISUAL-GUIDELINES, DESIGN-TOKENS — delegated ✅
- [x] T24 W-ROAD (roadmap): ROADMAP, PHASES, IMPLEMENTATION-PLAN — delegated ✅
- [x] T25 Gatekeeper ampliación: 73/73 .md, 0 código, 0 vacíos; 2 pasadas correctivas (W-FIX1 backend/arch, W-FIX2 product/qa/ux/fe, W-FIX3 residuales) → referencias legacy solo transicionales; cierre + memoria — inline ✅

## AMPLIACIÓN 0.4 — resolución de OQ bloqueantes e IDs (2026-09-23, 5ª sesión)
Decisiones de negocio tomadas por el usuario (una por una): OQ-001 regex canónica · OQ-004 egreso completo · OQ-008 30/40 días corridos · OQ-017 POST export-pdf · OQ-018 catálogo ADR-009 · OQ-022 observación siempre · OQ-025 matriz estricta · OQ-029 §4.4.3 completa · OQ-030 rezago op/ secuestro ADMIN · ID-009 BR-029 patente MERCOSUR canónica.

- [x] T26 Confirmar decisiones OQ-001/004/008/017/018/022/025/029/030 + ID-009 con el usuario — inline (orquestador, 5ª sesión, una pregunta por decisión, opciones recomendadas aceptadas)
- [x] T27 Actualizar MASTER-SPEC.md → v0.4: BR-002 regex, BR-006 observación alta, BR-014/015 alertas 30/40, BR-018 POST, BR-029 canónica, BR-043 egreso, BR-044 matriz estricta, BR-045 transiciones, BR-046 permisos, AlertSeverity (§4.3), decisión 11, §7 máquina de estados, §8 RBAC, §9 alertas, §10 POST, §11.6, §24 — inline (orquestador)
- [x] T28 Actualizar OPEN-QUESTIONS.md: OQ-001/004/008/017/018/022/025/029/030 → ✅ Resueltas; ID-009 → ✅ Resuelta; nota de consolidación 5ª sesión — inline (orquestador)
- [x] T29 Alinear IDs restantes (ID-001…008, ID-010) en los docs afectados + corregir referencia OQ-017→OQ-029 en VALIDATION.md — delegated ✅ (writer; 2 pasadas: 27 archivos + residuales; reportes verificados por gatekeeper)
- [x] T30 Gatekeeper: readback de archivos tocados + spot-check consistencia con MASTER-SPEC v0.4 (grep: 0 GET export-pdf activos; 0 OQ-017 mal apuntados; W1-Q9 resuelta en BUSINESS-RULES/USE-CASES; conteo 73 .md/0 vacíos en README) + cierre y memoria — inline ✅
- [x] T31 Cierre residuales finales: BUSINESS-RULES.md:128/560, USE-CASES.md:60/117/222/516 (W1-Q9→OQ-022 resuelta), README.md:74 (72→73), API-CONVENTIONS.md:230 (nota obsoleta) — inline (orquestador)
- [x] T32 Estado final OPEN-QUESTIONS.md: OQ-001/004/008/017/018/022/025/029/030 ✅ + ID-001…010 ✅; quedan abiertas OQ 🟡/🟢 (019, 020, 021, 023, 024, 026-028, 031-036…) y OQ-043…047 — no bloqueantes para FASE 1/3 — inline (orquestador)

## AMPLIACIÓN 0.5 — resolución COMPLETA de OQ (2026-09-24, 6ª sesión)
El usuario decidió resolver TODAS las OQ restantes antes de abrir FASE 1. 30 decisiones tomadas (una tanda de 4 lotes de preguntas + 1 tanda final con OQ-016/046): OQ-003 (1 carga→1 camión), OQ-005 (PDF Puppeteer/Chromium), OQ-006 (MinIO self-hosted), OQ-007 (Redis+BullMQ v1), OQ-010 (PWA mínima, SSR diferido), OQ-011/023 (notificaciones in-app v1), OQ-012/034 (es-AR + i18n listo, sin conmutador), OQ-013 (flujos núcleo primero), OQ-014 (sectores m² + Plazoleta camiones), OQ-015 (mapa estático), OQ-016 (RPO 15m/RTO 4h), OQ-019 (auditoría OPERATOR propios eventos), OQ-020 (presupuestos provisionales), OQ-021 (UPDATE genérico audit alertas), OQ-024 (sin drag&drop), OQ-026 (AlertStatus roles), OQ-027 (código inmutable), OQ-028 (formularios ADMIN), OQ-031 (dashboard en vivo), OQ-032 (online + Idempotency-Key), OQ-033 (sin landing), OQ-035 (ruta hija movimiento), OQ-036 (polling TTL 30s), OQ-043 (sobreocupación ADMIN +10%), OQ-044 (sin conversión), OQ-045 (percentage derivado), OQ-046 (umbrales 70/90), OQ-047 (listado ubicaciones), OQ-009 (cerrada por OQ-041/046).

- [x] T33 Confirmar con el usuario las 28 OQ pendientes en 4 lotes temáticos + tanda final (OQ-016/046) — inline (orquestador, 6ª sesión, opciones recomendadas aceptadas)
- [x] T34 Actualizar OPEN-QUESTIONS.md: OQ-003/005/006/007/009/010/011/012/013/014/015/016/019/020/021/023/024/026/027/028/031/032/033/034/035/036/043/044/045/046/047 → ✅ Resueltas; nota de consolidación 6ª sesión — inline (orquestador)
- [x] T35 Actualizar MASTER-SPEC.md → v0.5: §0 resumen 0.5, BR-036 ampliada (OQ-043), BR-047 (código inmutable), BR-048 (sin conversión), BR-049 (percentage), BR-050 (1 carga→1 camión), BR-051 (dashboard en vivo), BR-052 (Idempotency-Key), §4.1 entidades, §4.2 relación camión, §4.3 enums, §4.4 decisiones 12-23, §8 RBAC (auditoría OPERATOR + AlertStatus), §9 notificaciones/roles, §11.2/11.5/11.6 stack+mapa+PDF, §12 UX/UI, §13 umbrales, §14 performance/DR, §17 estado, §24 resumen — inline (orquestador)
- [x] T36 Actualizar ADR-012 → Accepted (Redis+BullMQ v1, OQ-007) y ADR-013 → Accepted completo (Puppeteer, OQ-005) — inline (orquestador)
- [x] T37 Replicar decisiones en ~30 docs de detalle — delegated ✅ (writer; 2 pasadas: detección de mislabels BR-044/045 en API.md §13 filas 3/4/7/9, confirmación de mapeo canónico, corrección + lote completo; 35+ refs stale corregidas; verificado por grep final: cero placeholders `[PENDIENTE OQ-xxx]`)
- [x] T38 Gatekeeper: readback MASTER-SPEC v0.5 + OPEN-QUESTIONS (0 OQ abiertas) + ADR-012/013 Accepted + grep BR-04x consistente; cierre y memoria — inline (orquestador)

## AMPLIACIÓN 0.5.1 — corrección de residuales/mislabels FASE 0 (2026-09-24, 7ª sesión)
El usuario eligió corregir quirúrgicamente los mislabels BR/OQ reportados en la 6ª sesión (opción "Residuales FASE 0"), sin tocar decisiones canónicas: MASTER-SPEC.md y OPEN-QUESTIONS.md INTACTOS (verificado por grep/readback al cierre).

- [x] T39 Corregir mislabels en docs de detalle — delegated ✅ (writer; 8 regiones en 7 archivos): API.md §13 #1 `BR-013/BR-019` → `BR-008/ADR-010` (BR-019=notificaciones; auditoría movimientos=BR-008) + L73 semántica null/0 → OQ-042→BR-042 (derivado; sin total/truckId → null, no 0); DTOs.md L244 ídem null (no 0) → OQ-042/BR-042; JOBS.md L85/L220 nota residual OQ-019 → solo OQ-031→BR-051 (OQ-019=auditoría OPERATOR); ARCHITECTURE.md L245 nota mislabel OQ-013 → MASTER-SPEC §11.3 (OQ-013=prioridad de flujos; OQ-029=matriz, NO reports); USE-CASES.md L175/L525 y BUSINESS-RULES.md L557 nota "OQ-018 pendiente de verificación" → W1-Q5→OQ-030→BR-046 (OQ-018 solo catálogo ADR-009, contexto relacionado) — verificado por gatekeeper: readback de las 8 regiones + greps (0 OQ-019 en JOBS·0 OQ-013 en ARCHITECTURE·0 BR-013/BR-019 en API·0 "pendiente de verificaci" en USE-CASES/BUSINESS-RULES/API/DTOs)
- [x] T39b MODULES.md "columna backlog": NO VERIFICABLE — grep "backlog" 0 matches en MODULES.md y en todo el tree de docs; tablas inspeccionadas (§3/§4/§6/§10/§11) sin columna ni vínculo backlog/FEATURE; SIN edición (reportado con evidencia, no inventado)
- [x] T39c Corregir mislabel OQ-013 en DOD-D2 (DEFINITION-OF-DONE.md L178) y PHS-D5 (PHASES.md L217): OQ-013 = prioridad de flujos (NO certificación); no existe OQ de certificación → quitada la ref OQ-013 y el paréntesis erróneo; hilo "certificación externa vs interna" marcado 🔶 residual local (sin OQ) — inline (orquestador, 7ª sesión); verificado: grep `OQ-013` = 0 en docs de detalle (solo MASTER-SPEC/OPEN-QUESTIONS canónicos preservan la ref legítima)

## Definition of Done verificado
- 73 archivos del manifest existen y son legibles ✅
- Coherentes con MASTER-SPEC (IDs BR/entidades/enums/roles/fases/tokens); inconsistencias detectadas centralizadas en ID-001…010 ✅
- Ambigüedades marcadas como DECISIÓN PENDIENTE / OPEN QUESTIONS (OQ-001…047) — nunca inventadas ✅
- Cero archivos de código en cargoops-docs ✅
- No hubo implementación de producción ✅
- Ampliación 0.2 (secciones 62-70, M:N) aplicada a MASTER-SPEC + 34+12 docs; legacy capacity* solo en notas transicionales ✅
- **v0.5 (2026-09-24): NO quedan OQ abiertas** — las 30 OQ restantes resueltas; MASTER-SPEC v0.5 + OPEN-QUESTIONS + ADR-012/013 Accepted + ~30 docs alineados; residuales locales no bloqueantes (🔶) documentados para FASE 1 ✅
- **v0.5.1 (2026-09-24): residuales/mislabels FASE 0 corregidos** — API/DTOs/JOBS/ARCHITECTURE/USE-CASES/BUSINESS-RULES/DOD/PHASES alineados con canon (T39/T39c); MODULES "columna backlog" no verificable (reportado con evidencia, sin edición); canónicos intactos ✅

## Evidencia
- Manifest: `cargoops-docs/docs/MASTER-SPEC.md` §17
- Índice: `cargoops-docs/docs/README.md`
- Pendientes: `cargoops-docs/docs/OPEN-QUESTIONS.md` (OQ-001…047 **todas ✅** + ID-001…010 ✅)
- Inconsistencias a alinear en FASE 1: ID-001…010 (numeración FEATURE, permisos, GET/POST export-pdf, UUID, etc.)
- Residuales FASE 1: mislabels BR/OQ menores CORREGIDOS (2026-09-24, AMPLIACIÓN 0.5.1) incluidos DOD-D2/PHS-D5 (ref OQ-013 como certificación → residual local sin OQ); MODULES "columna backlog" no verificable (no existe en el doc); gap validación geométrica de mapa (OQ-015 no la cubrió → candidata OQ nueva); residuales 🔶 locales (DP-*, W1-Q*, BR-021…031 ratificación, paleta/logo, NgRx)

## Paso siguiente recomendado
**FASE 0 v0.5 cerrada** — **no quedan OQ abiertas**: las 30 OQ 🟡/🟢 resueltas (2026-09-24) + IDs alineadas; residuales/mislabels FASE 0 corregidos (v0.5.1). La documentación está lista como fuente de verdad para FASE 1. Próximo hito (decisión del usuario): **abrir FASE 1** — crear repos git (cargoops-backend, cargoops-frontend, cargoops-infrastructure), inicializar stacks según MASTER-SPEC v0.5 (Angular 20+ PWA mínima es-AR · NestJS + Prisma + PostgreSQL · Redis+BullMQ · MinIO · Puppeteer/Chromium), e implementar por fases del roadmap (FASE 1 Foundation primero). Residuales no bloqueantes a tener presentes al arrancar: hilo certificación de release (DOD-D2/PHS-D5, 🔶 sin OQ); MODULES "columna backlog" no verificable; gap validación geométrica de mapa (candidata OQ nueva); performance/RPO-RTO a validar con el negocio.