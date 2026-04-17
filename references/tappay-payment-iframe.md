# TapPay Payment Iframe Protocol

Use this reference when the merchant checkout uses TapPay card iframes.

## Detecting TapPay iframes
The page uses TapPay iframes when:
- the payment area contains `<iframe>` elements (not plain `<input>` fields)
- `TPDirect` is defined in the page's JavaScript context
- `TPDirect.card.getTappayFieldsStatus()` is callable

If the payment area uses plain `<input>` elements instead, do NOT use this protocol.
See the merchant-specific reference file for plain-input instructions.

## Status code semantics
`TPDirect.card.getTappayFieldsStatus().status.<field>` returns:

| Code | Meaning |
|------|---------|
| `0`  | Field filled successfully, no issues — **this is SUCCESS** |
| `1`  | Field not yet filled |
| `2`  | Field has errors (errorColor shown in CardView) |
| `3`  | User is currently typing |

**Target state**: all three fields return `0` AND `canGetPrime === true`.

## Visual blanking is expected (do NOT treat as failure)
After entering the card number into a TapPay iframe, the field may appear visually blank.
This is **intentional security masking** by the TapPay SDK — the data is held internally.
Do NOT take a screenshot to "verify" card number input — the visual blank will mislead you.
Use `TPDirect.card.getTappayFieldsStatus()` as the ONLY reliable verification method.

## Verification priority (DEFAULT order)
1. **`TPDirect.card.getTappayFieldsStatus()`** — always use this first
2. page validation state (form error messages)
3. iframe container state changes
4. screenshot — **only as a last resort when SDK reports status `2` (error)**

Never claim success or failure based only on:
- successful click or focus event
- absence of thrown errors
- visual appearance of the field (fields may be blank due to security masking)

## Field execution order
Always fill in this order: number → expiry → ccv

For each field:
1. explicit click on the correct iframe area
2. input only that field's value
3. immediately call `TPDirect.card.getTappayFieldsStatus()` to verify
4. success when the field returns status `0`
5. move to the next field ONLY when current field status is `0`

## Pre-execution checklist
Before attempting any iframe input:
1. confirm payment stage is truly active (not just visually selected)
2. verify iframes are present in the DOM
3. verify `TPDirect.card.getTappayFieldsStatus()` is callable
4. identify the three iframe areas separately: number, expiry, ccv

## Step-by-step protocol

### Step 1: baseline
Capture:
- current URL
- current checkout/payment mode
- cardholder field state if present
- current `TPDirect.card.getTappayFieldsStatus()` (should show all fields at `1`)

### Step 2: locate the correct iframe
Identify number / expiry / ccv separately.
Do not treat all payment iframes as one combined target.

### Step 3: fill card number
1. click the card number iframe area
2. type the 16-digit card number
3. call `TPDirect.card.getTappayFieldsStatus()` — verify `status.number === 0`
4. if `status.number === 2`: there is an error — take a screenshot to investigate
5. if `status.number === 1`: field was not accepted — retry the click and type
6. if `status.number === 3`: still typing — wait and re-check
7. NOTE: the field will appear visually blank after input — this is NORMAL security masking

### Step 4: fill expiry
1. click the expiry iframe area
2. type expiry (format: MMYY, e.g. "0526")
3. call `TPDirect.card.getTappayFieldsStatus()` — verify `status.expiry === 0`
4. if not `0`: follow same recovery steps as card number

### Step 5: fill CVV
1. click the CVV iframe area
2. type the 3-digit CVV
3. call `TPDirect.card.getTappayFieldsStatus()` — verify `status.ccv === 0`
4. also verify `canGetPrime === true`

### Step 6: ready to submit
Proceed to submit ONLY when:
- `status.number === 0`
- `status.expiry === 0`
- `status.ccv === 0`
- `canGetPrime === true`

## Failure classes
- `unverified_input` — click/type succeeded but SDK status did not advance
- `sdk_status_not_advanced` — field appears filled but SDK still reports status `1`
- `sdk_status_error` — SDK reports status `2` (card data invalid or rejected)
- `iframe_limit` — browser automation cannot interact with the iframe at all
- `runtime_tab_fault` — tab or target was lost during operation
- `page_state_dirty` — page re-rendered and iframes were rebuilt
- `stale_state_claim` — using old SDK status from a previous page instance

## Hard-attack rules (last resort)
- test one field at a time; never batch
- do not loop endlessly — classify failure after 3 attempts per field
- focus on evidence from SDK status, not visual appearance
- if status does not advance after explicit click + type, classify as failure

## Cross-platform caution
- browser-only interaction is runtime-sensitive
- always re-verify SDK status fresh in the current page instance
- never carry over iframe state assumptions from a prior session or page load

## Reporting contract
For each attempt, return:
- `field_under_test`
- `sdk_status_before`
- `sdk_status_after`
- `can_get_prime`
- `result` (success / failure class)
- `next_safe_step`
