# Engram OpenCode V2 Migration

## Objective
Restore the global Engram OpenCode plugin under OpenCode V2 without changing its persistence behavior or exposing credentials.

## Problem
`C:\Users\juan-\.config\opencode\plugins\engram.ts` is a legacy V1 plugin. OpenCode V2 rejects it during module validation because it has no default V2 plugin definition, and its implementation still uses V1 imports, context properties, events, and hooks.

## Scope
- Port the plugin entrypoint to `Plugin.define({ id, setup })`.
- Preserve Engram HTTP calls, project/session lifecycle handling, prompt/tool capture, memory instruction injection, passive capture, and cleanup.
- Verify TypeScript/load behavior and run focused smoke checks.

## Out of scope
- Changing Engram server credentials or environment variables.
- Changing unrelated global plugins or project configuration.
- Reporting an upstream defect.

## Tasks
- [x] T1: Map V1 behavior to exact V2 APIs and event payloads.
- [x] T2: Port `engram.ts` with stable ID, V2 lifecycle, hooks, event subscription, and cleanup.
- [x] T3: Run type/load/smoke verification and inspect the resulting log/plugin list.
- [x] T4: Record verification evidence and final state.

## Acceptance criteria
- OpenCode V2 loads the plugin without `PluginModule.LoadError`.
- The plugin has a stable ID and a V2 default export.
- Session ownership, attributed writes, passive capture, memory instructions, and nudge behavior remain wired to valid V2 APIs.
- Cleanup aborts the event stream and closes attempted Engram sessions.
- No credential values are printed or persisted by the migration.

## Constraints and decisions
- Implementation route: delegated direct is required for the non-trivial port; the available exploration subagent was blocked by the provider, so the parent is using the exact local V2 type contracts and official documentation.
- TDD mode: not configured for this global plugin; use focused type/runtime verification.
- The workspace root is not a Git repository, so a feature-branch commit cannot be created here. Preserve the file-level rollback boundary instead.
- The Engram mirror is pending because the workspace contains two repositories and Engram requires an explicit project selection.

## Verification
- Initial `npx tsc --noEmit` check was unavailable because TypeScript is not installed in the global OpenCode package; no dependency was added. The cached OpenCode TypeScript compiler was then used for the authoritative strict check below.
- `node <cached-typescript>/tsc.js --noEmit --strict --skipLibCheck --target ES2022 --module ESNext --moduleResolution Bundler --esModuleInterop --allowSyntheticDefaultImports --types node C:\\Users\\juan-\\.config\\opencode\\plugins\\engram.ts` — passed with exit code 0.
- Static contract scan — passed: V2 default export present; no V1 import, legacy context fields, legacy hooks, or `event.properties` references.
- `opencode service status` — passed: `http://127.0.0.1:49374`.
- `opencode plugin list` (from `C:\\Users\\juan-\\Desktop\\IA\\Track`) — passed; `gentle-ai.engram` is listed from the global plugin path.
- Fresh server log (`run=b7446987`) — passed; `engram.ts` loaded and its watcher started, with no subsequent `PluginModule.LoadError` or `SchemaError`.
- `http://127.0.0.1:7437/health` — passed with HTTP 200 and Engram service version 2.0.0.
- Hook/event coverage — statically verified from the V2 type contracts: `ctx.session.hook` registers prompt/context/compaction; `ctx.tool.hook` registers execute.before/after; `ctx.event.subscribe` uses an abort signal; cleanup aborts, awaits the stream, disposes hooks, and closes registration attempts.
- Locationless lifecycle events — fixed so optional V2 event locations do not suppress `session.deleted` cleanup.

## Progress
- T1: API mapping completed from the V2 type declarations and official migration guide.
- T2: Port completed in `C:\\Users\\juan-\\.config\\opencode\\plugins\\engram.ts`; stable ID is `gentle-ai.engram`.
- T3: TypeScript, plugin-list, service-status, fresh-log, and Engram-health checks passed.
- T4: Final state recorded here. No Git commit was possible because the workspace root is not a Git repository; rollback boundary is the single global plugin file.
