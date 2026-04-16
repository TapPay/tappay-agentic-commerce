---
name: tappay-agentic-commerce
description: End-to-end TapPay shopping and checkout skill that combines wallet readiness, merchant selection, browser checkout execution, payment field handling, and stale-state recovery into one continuous contract. Use when the user wants the agent to find, prepare, and complete a TapPay-supported purchase without dropping context between wallet and web checkout stages.
---

# Goal
Run TapPay-supported shopping tasks as one continuous execution contract.

This skill combines what used to be split across:
- TapPay wallet orchestration
- merchant selection
- website checkout field writing
- payment field execution (HTML inputs or TapPay iframes depending on merchant)
- browser/session recovery discipline

The main purpose is to prevent handoff loss between wallet work and checkout work.

# When to use
Use this skill when the task involves any of the following in one user flow:
- finding a supported merchant
- verifying TapPay Agent Wallet readiness
- logging into or recovering wallet state
- obtaining a one-time virtual card
- operating a merchant checkout page
- filling shipping/contact/payment fields
- handling payment fields (HTML input or TapPay iframe)

Do not use this skill for:
- plain public browsing with no shopping intent
- unsupported merchants with no TapPay route
- non-TapPay payment flows unless the task still centrally depends on TapPay wallet state

# Core execution rule
Treat the shopping task as one integrated state machine, not as separate skills that loosely hand off.

The agent must preserve and continuously re-verify these state buckets together:
- wallet state
- merchant/site state
- browser/tab state
- checkout stage state
- payment-field verification state

Do not assume one state bucket implies another.
For example:
- valid wallet tokens do not prove the browser checkout tab is still alive
- a visible checkout page does not prove wallet readiness
- a previously issued virtual card does not prove the current payment page is still the same instance

# Unified workflow

## 0. Read reference files FIRST (mandatory — do this before any other step)
Before doing any merchant checkout work, read the relevant reference files:

**Always read:**
- `references/integrated-checklist.md`

**When merchant is eslite.com:**
- `references/eslite.md` — MUST be read before any step 2 checkout work on eslite.com
- Do NOT proceed to address or payment fields without reading this file first

**When TapPay card iframes are detected on page:**
- `references/tappay-payment-iframe.md` — read before attempting any iframe input

Not reading these files first is a protocol violation. Merchant-specific quirks (payment field type, 3D verification flow, failure recovery) are only documented in the reference files.

## 1. Confirm wallet tooling exists
Before doing any shopping work, confirm TapPay Agent Wallet tooling is installed and callable.

Preferred checks:
- inspect configured MCP servers
- call the server schema or list tools

Expected MCP server config:
```json
{
  "mcpServers": {
    "tappay-agent-wallet-mcp-server": {
      "type": "http",
      "url": "https://prod-client-mcp.tappaysdk.com/mcp"
    }
  }
}
```

Platform-neutral rule:
- do not assume the host uses the same MCP config file layout as the current workspace
- treat the JSON above as the canonical server entry, and adapt placement to the local MCP manager/runtime
- if the platform has a dedicated MCP config workflow, use that instead of hard-coding one file path

If missing:
- stop and ask whether to install/configure it
- do not pretend checkout can proceed without wallet tooling
- if the user approves, help add the MCP server config and then re-check availability before continuing

Install-guidance rule:
- when wallet tooling is missing, this skill should not merely say "install it"
- it should tell the user exactly which MCP server entry is required
- after configuration, it should verify that the server is callable before moving to wallet login or merchant checkout work

## 2. Recover wallet identity and tokens
Look for prior local state such as:
- `agentUuid`
- `agentSessionId`
- `accessToken`
- `refreshToken`
- stored wallet status metadata

Canonical persistence rule:
- do not treat chat memory as the primary source of wallet identity
- use local persistent state files as the primary source of truth
- recommended canonical files:
  - `.secrets/tappay-agent-wallet.json`
  - `.secrets/tappay-agent-wallet.meta.json`
- these files should persist:
  - `agentUuid`
  - `agentSessionId`
  - `accessToken`
  - `refreshToken`
  - `walletUuid` when known
  - `walletStatus` when known
  - `savedAt` / `lastCheck`

Recovery order:
1. canonical local secrets/state file
2. local meta/state file
3. any local registry entry if present
4. workspace memory or remembered context
5. refresh-token path
6. login using remembered `agentUuid`
7. first-time login only as the last resort

Rules:
- prefer recovery with remembered `agentUuid`
- refresh expired access tokens before re-login
- if refresh fails, fall back to login
- do not silently rotate to a new agent identity unless the user explicitly wants that
- missing from chat memory does not mean the wallet identity does not exist
- before concluding `agentUuid` is missing, search local persistent state first

Token expiry rule:
- accessToken typically expires in 1 hour
- for long sessions (many tool calls, extended user interaction), proactively call refresh_token before attempting exchange_virtual_card
- if exchange_virtual_card returns error 34026 (ACCESS_TOKEN_INVALID), call refresh_token immediately and retry

## 3. Confirm wallet readiness
Before merchant checkout work, verify:
- wallet exists
- wallet status is active
- at least one bound card is active
- transaction limits are known

If not ready:
- stop and report the exact missing condition

Bootstrap-state rule:
Treat these as separate readiness layers:
- `tooling_ready`: the MCP server exists and is callable
- `wallet_ready`: login/binding/card state is actually usable
- `checkout_ready`: the merchant browser checkout is current and actionable

Do not collapse these layers into one generic "ready" claim.

First-time bootstrap checklist:
1. MCP server callable
2. login flow completed
3. tokens retrieved
4. binding checked
5. wallet/card readiness verified
6. canonical local state persisted
7. only then may shopping execution begin

## 4. Discover or confirm merchant
If the user has not already chosen a merchant:
- call merchant discovery
- use shop descriptions as the first-pass routing signal
- recommend the strongest candidate or best 2 to 3 options

If the user is already on a known supported merchant:
- verify it matches a TapPay-supported shop
- continue without unnecessary rediscovery

No-playbook merchant rule:
- if no merchant-specific reference exists for the current shop, do not stop at "unsupported by playbook"
- run a generic merchant checkout contract first
- if the run succeeds or exposes stable patterns, summarize them into a reusable merchant reference rather than leaving them only in chat history

## 5. Establish browser reality before claiming page progress
Before any website execution:
- confirm there is a live browser target
- confirm the current URL/page matches the expected merchant flow
- gather fresh evidence from this turn segment

Platform-neutral browser rule:
- do not assume that browser tool, CLI browser wrapper, or another automation bridge share the same target/session semantics
- when switching control paths, re-verify target existence and page identity before carrying forward the old narrative
- if a platform does not support screenshots, use the strongest available fresh evidence and state that screenshot proof is unavailable

Allowed evidence:
- fresh screenshot
- fresh snapshot
- fresh readback/evaluate
- explicit confirmation that the same target still exists and is controllable

Disallowed evidence:
- old screenshots/snapshots from earlier turns
- remembered refs after a target fault
- old browser responses after the user says no browser/page is visible
- any stale narrative carried across login redirects, page re-renders, or target loss

If evidence is missing:
- current browser status is `unknown_current_state`
- do not narrate as if checkout is still live

## 6. Build and verify checkout stage
Once a live merchant page is confirmed:
1. verify login state on the merchant site if required
2. verify cart contents if product identity matters
3. move to checkout step by step
4. restore shipping/contact fields in verified blocks
5. select payment mode in a verified way
6. only then move to payment-field work

For write-heavy pages:
- write 1 to 4 related fields at a time
- read them back immediately
- if the page re-renders, re-check stage before continuing

Site-reference rule:
- if the merchant has a site reference under `references/<site>.md`, it MUST be read before any checkout-stage write work
- for `www.eslite.com`, use `references/eslite.md`
- this is not optional — see step 0

Payment field type detection:
- before filling payment fields, determine whether the merchant uses plain HTML inputs or TapPay JS SDK iframes
- eslite.com uses plain HTML inputs — do NOT use iframe protocol here
- sites with TapPay JS SDK will have iframe elements in the payment area — use `references/tappay-payment-iframe.md` for those

Generic merchant checkout contract:
If no site-specific reference exists yet, execute this minimum contract:
1. classify the page: cart / login / shipping / payment / confirmation
2. build a field map for recipient, phone, city, district/region, zip, address, payment option, cardholder, terms checkbox, submit button
3. restore data in verified blocks
4. verify payment-mode activation using real page signals, not only visual styling
5. verify the checkout page remains current before exchanging a virtual card
6. after success or strong learnings, record stable selectors, stage order, success signals, and failure patterns into a reusable merchant reference

## 7. Mandatory reset triggers
The following events invalidate earlier page claims until re-verified:
- browser attach/connect failure
- `tab not found`
- detached target
- merchant login redirect when checkout was expected
- visible page re-render that may invalidate refs or field values
- screenshot/snapshot failure after claiming active page work
- the user reports that the claimed page/browser is not actually visible
- switching browser control paths (browser tool, CLI browser, another wrapper)

When a reset trigger occurs:
1. stop relying on earlier page claims
2. re-verify wallet/browser/checkout state as needed
3. downgrade browser status to `blocked` or `unknown_current_state`
4. do not continue the old page narrative

## 8. Obtain virtual card only when checkout is truly ready
Obtain a one-time virtual card only when all are true:
- supported merchant is confirmed
- checkout is at the payment stage or one step away
- wallet is active
- transaction amount is known
- the browser page has been freshly verified in the current run segment

Transaction-amount rule:
- derive the amount from the current checkout page when possible
- if the visible amount cannot be freshly verified, pause before exchanging a virtual card
- do not rely only on remembered cart totals from an older page instance

Security rule:
- do not store virtual card number, expiry, or CCV in long-term memory
- use immediately, then discard
- this rule does not apply to durable wallet identity metadata such as `agentUuid`, `agentSessionId`, token state, wallet status, or recovery metadata, which should be persisted in canonical local state files

## 9. Payment field execution
Payment fields may be plain HTML inputs or TapPay JS SDK iframes depending on the merchant.

**Detecting field type:**
- if the payment section contains `<iframe>` elements → use TapPay iframe protocol (see `references/tappay-payment-iframe.md`)
- if the payment section contains standard `<input>` elements → fill directly with click + type or form_input

**For plain HTML inputs (e.g. eslite.com):**
1. fill cardholder name via form_input ref
2. click card number field, type 16-digit number
3. click expiry field, type as 4 digits (field may auto-format)
4. triple-click CVV field to clear placeholder, type 3-digit code
5. screenshot to verify before submitting

**For TapPay JS SDK iframes:**
See `references/tappay-payment-iframe.md` for the full protocol.

Minimum result contract (iframe path only):
- `cardholder_done`
- `number_status`
- `expiry_status`
- `ccv_status`
- `can_get_prime`
- `blocked_reason` when not ready

## 10. 3D Verification handling
Some merchants redirect to a bank 3D secure page after submit. This is expected.

General rules:
- wait up to 15 seconds after submit before treating a loading state as a fault — 3D redirect may be in progress
- if URL changes to a bank domain (e.g. ctbcbank.com, esunbank.com), treat it as 3D verification
- do NOT attempt to auto-fill OTP — ask the user for the network identifier code and OTP
- after user provides OTP, fill and submit on the bank page
- success signal: URL returns to merchant's order confirmation page

Merchant-specific 3D flow details are documented in the merchant reference file (e.g. `references/eslite.md`).

## 11. Transaction failure recovery
If a transaction fails after submission:
- do NOT reuse the virtual card from the failed transaction
- call exchange_virtual_card again to obtain a fresh card
- if the merchant clears the cart on failure, re-add all items before retrying
- check the merchant reference file for merchant-specific recovery steps

Failure classification:
- `transaction_failed`: order was created but payment was rejected by the bank
- `runtime_tab_fault`: browser tab lost during processing
- `page_state_dirty`: page re-rendered and fields were reset
- `3d_verification_failed`: OTP or network identifier was rejected
- `token_expired`: accessToken expired mid-session — call refresh_token

## 12. Payment iframe fallback tree (iframe merchants only)
Use this order:
1. front-end bridge if available
2. dedicated payment iframe MCP if available
3. browser-only hard-attack protocol as last resort

Cross-platform caution:
- browser-only hard attack is runtime-sensitive
- success on one host/browser/tool path does not imply success on another
- do not describe the browser-only path as generally reliable unless the current runtime has freshly proven it

Hard-attack rules:
- test one field at a time
- do not loop endlessly
- focus on evidence, not just successful clicks
- if status does not advance, classify it rather than pretending progress

## 13. Evidence-first reporting
When reporting progress, include:
- wallet_tooling_status
- login_status
- merchant_status
- browser_status
- checkout_stage
- payment_stage_status
- evidence_of_result
- current_status: success / partial / failed / blocked
- next_safe_step

Progress-claim rules:
- do not say you are still operating a visible merchant page unless it was freshly verified this run segment
- if the user questions whether the browser page is open, verify immediately before any further progress claim
- if fresh evidence contradicts the current narrative, replace the narrative immediately

## 14. Anti-fragmentation rules
To avoid losing context mid-task:
- keep wallet state and checkout state in the same running narrative
- do not describe handoff as complete until the receiving stage has been freshly verified
- after every major fault, reconstruct the full state summary before continuing
- if the task gets long, prefer a short explicit integrated status recap over silently relying on remembered state

# Reference layout
Use references in this order:
1. `references/integrated-checklist.md` — always read first
2. `references/<site>.md` — when the merchant has a dedicated site reference (MANDATORY before checkout)
3. `references/tappay-payment-iframe.md` — only when TapPay JS SDK iframes are detected on the page

Current references:
- `references/integrated-checklist.md`
- `references/eslite.md`
- `references/tappay-payment-iframe.md`

# Output expectations
For non-trivial shopping tasks, always report:
- wallet_tooling_status
- login_status
- selected_shop
- browser_status
- checkout_stage
- payment_stage_status
- evidence_of_result
- current_status
- next_safe_step

# Execution rule
If the user says continue, proceed with the next real tool step.
If state is uncertain, the next real tool step is state verification, not storytelling.
