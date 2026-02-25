## SolCon Finisher

Use this as the standalone SolCon sync/event contract. Keep existing product functionality and UI, but restructure identity/session/event plumbing so the web app works safely inside the native container.

### What this fixes
- Session-start handshake loops.
- `n2 -> n1` bouncebacks after manual switch.
- Duplicate apply/sync from multiple listeners or touch double-fire.
- Missing native Event Log forwarding for web custom events.

### Minimal finish rules
- Keep one identity entrypoint (`startWebSession`, `setUser`, `listenForNative`).
- Call initial session start once on load/config select.
- Register native listener once.
- Route all user changes (UI + native) through one apply/reducer path.
- Add dedupe + native echo suppression.
- Add short manual lock window (2-5s) so `default` cannot override a recent `manual` change.
- Route all custom events through one `trackEvent(name, properties)` helper.
- Keep `demo_bridge_entry.js` identity-only; do not add any additional helpers there.
- Default custom-event path: `trackEvent` calls `braze.logCustomEvent(...)`; native Event Log visibility comes from the container's Braze custom-event hook.
- Use explicit bridge + Braze dual-write only when no native hook exists, and add dedupe to avoid duplicate native Event Log entries.

### Canonical DemoBridge contract (shared for all SolCons)
Do not invent custom bridge names or payload shapes per app. Use this contract for every generated site.

Required methods (when running in native WebView):
- `window.DemoBridge.startSession({ userId, configId, reason })`
- `window.DemoBridge.setConfigId(configId)`
- `window.DemoBridge.initNativeListener(callback)`
- `window.DemoBridge.logCustomEvent(name, properties)`

Required sync payload fields:
- `type: "syncUser"`
- `protocol: "user-sync/v1"`
- `sessionId`, `configId`, `userId`, `authority`, `reason`, `timestamp`

Browser fallback rule:
- If `window.DemoBridge` is missing (normal browser dev), app must not crash.
- Guard bridge calls (`if (window.DemoBridge?.method)`) and continue with local UI + Braze web behavior.

### Required centralized event helper
All web custom events must route through a single helper:

```js
function trackEvent(name, properties = {}) {
  const payload = { ...properties, source: "web" };
  if (window.braze?.logCustomEvent) {
    window.braze.logCustomEvent(name, payload);
  }
}
```

### Keep it simple
- No extra sync buses/wrapper layers unless explicitly requested.
- Reuse `solcon-finisher/sync_state_reference.js` instead of inventing new sync rules.

