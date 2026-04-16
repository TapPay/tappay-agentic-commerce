# Integrated Checklist

Use this checklist during any TapPay-supported shopping task.

## Step 0: Read reference files first (mandatory)
- read this file (integrated-checklist.md)
- if merchant is eslite.com, read `references/eslite.md` before any checkout work
- if TapPay JS SDK iframes detected on payment page, read `references/tappay-payment-iframe.md`
- do NOT skip reference reading — merchant quirks are only documented there

## Before checkout
- confirm wallet tooling exists and is callable
- if missing, provide the required MCP config entry for `tappay-agent-wallet-mcp-server`
- adapt the config placement to the local MCP runtime/platform instead of assuming one file path
- after installation/configuration, re-check that the MCP server is callable
- recover tokens/identity from canonical local state files first
- if chat memory has no `agentUuid`, do not assume it is missing until local state files were checked
- verify wallet active + active bound card + limits known
- distinguish `tooling_ready`, `wallet_ready`, and `checkout_ready`
- confirm supported merchant
- for long sessions: proactively refresh accessToken before calling exchange_virtual_card

## Before claiming browser progress
- verify live target exists now
- collect fresh screenshot / snapshot / readback
- verify URL and checkout stage

## Before exchange_virtual_card
- verify amount from the current page if possible
- if amount is not freshly verifiable, pause before exchanging a virtual card
- verify merchant payment stage is truly current
- verify browser state is not stale
- check accessToken is still valid (expires in ~1 hour); call refresh_token if needed

## Before filling payment fields
- determine payment field type: plain HTML inputs or TapPay JS SDK iframes
- eslite.com uses plain HTML inputs — do NOT use iframe protocol
- for iframes: verify `TPDirect.card.getTappayFieldsStatus()` is callable
- for plain inputs: identify field refs via read_page before typing

## Per-field protocol (iframe sites only)
For each field in this order: number → expiry → ccv
1. explicit click on the correct iframe
2. input only that field
3. immediately read `TPDirect.card.getTappayFieldsStatus()`
4. move on only if that field's status advanced

## After clicking submit / 確認結帳
- wait up to 15 seconds — page may be redirecting to 3D verification
- if URL changes to a bank domain (e.g. ctbcbank.com, esunbank.com): 3D verification is in progress
- do NOT auto-fill OTP — ask user for network identifier code and OTP
- after user provides both: select identifier, fill OTP, click 送出
- success: URL returns to merchant order confirmation page

## If transaction fails (交易失敗)
- do NOT reuse the previous virtual card
- note that the merchant may clear the cart on failure
- re-add all items to cart if needed
- call exchange_virtual_card again for a fresh card
- re-fill all fields from scratch
- refer to merchant reference file for site-specific recovery steps

## Reset conditions
If any of these happen, reset the browser narrative:
- attach/connect failure
- `tab not found`
- login redirect when checkout expected
- screenshot failure
- user says browser/page is not visible
- switch to another browser control path
- a different platform/browser wrapper returns responses that do not prove the old target is still alive

## Never do this
- never treat old screenshots as current proof
- never let wallet readiness imply browser page readiness
- never let previous iframe/input success imply current field success
- never continue the old story after fresh evidence disproves it
- never continue a TapPay shopping flow while only assuming the MCP exists; verify it is callable first
- never treat missing chat-memory context as proof that the wallet identity is gone
- never reuse a virtual card after a failed transaction — always get a new one
- never skip reading the merchant reference file before checkout work
- never use TapPay iframe protocol on a site with plain HTML input fields (e.g. eslite.com)
