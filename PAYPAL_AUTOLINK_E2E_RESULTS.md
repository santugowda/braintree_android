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

## Case 4 — Auto-link failure (BTGW rejects) ✅ PASS

**Reproduction note:** the sandbox BTGW tokenizes the standalone `billing_agreement_token`
**leniently** — it returns a nonce even for an unapproved or corrupted `ba_token` (attempts to
force a real `BILLING_AGREEMENT_NOT_APPROVED` by not approving, and by corrupting the token, both
still returned SUCCESS). The real 422 only occurs later at **transaction creation**, not at
tokenize time. To validate the SDK's failure handling deterministically, a `[QA] Force auto-link
failure` toggle makes `attemptAutoLinkTokenization` throw a simulated exception in place of the
BTGW call.

### Case 4a — Path A failure (`handleReturnToApp` → tokenize)

**Steps:** App Switch ON, Skip OFF, Force-failure ON, App Link disabled → tap BA → return manually.

**Actual logs**
```
handleReturnToApp: NoResult + valid pending session (baToken=BA-44W71636H8283143K) -> routing to auto-link (autoLinkPending)
tokenize: autoLinkPending -> tokenizeAutoLink()
attemptAutoLink: INITIATOR calling BTGW (baToken=BA-44W71636H8283143K)
attemptAutoLink: BTGW FAILED -> QA forced auto-link failure (simulated BILLING_AGREEMENT_NOT_APPROVED)
tokenizeAutoLink: FAILURE -> QA forced auto-link failure (simulated BILLING_AGREEMENT_NOT_APPROVED)
```
**Result:** PASS. Error propagates to `PayPalResult.Failure`; the Demo surfaces it via `handleError`.

### Case 4b — Path B failure (re-click fall-through)

**Steps:** App Switch ON, Skip ON, Force-failure ON → tap BA → return → tap BA again.

**Actual logs**
```
Demo onResume: SKIP handleReturnToApp (re-click test mode) — session preserved
re-click: STARTED (baToken=BA-0A08708461289983N)
attemptAutoLink: INITIATOR calling BTGW (baToken=BA-0A08708461289983N)
attemptAutoLink: BTGW FAILED -> QA forced auto-link failure (simulated BILLING_AGREEMENT_NOT_APPROVED)
re-click: FAILED (...) -> falling through to normal flow
stored pending session for app-switch vault (baToken=BA-4YE82970PR612163X)   ← fresh flow launched
```
**Result:** PASS. On re-click failure no error is surfaced; the SDK falls through to a fresh
`createPaymentAuthRequest` (new session stored) — as designed.

---

## Summary

| Case | Scenario | Result |
|------|----------|--------|
| 1 | Path A — `handleReturnToApp` NoResult → auto-link (manual return) | ✅ PASS |
| 2 | URL return wins — auto-link suppressed, store cleared (App Link works) | ✅ PASS |
| 3 | Path B — re-click delivers nonce via `tokenizeCallback` | ✅ PASS |
| 4a | Path A failure — BTGW error → `PayPalResult.Failure` (error shown) | ✅ PASS |
| 4b | Path B failure — re-click falls through to a fresh flow | ✅ PASS |
| 5 | Charge nonce (CREATE A TRANSACTION) | ⚠️ N/A — Demo/gateway limitation (fails for normal nonces too) |

Both PR 1 auto-link entry points, the URL-return-wins guard, and both failure paths are validated
end-to-end on a real device against the PayPal sandbox. Charging the nonce could not be validated
through the Demo (see Case 5). A separate finding: the auto-link nonce carries no payer details.

## Key finding — sandbox tokenizes leniently

The auto-link `billing_agreement_token` tokenization **succeeds in sandbox even for an unapproved or
corrupted `ba_token`** (verified: not approving → success; corrupting the token → success). The
`BILLING_AGREEMENT_NOT_APPROVED` 422 surfaces only at **transaction creation**, downstream of the
SDK's tokenize step. Failure handling was therefore validated via a client-side simulated throw
(`[QA] Force auto-link failure`) and by unit tests that stub the BTGW error.

> Update: even with the BTGW `BILLING_AGREEMENT_NOT_APPROVED` change deployed in sandbox, we could
> not reproduce a real 422 at tokenize time — returning buyers are auto-approved, and a fresh
> never-consented buyer + corrupted token both still returned SUCCESS. So the auto-link
> `billing_agreement_token` tokenization does not appear to be gated by that change at tokenize time
> in sandbox. This is a **BTGW-server** question, not an SDK one — worth confirming with the BTGW team
> whether the standalone `billing_agreement_token` path is covered.

## Case 5 — Charging the nonce (CREATE A TRANSACTION) — Demo/gateway limitation, not a PR issue

We attempted to charge the resulting nonce via the Demo's **CREATE A TRANSACTION** button to confirm
it is usable. Every attempt failed with `Transaction Failed: Unknown or expired payment_method_nonce`
— **including a control run using a normal, non-auto-link nonce**.

**Control test (decisive):** App Switch **OFF** → normal browser Billing Agreement → nonce with
**full payer details** (First/Last name, email, Payer ID `HBA5XYWLXHUCU`, billing address) →
**CREATE A TRANSACTION** → same `Unknown or expired payment_method_nonce`.

**Conclusion:** the charge failure reproduces with a completely normal PayPal BA nonce, so it is a
**Demo / sample-merchant-server / gateway limitation** for PayPal billing-agreement charges in this
environment — **not related to the auto-link feature**. Charging cannot be validated through this
Demo; a real charge test needs a proper merchant server or the BTGW/PayPal team.

## Known limitation — auto-link nonce has no payer details

Comparing the two nonce screens:

| Field | Normal (URL-return) nonce | Auto-link nonce |
|-------|---------------------------|-----------------|
| First / Last name | `Charlie Williams` | *(empty)* |
| Email | `charlie-williams@paypal.com` | `null` |
| Payer ID | `HBA5XYWLXHUCU` | *(empty)* |
| Billing address | `1 Main St, San Jose CA 95131 US` | `null` |

The auto-link `billing_agreement_token` path returns a nonce **without enriched payer details**,
whereas the normal web-URL path populates them. Merchants who read `PayPalAccountNonce` fields
(email / payerId / name / address) will get **nulls from the auto-link path**. Worth flagging to
product/reviewers, and to BTGW (does the standalone `billing_agreement_token` tokenization return
payer details?).

## Not yet exercised

- **Session expiry (TTL 30 min)** — return after TTL → `attemptAutoLink: session EXPIRED` /
  `AUTO_LINK_EXPIRED`. (Covered by unit tests; on-device would need a TTL override or a 30-min wait.)
- **Deduplication** — concurrent triggers → single BTGW call
  (`attemptAutoLink: awaiting in-flight BTGW call (dedup)`). (Covered by unit tests; hard to race manually.)

## Notes

- Debug `Log.d` traces (tag `PayPalAutoLink`), the `[QA] Skip handleReturnToApp` Settings toggle,
  and the `onResume` skip branch are **temporary** QA scaffolding, marked `TODO: remove before merge`
  and left **uncommitted** so PR 1 stays clean.
- The Demo module does not compile via local Gradle (JDK-21 toolchain quirk); builds were run from
  Android Studio.
