# SolCon Prompt (Copy/Paste)

Use this file to alter an existing web app so it works safely inside the DemoBrazeAIApp native container. This is finisher mode: patch existing code only. Preserve all existing user-facing functionality and current UI behavior. Only restructure identity/session sync and custom-event forwarding.

## Quick execution brief (do this first)
- Finisher mode only: do not scaffold or redesign the app architecture.
- Do not redesign UI unless explicitly requested.
- Keep existing product logic/features intact.
- Use one shared DemoBridge contract (no app-specific bridge variants).
- Make the smallest possible code diff to pass all acceptance checks.

## Execution order (follow exactly)
1) Inspect current code paths for user/session/event handling.
2) Centralize bridge calls into one module (`demo_bridge_entry.js`).
3) Route both web UI and native callbacks through one reducer/state writer (`applyIncomingSync` pattern).
4) Enforce dedupe, echo suppression, manual lock window, and touch idempotency.
5) Centralize custom events in one `trackEvent(name, properties)` helper.
6) Validate against all acceptance checks and report results.
7) If any hardened file changed intentionally, regenerate integrity manifest hashes before final handoff.

## File placement rules (required)
- Copy `solcon-finisher-projects-main/demo_bridge_entry.js` into the target app codebase (for example `src/bridge/demo_bridge_entry.js`) and rewire app code to use it.
- Copy `solcon-finisher-projects-main/sync_state_reference.js` into the target app codebase (for example `src/bridge/sync_state_reference.js`) and use it as the reducer/state pattern reference.
- If these files already exist in the target app, compare against the canonical versions and make only minimal patches needed for parity.
- Do not re-invent or rename DemoBridge methods; preserve the canonical contract shape exactly.
- Replace existing file content when patching; never append duplicate file blocks.

## Must-use references
- `solcon-finisher-projects-main/SOLCON_PROMPT_PROJECTS.md` (this file)
- `solcon-finisher-projects-main/README.md`
- `solcon-finisher-projects-main/VALIDATION.md`
- `solcon-finisher-projects-main/sync_state_reference.js`
- `solcon-finisher-projects-main/demo_bridge_entry.js`

## Prompt to give SolCon/AI builder
Use `solcon-finisher-projects-main/SOLCON_PROMPT_PROJECTS.md` and the files in its "Must-use references" section to alter this existing codebase for native app-container compatibility while preserving all currently working user-facing functionality. Do not redesign UI unless explicitly requested. Apply only the identity/session sync and custom-event forwarding rules below, then return files changed, rationale per change, and validation results.

## Non-negotiables
- Focus on stable identity/session sync + event forwarding only.
- One bridge module only (`demo_bridge_entry.js`) as the sole place that calls DemoBridge.
- Do not edit `demo_bridge_entry.js` (contract file) unless explicitly requested. Rewire app code to use it as-is. If a change seems required, pause and request approval with a minimal patch.
- One state writer/reducer (`applyIncomingSync`) for both web and native-origin updates.
- One custom-event helper (`trackEvent`) for all UI events.
- Keep `demo_bridge_entry.js` identity-only. Do not add any additional helpers to that file.
- Do not drop native sync metadata; preserve and forward `detail` payload through the same reducer path.
- In native/container mode, avoid duplicate identity writes: do not execute direct `braze.changeUser/openSession` in the same branch that calls `DemoBridge.startSession(...)` if the container bridge already applies identity.

## Canonical DemoBridge contract (container mode)
- `window.DemoBridge.startSession({ userId, configId, reason })`
- `window.DemoBridge.setConfigId(configId)`
- `window.DemoBridge.initNativeListener(callback)`
- `window.DemoBridge.logCustomEvent(name, properties)`

## Identity/session requirements
- On first paint/config load: call `startWebSession({ userId, configId })` with `reason="default"`.
- On each web user switch: call `setUser(newUserId, "manual")`.
- Register native listener exactly once; route native updates through the same reducer.
- Native listener callback must use `(incomingUserId, detail)` and pass `detail` through unchanged to reducer logic.
- Preserve native `detail` fields end-to-end: `reason`, `authority`, `sessionId`, `configId` (default only when truly absent).
- Keep session start single-fire (no idle handshake/session spam).
- Dedupe repeated payloads using a signature like `userId|sessionId|authority|reason`.
- Suppress native echo loops (native-origin apply must not re-send identical outbound update).
- Add manual lock window exactly `300ms`: a recent manual switch cannot be overridden by `default`.
- Normalize IDs with `trim` only. Preserve exact userId casing when applying identity and forwarding payloads.
- All identity writes must be single-owner: `DemoBridge.startSession`/`DemoBridge.setUser` and `braze.changeUser`/`braze.openSession` must be invoked only through the sync-owned `setUser` path.
- Make touch handlers idempotent (one apply per tap).

## Braze handling in finisher mode
- If Braze Web SDK is already present, keep it and centralize `braze.changeUser` and `braze.openSession` (same places as `startWebSession`/`setUser`).
- If Braze Web SDK is not present, do not add it unless explicitly requested.
- Route all custom events through `trackEvent(name, properties)`.
- Default mode (recommended for DemoBrazeAIApp container): call `braze.logCustomEvent(...)` only. The native bridge utility hooks Braze custom events and forwards them to native Event Log.
- Do not edit `demo_bridge_entry.js` to add custom-event forwarding.
- Do not dual-write custom events to bridge + Braze unless explicitly requested. If explicit dual-write is required, add dedupe to prevent duplicate native Event Log entries.

Use this default event helper pattern:

```js
function trackEvent(name, properties = {}) {
  const payload = { ...properties, source: "web" };
  if (window.braze?.logCustomEvent) {
    window.braze.logCustomEvent(name, payload);
  }
}
```

## Browser fallback requirement
- In browser-only mode (no DemoBridge), app must not crash.
- Guard bridge calls with optional checks and no-op cleanly.
- Guard direct Braze identity writes too (safe no-op when SDK unavailable/uninitialized).

## Preview embedding support (required)
- Demo preview flows may be opened inside an iframe from this parent origin:
  - `https://doppel-dashboard-staging-a7496acff9c6.herokuapp.com/`
- Allow embedding from that exact parent origin on preview-capable routes.
- Prefer CSP `frame-ancestors` allowlisting over broad global header changes.
- Do not disable iframe protections globally across unrelated routes.
- If hosting/deployment protection injects iframe blocking (`401/403`, `X-Frame-Options: DENY`, restrictive CSP), call out that platform-level setting must be changed.

## Logging/debug requirements
- Keep logs centralized (not scattered in components).
- Use prefixes:
  - `[Bridge]` for web/native sync
  - `[Braze]` for Braze SDK actions
- Include: `userId`, `configId`, `sessionId`, `reason`, `authority` (when available).

## Acceptance criteria (must pass)
1) Idle 30-60s: no repeated `[WEB] Session Started`/handshake spam.
2) Web manual switch once: one sync flow, no `n2 -> n1` bounceback.
3) Native manual switch once: web updates once, no duplicate apply.
4) Web custom event once: appears in Braze web path through `trackEvent -> braze.logCustomEvent` only (no direct DemoBridge event forwarding from app code).
5) iPhone touch interaction: one apply per tap.
6) Browser-only mode: no runtime errors from missing DemoBridge.
7) Native metadata integrity: reducer/log path contains `reason`, `authority`, `sessionId`, `configId` from native `detail` payload.
8) Contract surface check: no direct `window.DemoBridge` calls outside `demo_bridge_entry.js`.

## Delivery format
- List files changed.
- Explain why each change prevents loops/bounce/duplication.
- Provide quick test steps and observed results.
- If hardened files changed, state that integrity manifest was regenerated and include the command used.
- Keep implementation simple; no unnecessary abstraction layers.
