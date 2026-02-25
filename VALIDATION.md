## SolCon Finisher Validation

1) Idle
- Open Demo and wait 30-60s.
- Pass: no session-start spam loop.

2) Web switch
- Change user once in web UI.
- Pass: one sync flow, no `n2 -> n1` bounce.

3) Native switch
- Change user once from native.
- Pass: web updates once, no duplicate apply.
- Pass: native callback path uses `(incomingUserId, detail)` and forwards `detail` unchanged.
- Pass: logs/reducer path retain `reason`, `authority`, `sessionId`, `configId` from native payload.

4) Custom event
- Trigger one web custom event.
- Pass: appears in Braze Web SDK path.
- Pass: appears in native Event Log via container custom-event hook (or explicit bridge fallback when hook is unavailable).
- Pass: single native Event Log entry per custom event (no duplicate forwarding).

5) iPhone touch check
- Tap user switch on device.
- Pass: one apply per tap.

6) Browser fallback
- Open site in a normal browser (no native container bridge).
- Pass: no runtime crash from missing `window.DemoBridge`.
- Pass: guarded bridge calls no-op cleanly.

7) Contract surface check
- Search for `window.DemoBridge` usages.
- Pass: only `demo_bridge_entry.js` calls DemoBridge directly.
- Pass: `demo_bridge_entry.js` remains identity-only with no additional helper exports.

8) File integrity check
- Open each modified file once after patching.
- Pass: file contains one implementation block only (no accidental concatenated duplicates).

## If it fails
- Duplicate `startWebSession` calls.
- Native listener registered more than once.
- Multiple user state writers outside one apply path.
- No dedupe or no manual lock window.
- No native echo suppression.
- Custom events not centralized in `trackEvent`.
- `trackEvent` skips Braze SDK logging.
- Additional helpers were added to `demo_bridge_entry.js`.
- Duplicate native Event Log entries caused by hook-forwarding plus explicit bridge dual-write without dedupe.
- App-specific bridge method names diverge from canonical `DemoBridge` contract.
- Native `detail` metadata dropped before reducer (`reason/authority/sessionId/configId` missing).
- Direct `window.DemoBridge` calls scattered outside `demo_bridge_entry.js`.
- Full file content appended repeatedly instead of replaced.

