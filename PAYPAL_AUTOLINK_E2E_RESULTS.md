# PayPal Auto-Link (PR 1) — E2E Test Results

Manual end-to-end validation of **PR 1** (`paypal-autolink-handle-return`): auto-link on manual
return via `handleReturnToApp` NoResult + re-click. Results captured from device logcat.

## Environment

| Item | Value |
|------|-------|
| Branch | `paypal-autolink-handle-return` |
| App | Demo (`:Demo`) |
| Device | `R5CY83XB7QH` (physical) |
| Flow | PayPal **Billing Agreement** (vault) |
| Settings | **PayPal App Switch** ON |
| Instrumentation | Temporary `Log.d(tag="PayPalAutoLink", ...)` traces + a `[QA] Skip handleReturnToApp` Demo toggle (both marked `TODO: remove before merge`, uncommitted) |
| Log capture | `adb logcat -d -s PayPalAutoLink` |

> To force a **manual return** (no App Link), disable the buyer's "Open supported links" for PayPal
> in Android settings. The `[QA] Skip handleReturnToApp` toggle keeps the pending session alive so
> the re-click path is reachable in the Demo (whose `onResume` otherwise consumes it via Path A).

---

## Case 1 — Path A: `handleReturnToApp` NoResult → auto-link ✅ PASS

**Steps**
1. App Switch ON, App Link disabled.
2. Tap **Billing Agreement** → app switches to PayPal.
3. **Approve** the billing agreement.
4. **Manually** return to the Demo (no deep link).
5. `onResume` → `handleReturnToApp`.

**Actual logs**
```
stored pending session for app-switch vault (baToken=BA-3U606475MX199930J)
handleReturnToApp: started, intentData=null
handleReturnToApp: NoResult + valid pending session (baToken=BA-3U606475MX199930J) -> routing to auto-link (autoLinkPending)
tokenize: autoLinkPending -> tokenizeAutoLink()
attemptAutoLink: INITIATOR calling BTGW (baToken=BA-3U606475MX199930J)
attemptAutoLink: BTGW SUCCESS, nonce=46998495-46ed-13ba-73e6-06b66a067a18
tokenizeAutoLink: SUCCESS -> PayPalResult.Success
```

**Result:** PASS. `intentData=null` confirms a genuine manual return with no App Link; the SDK
resolved a real nonce directly via BTGW. Nonce displayed on the DisplayNonce screen.

---

## Case 2 — URL return wins (regression) ✅ PASS

**Steps**
1. App Link **working** (buyer returns via App Link).
2. Tap **Billing Agreement** → app switch → return via App Link.

**Actual logs (run a — cancel URL)**
```
stored pending session for app-switch vault (baToken=BA-2S789630EX3970636)
handleReturnToApp: started, intentData=https://mobile-sdk-demo-site-.../braintree-payments/cancel?token=EC-9J685002AT0590737&ba_token=BA-2S789630EX3970636&switch_initiated_time=...
handleReturnToApp: URL return SUCCESS (auto-link suppressed, store cleared)
```

**Actual logs (run b — success URL)**
```
stored pending session for app-switch vault (baToken=BA-45U39892042928327)
handleReturnToApp: started, intentData=https://mobile-sdk-demo-site-.../braintree-payments/success?token=EC-9D175271D3602570E&ba_token=BA-45U39892042928327&PayerID=M37G3U8K6G2QN&switch_initiated_time=...
handleReturnToApp: URL return SUCCESS (auto-link suppressed, store cleared)
```

**Result:** PASS. When a real URL return arrives, `completeRequest` returns `Success`; auto-link is
**suppressed** and the pending session is **cleared**. No `auto-link:*` traces — the URL path is
authoritative.

---

## Case 3 — Path B: re-click via `tokenizeCallback` ✅ PASS

**Steps**
1. App Switch ON, **[QA] Skip handleReturnToApp** ON.
2. Tap **Billing Agreement** → app switch → **approve** BA → return.
   - Skip toggle preserves the pending session (no Path A consumption).
3. Tap **Billing Agreement again** (the re-click).

**Actual logs**
```
re-click: no pending session -> normal flow                                  (1st tap: nothing pending)
stored pending session for app-switch vault (baToken=BA-48F67264AG353390B)   (app switch launched)
Demo onResume: SKIP handleReturnToApp (re-click test mode) — session preserved
re-click: STARTED (baToken=BA-48F67264AG353390B)                             (2nd tap)
attemptAutoLink: INITIATOR calling BTGW (baToken=BA-48F67264AG353390B)
attemptAutoLink: BTGW SUCCESS, nonce=46e66eaf-3d68-12f3-4340-ac04d9aa12ef
re-click: SUCCESS, delivering nonce via tokenizeCallback
```

**Result:** PASS. The same `baToken` is threaded end-to-end, a single BTGW call is made, and the
nonce is delivered via `tokenizeCallback` — the full flow is not restarted.

---

## Summary

| Case | Scenario | Result |
|------|----------|--------|
| 1 | Path A — `handleReturnToApp` NoResult → auto-link (manual return) | ✅ PASS |
| 2 | URL return wins — auto-link suppressed, store cleared (App Link works) | ✅ PASS |
| 3 | Path B — re-click delivers nonce via `tokenizeCallback` | ✅ PASS |

Both PR 1 auto-link entry points and the URL-return-wins guard are validated end-to-end on a real
device against the PayPal sandbox.

## Not yet exercised

- **422 `BILLING_AGREEMENT_NOT_APPROVED`** — start BA flow, do **not** approve in PayPal, return, then
  trigger auto-link. Expected: `attemptAutoLink: BTGW FAILED` → Path A surfaces `PayPalResult.Failure`;
  re-click (Path B) logs `re-click: FAILED -> falling through to normal flow`.
- **Session expiry (TTL 30 min)** — return after TTL → `attemptAutoLink: session EXPIRED` / `AUTO_LINK_EXPIRED`.
- **Deduplication** — concurrent triggers → single BTGW call (`attemptAutoLink: awaiting in-flight BTGW call (dedup)`).

## Notes

- Debug `Log.d` traces (tag `PayPalAutoLink`), the `[QA] Skip handleReturnToApp` Settings toggle,
  and the `onResume` skip branch are **temporary** QA scaffolding, marked `TODO: remove before merge`
  and left **uncommitted** so PR 1 stays clean.
- The Demo module does not compile via local Gradle (JDK-21 toolchain quirk); builds were run from
  Android Studio.
