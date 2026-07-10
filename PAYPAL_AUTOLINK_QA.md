# PayPal Auto-Link on Manual Return — Manual QA

Manual test plan for the **auto-link on manual return** feature (PRs #1633 + #1634).

## What the feature does

When a PayPal **app-switch vault** (billing agreement) flow completes but the **App Link return fails**
(the buyer unchecked "Open supported links", so PayPal can't deep-link back), the buyer manually
switches back to the merchant app. Instead of a dead end, the SDK tokenizes the stored billing
agreement token **directly with BTGW** using a standalone `billing_agreement_token` — no URL return
required.

### Three entry points

| # | Trigger | Integration style | Delivered in | PR |
|---|---------|-------------------|--------------|-----|
| A | `handleReturnToApp` returns `NoResult` while a valid session is pending | `onResume` (default) | `tokenize()` → `PayPalResult.Success` | #1633 |
| B | Buyer **re-taps** PayPal (`createPaymentAuthRequest(..., tokenizeCallback)`) | any | `tokenizeCallback` | #1633 |
| C | App returns to foreground (`ProcessLifecycleOwner` → `AppForegroundDetector`) | `onNewIntent` / `singleTop` (no new intent on manual return) | pre-resolves nonce, picked up by A or B | #1634 |

**A real URL return always wins.** If the App Link works, `handleReturnToApp` gets `Success`, the store
is cleared, and auto-link is suppressed — no timer involved.

---

## Prerequisites

1. **Demo app** (`:Demo`) installed on a device/emulator.
2. A **PayPal sandbox buyer** account.
3. In the Demo **Settings**:
   - ✅ **PayPal App Switch** (`paypal_app_switch`) — enables `setEnablePayPalAppSwitch(true)`.
   - Use a **Billing Agreement / Vault** flow (auto-link only applies to vault, not one-time payment).
4. The **PayPal app** installed and logged into the sandbox buyer (required for app switch).
5. Demo client is already configured with an App Link + deep-link fallback:
   ```java
   new PayPalClient(context, auth,
       Uri.parse("https://mobile-sdk-demo-site-...herokuapp.com/braintree-payments"),
       "com.braintreepayments.demo.braintree");
   ```

### Forcing App Link failure (the whole point of the feature)

On the device: **Settings → Apps → PayPal → Open by default → "Open supported links"** → turn **off**
(or "Don't open in this app"). Now PayPal can't App-Link back to the Demo, so returning is manual.

---

## Test matrix

> Watch analytics via FPTI/network logs or logcat. Event names are listed per case.

### Case 1 — Path A: `NoResult` auto-link (happy path) ✅ primary

1. Settings: App Switch ON, App Link disabled (above).
2. Start a **vault** flow → app switches to PayPal.
3. **Approve** the billing agreement in PayPal.
4. **Manually** return to the Demo (Recents / back), no deep link fires.
5. Demo's `onResume` calls `handleReturnToApp` → `NoResult` → `Success(autoLinkPending)` → `tokenize`.

**Expected:** nonce displayed (DisplayNonce screen). No restart of the PayPal flow.
**Analytics:** `...auto-link:handle-return:pending` → `...auto-link:started` → `...auto-link:succeeded` → `...tokenize:succeeded`.

### Case 2 — Path B: re-click auto-link ✅

1. Same setup; approve the BA, then return manually **without** letting `handleReturnToApp` resolve
   (e.g. navigate away, or use a `singleTop`/`onNewIntent` build so `onResume` doesn't complete it).
2. **Re-tap** the PayPal button.

**Expected:** nonce delivered via `tokenizeCallback` (the Demo wires this for BA flows) — the full
flow is **not** restarted.
**Analytics:** `...auto-link:reclick:started` → `...auto-link:started` → `...auto-link:succeeded` → `...auto-link:reclick:succeeded`.

### Case 3 — Path C: foreground trigger (`onNewIntent` / `singleTop`) ✅

> Demo's main activity is `singleTask`; to exercise Path C cleanly you may need a `singleTop`
> activity that only calls `handleReturnToApp` in `onNewIntent` (so a manual return with no new
> intent never calls it). Path C then fires from `ProcessLifecycleOwner`.

1. Setup as Case 1; approve the BA.
2. Manually return. No new intent → `handleReturnToApp` not called by the merchant.
3. `AppForegroundDetector.onStart` fires and pre-resolves the nonce.

**Expected:** nonce is resolved in the background; delivered when the merchant next calls
`handleReturnToApp` (→ `Success(nonce)`) or on a re-tap. Exactly **one** BTGW call.
**Analytics:** `...auto-link:started` → `...auto-link:succeeded`; on delivery `...auto-link:handle-return:succeeded`.

### Case 4 — URL return wins (regression) ✅

1. **Re-enable** "Open supported links" for PayPal (App Link works).
2. Run a vault flow, approve, let it App-Link back normally.

**Expected:** normal URL tokenization. **No** auto-link BTGW call. Store is cleared.
**Analytics:** `...handle-return:succeeded` → `...tokenize:succeeded`. **Absent:** any `auto-link:*`.

### Case 5 — 422 `BILLING_AGREEMENT_NOT_APPROVED` (error path) ⚠️ see below

### Case 6 — Session expiry (TTL 30 min)

1. Start a vault app switch (session stored), approve.
2. Wait **> 30 minutes** before returning (or temporarily lower `PendingPaymentStore.TTL_MS` in a
   local build to shorten the wait).
3. Return manually.

**Expected:** `handleReturnToApp` → `NoResult` (no auto-link); if `tokenize` is somehow reached with a
pending-but-expired session it returns `Failure` with `AUTO_LINK_EXPIRED_MESSAGE`.
**Analytics:** `...auto-link:expired`.

### Case 7 — No duplicate BTGW call (dedup)

1. Trigger Path C (foreground) **and** then call `handleReturnToApp` / re-tap quickly.

**Expected:** only **one** `...auto-link:started` / one BTGW request — the `CompletableDeferred`
funnels all callers to a single call; later callers await the same result.

---

## Case 5 in detail — 422 `BILLING_AGREEMENT_NOT_APPROVED`

This is the error case to verify. It happens when the SDK tries to tokenize a `ba_token` that the
buyer **never approved** — BTGW rejects it with HTTP 422 `BILLING_AGREEMENT_NOT_APPROVED`.

### How to reproduce (no server changes needed — sandbox returns it naturally)

1. Settings: App Switch ON, App Link disabled.
2. Start a **vault** flow → app switches to PayPal. *(The pending session is stored at
   `createPaymentAuthRequest`, before approval.)*
3. In PayPal, **do NOT approve** — press back / cancel / just switch away.
4. **Manually** return to the Demo.
5. Auto-link attempts to tokenize the unapproved BA → BTGW **422**.

### Expected behavior per path

| Path | What the buyer sees | Analytics | State after |
|------|--------------------|-----------|-------------|
| **A** (`handleReturnToApp`→`tokenize`) | Error surfaced via Demo `handleError(...)` (the 422 exception) | `...auto-link:handle-return:pending` → `...auto-link:started` → `...auto-link:failed` → `...tokenize:failed` | store **cleared** |
| **B** (re-click) | **No error shown** — re-click failure **falls through to a fresh PayPal flow** (by design) | `...auto-link:reclick:started` → `...auto-link:started` → `...auto-link:failed` → `...auto-link:reclick:failed` → normal `tokenize:started` | store **cleared**, new flow launched |
| **C** (foreground) | Nothing yet — failure is **swallowed**, session **preserved** for retry | `...auto-link:started` → `...auto-link:failed` | session kept; next `handleReturnToApp`/re-tap retries and then surfaces via Path A/B |

> **Note on Path B fall-through:** on a re-click auto-link failure, the SDK intentionally does not
> deliver a `Failure` to `tokenizeCallback`; it proceeds with a normal `createPaymentAuthRequest` so
> the buyer can complete a fresh flow. The 422 is only observable via analytics on this path.

### Wiring — already in place

No extra wiring is required to observe the 422:

- **Path A** — `PayPalFragment.completePayPalFlow(...)` passes `PayPalResult.Failure` to
  `handleError(...)`, which shows the error. ✅
- **Path B** — the BA `tokenizeCallback` in `PayPalFragment.launchPayPal(...)` also routes
  `PayPalResult.Failure` → `handleError(...)` (though, per above, re-click failures fall through and
  won't hit this for the 422). ✅
- **Path C** — failure is silent by design; verify via analytics/logcat, then confirm it surfaces on
  the subsequent Path A/B retry.

### Automated coverage (already merged in the PRs)

- `PayPalClientUnitTest`:
  - `tokenize with autoLinkPending returns Failure when BTGW fails` — asserts `PayPalResult.Failure`
    + `...auto-link:failed` + store cleared (this is the 422 case at the unit level).
  - `tokenize with autoLinkPending returns Failure when no session`.
- `AutoLinkTokenizeUseCaseUnitTest`: `invoke propagates tokenization errors`.

To assert the **specific** BTGW error shape in an integration test, stub
`internalPayPalClient.tokenize(...)` to throw a `BraintreeException("... BILLING_AGREEMENT_NOT_APPROVED ...")`
and assert the message flows into `PayPalResult.Failure.error.message` and the
`...auto-link:failed` event's `errorDescription`.

---

## Analytics quick reference

```
paypal:tokenize:auto-link:started
paypal:tokenize:auto-link:succeeded
paypal:tokenize:auto-link:failed
paypal:tokenize:auto-link:expired
paypal:tokenize:auto-link:handle-return:pending      # Path A routed to auto-link
paypal:tokenize:auto-link:handle-return:succeeded    # Path C nonce delivered on handle-return
paypal:tokenize:auto-link:reclick:started
paypal:tokenize:auto-link:reclick:succeeded
paypal:tokenize:auto-link:reclick:failed
```

## Sign-off checklist

- [ ] Case 1 — Path A happy path
- [ ] Case 2 — Path B re-click
- [ ] Case 3 — Path C foreground trigger (`singleTop`/`onNewIntent`)
- [ ] Case 4 — URL return wins (no auto-link when App Link works)
- [ ] Case 5 — 422 `BILLING_AGREEMENT_NOT_APPROVED` across paths A/B/C
- [ ] Case 6 — expiry (TTL)
- [ ] Case 7 — single BTGW call under racing triggers
