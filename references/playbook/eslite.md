# Eslite Checkout Rules

Use this reference when the merchant is `www.eslite.com`.

## Scope
Covers:
- checkout step 2 recovery
- recipient/address restoration
- payment-mode selection
- TapPay cardholder + iframe stage handling

## Required checkout sequence
For Eslite step 2, verify this order before payment work:
1. delivery method selected
2. orderer information stable
3. recipient information stable
4. payment mode selected as intended
5. cardholder field visible
6. TapPay iframes visible
7. only then attempt payment iframe execution

## Known stable selectors/hints
- `input[name='recipientName']`
- `input[name='recipientMobileNumber']`
- `input[name='recipientZip']`
- `select[name='recipientCity']`
- `select[name='recipientDistrict']`
- `input[name='recipientAddress']`
- `#step2-credit-card-name`
- `#cart-step2-checkout-btn`

## Address strategy
For Taiwan home delivery:
1. set city first using the real select interaction
2. wait for district options to refresh
3. set district
4. verify zip
5. then set detailed address

Proven common path:
- city `台北市` => value `2`
- district `中正區` => value `1`
- zip `100`

## Payment selection rule
Do not trust visual highlight alone.
Treat payment mode as active only when all relevant signals match:
- the credit-card option was really clicked
- cardholder field appears
- TapPay iframes appear
- checkout button remains plausible in context

Preferred click target:
- the visible clickable container whose text includes `信用卡一次付清`

## Dirty-page rule
Treat the page as dirty when any of these occur:
- login redirect happened mid-checkout
- address fields were cleared/re-rendered
- payment block collapsed and had to be reopened
- TapPay iframe targets were rebuilt after prior attempts

When dirty:
1. rebuild step 2 in order
2. restore address block
3. restore payment mode
4. re-check cardholder + iframe presence
5. only then attempt payment-field work

## Eslite + TapPay proven payment rule
The winning browser-only protocol on this site is:
- click the correct iframe for a single field
- input only that field
- immediately verify through `TPDirect.card.getTappayFieldsStatus()`
- do not move to the next field until the current field advanced
- do not use `Tab` as the main route

## Failure interpretation
- if click/focus succeeds but SDK status does not advance, classify as `unverified_input` or `sdk_status_not_advanced`
- if the tab/target disappears, classify as `runtime_tab_fault`
- if prior checkout work was invalidated by re-render/login bounce, classify as `page_state_dirty`
