# Eslite Checkout Rules

Use this reference when the merchant is `www.eslite.com`.

## Scope
Covers:
- checkout step 2 recovery
- recipient/address restoration
- payment-mode selection
- credit card field filling (plain HTML inputs, NOT TapPay iframes)
- 3D verification OTP flow
- transaction failure recovery

## Important: Payment fields are plain HTML inputs
Eslite's credit card fields are standard HTML input elements, NOT TapPay JS SDK iframes.
Do NOT attempt `TPDirect.card.getTappayFieldsStatus()` on this site — it will not work.
Fill fields directly using click + type or form_input with ref IDs.

## Required checkout sequence
For Eslite step 2, verify this order before payment work:
1. delivery method selected (台灣-宅配(台灣地區) is the default)
2. orderer information stable (auto-filled from account)
3. recipient information filled (name, phone, address)
4. payment mode selected as 信用卡一次付清
5. cardholder name field visible and filled
6. card number, expiry, CVV fields visible and filled
7. click 確認結帳 to submit

## Known stable selectors/hints
- `input[placeholder="姓名"]` — recipient name
- `input[placeholder="號碼"]` — recipient phone
- zip code: plain text input (unlabelled, appears before city dropdown)
- city dropdown: `combobox "縣市"` — select by option text e.g. "台北市"
- district dropdown: `combobox "地區"` — select by option text e.g. "中正區"
- `input[placeholder="請填入詳細地址"]` — detailed address
- cardholder: `input[placeholder="請填入信用卡面持卡人英文姓名"]`
- card number: plain input field labelled "信用卡號"
- expiry: plain input field labelled "有效期限", format MM/YY — type as "0526" (no slash, field auto-formats)
- CVV: plain input field labelled "背面末三碼", default placeholder "000"
- submit: button "確認結帳"

## Address strategy
For Taiwan home delivery:
1. fill zip code first (plain text input)
2. select city using form_input with combobox ref
3. wait for district options to refresh
4. select district using form_input with combobox ref
5. fill detailed address

Proven common path:
- city `台北市` => value `2`
- district `中正區` => value `1`
- zip `100`

## Payment field filling strategy
Since fields are plain HTML inputs (not iframes):
1. fill cardholder name via form_input ref
2. click on card number field, then type the 16-digit number
3. click on expiry field, type as 4 digits e.g. "0526" (auto-formats to "05 / 26")
4. triple-click on CVV field to clear default "000", then type the 3-digit code
5. take a screenshot to verify all fields before submitting

## Payment selection rule
Treat payment mode as active only when:
- the 信用卡一次付清 option is visually highlighted (dark background)
- cardholder and card number fields are visible below it

Preferred click target:
- the radio button or row whose text includes `信用卡一次付清`

## 3D Verification flow (玉山銀行 / ctbcbank.com)
After clicking 確認結帳, the page redirects to a 3D secure verification page at `nmpi.ctbcbank.com`. This is expected and normal.

The page shows:
- Transaction details (merchant, amount, date, card last 4 digits)
- Verification method choice: "傳送OTP驗證碼" or "玉山Wallet App驗證"
- Four network identifier codes (e.g. BDFE, GHNJ, SLBQ, VFCI)
- An OTP input field

Steps:
1. Confirm amount and card match what is expected
2. Ensure "傳送OTP驗證碼" is selected
3. Click 下一步
4. Ask the user: (a) which network identifier matches their SMS, and (b) the OTP code
5. Select the correct network identifier radio button
6. Type the OTP into the input field
7. Click 送出
8. Wait for redirect back to eslite.com/cart/step3

Success signal: URL becomes `eslite.com/cart/step3` with `status=0` and an order number in query params.

## Transaction failure recovery
If the order shows 訂單成立 + 交易失敗:
- The cart is automatically cleared after failure
- There is no retry payment button on the order page
- Recovery steps:
  1. Note the failed order number
  2. Re-search and re-add all items to cart by product name
  3. Navigate to cart/step1, confirm items and total
  4. Proceed through checkout again
  5. Call exchange_virtual_card again to get a fresh virtual card — never reuse the old one
  6. Fill all fields fresh and resubmit

## Dirty-page rule
Treat the page as dirty when any of these occur:
- login redirect happened mid-checkout
- address fields were cleared or re-rendered
- payment block collapsed and had to be reopened

When dirty:
1. rebuild step 2 in full order
2. restore address block
3. re-select 信用卡一次付清
4. re-fill cardholder + card fields
5. verify via screenshot before submitting

## Failure interpretation
- card fields appear empty after typing → click field first, triple-click to clear default, then type
- 確認結帳 triggers loading overlay that never resolves → wait up to 15s; it may be an in-page 3D redirect
- URL stays on step2 after submit → look for red validation error messages
- 交易失敗 on order page → follow transaction failure recovery steps above
