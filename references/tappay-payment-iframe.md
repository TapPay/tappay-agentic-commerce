# TapPay Payment Iframe Protocol

Use this reference when the merchant's payment page uses TapPay JS SDK iframes for card input.

## Detecting TapPay iframes
The page uses TapPay iframes when:
- the payment area contains `<iframe>` elements (not plain `<input>` fields)
- `TPDirect` is defined in the page's JavaScript context
- `TPDirect.card.getTappayFieldsStatus()` is callable

If the payment area uses plain `<input>` elements instead, do NOT use this protocol.
See the merchant-specific reference file for plain-input instructions.

## Pre-execution checklist
Before attempting any iframe input:
1. confirm payment stage is truly active (not just visually selected)
2. verify iframes are present in the DOM
3. verify `TPDirect.card.getTappayFieldsStatus()` is callable
4. identify the three iframe areas separately: number, expiry, ccv

## Field execution order
Always fill in this order: number → expiry → ccv

For each field:
1. explicit click on the correct iframe area (do not rely on tab key)
2. input only that field's value
3. immediately call `TPDirect.card.getTappayFieldsStatus()` to verify
4. move to the next field ONLY when the current field's status has advanced
5. never assume a field was accepted just because the click/type succeeded

## Verification method
Primary: `TPDirect.card.getTappayFieldsStatus()`

Expected response shape:
```json
{
  "status": {
    "number": { "status": 2 },
    "expiry": { "status": 2 },
    "ccv": { "status": 2 }
  },
  "canGetPrime": true
}
```

Status codes:
- `0` = empty
- `1` = invalid
- `2` = valid / complete

Only proceed to submit when all three fields show status `2` and `canGetPrime` is `true`.

## Minimum result contract
Before claiming payment fields are ready, verify all of:
- `cardholder_done` — cardholder name field filled (if present)
- `number_status` — card number field status from SDK
- `expiry_status` — expiry field status from SDK
- `ccv_status` — CVV field status from SDK
- `can_get_prime` — SDK reports ready to get prime
- `blocked_reason` — explain why not ready if `can_get_prime` is false

## Fallback tree
If primary SDK verification fails, use in order:
1. front-end bridge (inject JS to read field state)
2. dedicated payment iframe MCP if available
3. browser-only hard-attack protocol (last resort)

## Hard-attack rules (browser-only last resort)
- test one field at a time; never batch
- do not loop endlessly — classify failure after 3 attempts per field
- focus on evidence from SDK status, not just successful clicks
- if status does not advance after explicit click + type, classify as a failure

## Failure classes
- `unverified_input` — click/type succeeded but SDK status did not advance
- `sdk_status_not_advanced` — field appears filled but SDK reports status 0 or 1
- `iframe_limit` — browser automation cannot interact with the iframe at all
- `runtime_tab_fault` — tab or target was lost during operation
- `page_state_dirty` — page re-rendered and iframes were rebuilt
- `stale_state_claim` — using old SDK status from a previous page instance

## Cross-platform caution
- browser-only hard attack is runtime-sensitive
- success in one browser/tool environment does not guarantee success in another
- always re-verify SDK status fresh in the current page instance
- never carry over iframe state assumptions from a prior session or page load
