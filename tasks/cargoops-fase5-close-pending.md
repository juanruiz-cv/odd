# FASE 5 close-pending — alta directa a sector + BR-036 sobreocupación

> ODD feature document — cargoops-backend. Repo-relative locator: `odd/tasks/cargoops-fase5-close-pending.md` (tracking repo `odd`).
> Engram mirror topic: `odd/fase5-close-pending/tasks` (project `cargoops-backend`).

## Objective

Close the two deferred functional gaps of FASE 5 (Movements) that the phase left open, in order:

1. **GAP A — Alta directa a sector** (`REGISTERED → STORED`, D-64/D-71/D-85): the confirmed transition
   "alta sin movimiento" (VALIDATION.md §4.4.1 L112, OQ-029 → BR-045) is structurally unreachable:
   `POST /cargos` creates REGISTERED without location, no intake DTO carries a location, and the state
   machine edge has `kinds: []` so `move()` rejects it 422. Close it by making the ALTA itself able to
   set the initial state `STORED` with an initial ACTIVE `CargoLocation` — a no-movement intake, exactly
   what ARCHITECTURE.md L177 already describes ("si el alta incluye ubicación inicial … insert del segmento
   inicial CargoLocation (status ACTIVE; BR-032)").
2. **GAP B — BR-036 overoccupation enforce** (OQ-043 resolved 2026-09-24): `allowOverOccupation` +
   `overOccupationLimitPercent` (default **+10%**) are governance flags on `Location` (set by ADMIN via
   `locations.manage`, audited `CAPACITY_CHANGE`, asserted by ESC-010 step 3) but the movements write
   paths never consult them — every refusal is at 100%. Enforce the ceiling in `assertCapacityFits`
   (persistent flag model per ESC-010 step 4: the distribution is executed by an OPERATOR; only enabling
   the flag is ADMIN-only), auditing the exception.

## Why

- D-71/D-85 explicitly defer "resolver alta directa a sector"; closing it removes an unreachable
  confirmed transition and an orphan use case (FA-2, USE-CASES.md L114).
- BR-036 is documented as ready everywhere (MASTER-SPEC §6 L204, OQ-043) and QA contracts exist and are
  red against the code (AC-060, TC-068, ESC-010). The docs promise capability the backend refuses.
- User instruction 2026-09-30: finish phase pendings in order, then ask again about next phase.

## Scope

- Backend `cargoops-backend`:
  - GAP A: `CreateCargoDto` + optional `locationId` (mutually exclusive with `truckId` — an intake is
    either direct-to-sector or via truck); `CargoService.create` branch: validate location ACTIVE
    (BR-004), BR-044/BR-035/BR-036, insert Cargo `STORED` + initial ACTIVE `CargoLocation` in the same
    transaction; **no Movement row** (alta sin movimiento); state-machine comment updated (edge stays
    `kinds: []` — it is an ALTA, not a movement; `STATUS_LOCATION_TYPES.REGISTERED = []` unchanged).
  - GAP B: widen `DESTINATION_SELECT` + `assertCapacityFits` param type with the two BR-036 fields;
    ceiling = `capacity * (1 + overOccupationLimitPercent/100)` when `allowOverOccupation = true`,
    else 100%; distinct exception detail for over-limit; **audit `CAPACITY_CHANGE` exception-type with
    `metadata.overOccupation`** whenever a write is accepted above 100% capacity; update class JSDoc.
  - Tests: state-machine spec L48/L507-547 (intake edge semantics), movement.service.spec L1569-1590
    (invert BR-036 test), new intake coverage, e2e (cargos intake; movements ESC-010/TC-068 patterns).
- Docs sync in `cargoops-docs` (D-64 reconciliation):
  - `DTOs.md` §4.4/§5.1: "NO existe locationId único en el alta" → becomes "opcional: alta directa a sector".
  - `API.md` §5.2: service the `locationId` error contract (already documents 404/422) and add the
    intake branch; OPENAPI entry.
  - `ARCHITECTURE.md` L177: now describes real behavior (no edit needed beyond status check).
  - `DATABASE.md` L417: enforcement lives in `movements` service, not `locations`.
  - `VALIDATION.md` §4.4.1 L112 stays (now implemented); §8 pendiente #10 (L266) → closed.
  - `USE-CASES.md` FA-2 (L114): add the actual API contract (POST /cargos with locationId).
  - `PHASES.md` L219 (PHS-D7): rule no longer undefined.
- NOT included: new permission code (reuse `location.manage` for the flag — already implemented);
  request-time authorization model; PERCENT-based intake quantity; conversion (BR-048); auto-movements.

## Constraints

- Contract is authoritative: VALIDATION §4.4.1 / §4.5 + ESC-010/AC-060/TC-068 + OQ-043 (all read above).
- No `$queryRaw` (Prisma 7.10, D-78): `$executeRaw` only for advisory locks.
- Audit outside the transaction (repo precedent D-70/D-84).
- Unit specs are pure mocks; e2e needs CI (no local Postgres, D-68).
- A no-movement intake must NOT create a Movement row (BR-008 history stays reconstruction-safe: the
  initial state is part of the Cargo, not a transition).
- `capacity = 0` = UNBOUNDED (skip capacity check) — unchanged.
- Idempotency-Key stays accepted-and-ignored (D-86).

## Delivery strategy (cached from FASE 5)

- Chained PRs **stacked-to-main** (strategy cached in session), each targeting `main`, merged in order,
  merge = user decision (ordinary repo policy).
- Slice 1 (`feat/f5-pending-intake`): GAP A. Slice 2 (`feat/f5-pending-overoccupation`): GAP B.
- Doc sync in `cargoops-docs`: commits directos a `main` (repo convention).
- Native RDD unavailable in this runtime → assess registers `unavailable`, never invented PASS;
  verification = functional checks (unit + lint + build local, CI for e2e) + parent spot check.

## Tasks

- [ ] **T1 — GAP A alta directa a sector (intake con locationId)** — `CreateCargoDto.locationId?`
      exclusive with `truckId`; `CargoService.create` → STORED + initial ACTIVE CargoLocation (BR-032),
      validations BR-004/035/036 in-tx, no Movement row; state-machine comment update; unit + e2e specs.
      Acceptance: `POST /cargos` with `locationId` → 201 cargo `STORED` + 1 segment ACTIVE; without →
      behavior unchanged (REGISTERED/IN_TRUCK); invalid location 404 (BR-003); inactive 409; capacity
      refused over ceiling; no Movement row created; spec state-machine L507-547 adjusted to new semantics.
- [ ] **T2 — GAP B BR-036 enforce** — `DESTINATION_SELECT` + `assertCapacityFits` type widened;
      ceiling per flag/limit; distinct over-limit detail; accepted-above-100% writes audited
      `CAPACITY_CHANGE` exception with metadata; JSDoc updated; movement.service.spec L1569-1590 inverted
      + flag matrix; e2e ESC-010/TC-068 pattern.
      Acceptance: flag off → 409 at 100% (unchanged); flag on → allowed up to +10%, refused above;
      accepted over-capacity write audits exception; unbounded (capacity 0) unchanged.
- [ ] **T3 — Docs sync (D-64 reconciliation)** — DTOs.md §4.4/§5.1, API.md §5.2, OPENAPI, VALIDATION
      §8 #10, USE-CASES FA-2, DATABASE.md L417, PHASES L219. Acceptance: no doc claims a contract the
      code refuses; FA-2 has a real endpoint; D-64 closed in feature doc.

## Route declaration

- T1/T2: delegated writer (2+ non-trivial files each). T3: delegated writer (multi-file docs sync).
- Trigger evidence: mapping trigger (backend module + 4+ files + docs, satisfied by explore worker
  2026-09-30); writer trigger (each task touches 2+ non-trivial files).
- Orchestrator gatekeeper: artifact readback + spot-check after each writer, per ODD.