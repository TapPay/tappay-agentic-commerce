# FunNow Checkout Playbook

Use this reference when the merchant is `www.myfunnow.com`.

## Site entry URL
Always use the TapPay-tracked entry URL:
```
https://www.myfunnow.com/zh-tw?utm_source=ai_agent&utm_medium=experimental_payment
```

## Scope
Covers:
- category and date/time filtering
- product selection and time slot selection
- checkout preview page field handling
- TapPay iframe payment execution
- 3D verification flow (esunbank)
- order confirmation

---

## Booking flow (step by step)

### Step 1: Set filters on the homepage
After navigating to the entry URL, the homepage shows a filter bar with:
- 享樂類別 (category)
- Now / 日期 (date)
- 時間 (time)
- 人數 (headcount)
- Search box

**Date filter:**
- Click「日期」to open the calendar
- Select the desired date (today = highlighted in orange/red)
- Click「顯示超過 100 筆商品」to apply

**Category filter:**
- After setting the date, click the desired category icon (e.g.「按摩」)
- This navigates to `/zh-tw/regions/1/categories/10` for massage in Taipei/桃園

**Time filter (on category page):**
- Click「時間」filter button on the category listing page
- Time slots are grouped: 早上 06:00–10:59 / 中午 11:00–13:59 / 下午 14:00–16:59
- Click「全部時段」checkbox next to the desired period (e.g. 下午) to select all slots in that range
- Click「顯示超過 100 筆商品」to apply
- The filter bar shows「下午」in orange when active

**Category → region ID mapping (Taipei/桃園):**
- 按摩: `/zh-tw/regions/1/categories/10`
- URL filter params for afternoon: `?filter_times=14:00&filter_times=14:30&filter_times=15:00&filter_times=15:30&filter_times=16:00&filter_times=16:30`

### Step 2: Select a shop
- Browse listing cards; each shows name, price (起), rating, and「最早可預訂」time
- Check「最早可預訂」to confirm today's availability — if it shows a future date (e.g. 04/21), the shop is not available today
- Click a shop card — it opens in a new tab
- Shop page shows: store info, operating hours, 店家介紹, and 熱門商品

### Step 3: Select a product
- Scroll to「熱門商品」section
- Click the desired product card — it opens the product detail page
- Product page shows: photos, discounted price (with strikethrough original), rating, and「看時間」button at the bottom

### Step 4: Select a time slot
- Click「看時間」— navigates to `/zh-tw/booking/<product_id>`
- A calendar + horizontal time slot carousel appears
- The calendar defaults to today; click a date to change
- Time slot carousel: click the `›` right arrow button to scroll forward (each click = one 15-minute slot)
  - From 11:30 to 14:00 requires approximately 10 right-arrow clicks
  - The currently selected slot is shown highlighted in orange/red with the time below it
  - The right panel live preview shows: 開始 / 結束 times and 結帳金額
- Click「立即預訂」to proceed to checkout

### Step 5: Checkout preview page (`/booking/<id>/preview`)
The preview page has these sections (top to bottom):
1. **填寫備註** — optional special requests note
2. **結帳明細** — order summary: item name, discount (選擇優惠), Fun幣折抵, 結帳金額, 1% 回饋
3. **報帳** — expense receipt toggle (開立報帳發票)
4. **付款方式** — payment method selection (see below)
5. **取消與更改訂單** — cancellation and change policy
6. **聯絡方式** — contact email (pre-filled from FunNow account)
7. **儲存卡片，TWD X** — the submit/pay button at the very bottom

**Payment method options:**
- 我的信用卡 (saved card)
- **新增信用卡** ← use this for TapPay virtual card
- LINE Pay
- AFTEE先享後付

**When「新增信用卡」is selected**, three card fields appear:
- 信用卡卡號 (TapPay iframe)
- MM / YY (TapPay iframe)
- CVC / CVV (TapPay iframe)
- 信用卡上顯示之英文姓名 (plain HTML input)

A notice appears below:「因應信用卡政策，請填寫下方驗證資訊」with two pre-filled fields:
- 發卡銀行留存之手機號碼 (from FunNow account)
- 發卡銀行留存之 E-mail (from FunNow account)

---

## Payment field type
FunNow uses **TapPay JS SDK iframes** for card number, expiry, and CVV.
Do NOT attempt to fill these with plain input methods. Always use the iframe protocol.

### Confirming TapPay presence
```javascript
typeof TPDirect !== 'undefined'
// → true

// Baseline: all fields should be 1 (unfilled) before input
JSON.stringify(TPDirect.card.getTappayFieldsStatus())
// → {"canGetPrime":false,"hasError":false,"cardType":"unknown","status":{"number":1,"expiry":1,"ccv":1}}
```

### Locating iframe positions (run after each scroll)
```javascript
Array.from(document.querySelectorAll('iframe'))
  .map((f, i) => {
    const r = f.getBoundingClientRect();
    return {index: i, src: f.src.substring(0,50), x: Math.round(r.x), y: Math.round(r.y), w: Math.round(r.width), h: Math.round(r.height)};
  })
  .filter(f => f.h > 10 && f.src.includes('tappay'))
// Returns 3 iframes:
// index 0: card number — wide (~318px), appears first
// index 1: expiry      — narrow left (~140px)
// index 2: CVV         — narrow right (~136px), same row as expiry
```

**CRITICAL:** iframe viewport y-coordinates shift after scrolling. Always re-query immediately before clicking.

### Proven fill protocol
Fill strictly in this order: **number → expiry → ccv**

For each field:
1. Re-query iframe positions with the JavaScript above (after any scroll)
2. Click the center of the target iframe: `(x + width/2, y + height/2)`
3. Type the value immediately
4. Verify: `JSON.stringify(TPDirect.card.getTappayFieldsStatus())`
5. Field must return `0` before moving to the next

**Values to type:**
- Card number: 16-digit string (no spaces)
- Expiry: 4 digits `MMYY` (e.g. `0526` for 05/26)
- CVV: 3-digit string

**Target state before submit:**
```javascript
{"canGetPrime":true,"hasError":false,"status":{"number":0,"expiry":0,"ccv":0}}
```

### Cardholder name (plain input)
- After iframes are complete, fill the cardholder name field
- It is a standard HTML `<input>` with placeholder `信用卡上顯示之英文姓名`
- Use `form_input` with its ref from `read_page` to set the value
- Use the user's English name in UPPERCASE (e.g. `JOSEPH`)

### Submit button
- Label: `儲存卡片，TWD X` (where X is the checkout amount)
- Located at the very bottom of the page below the contact info section
- Scroll down fully to find it

---

## 3D Verification flow (esunbank)

After clicking「儲存卡片」, the page redirects to:
```
https://acs.esunbank.com.tw/acs-auth-web/challenge/brw/...
```

This is **玉山銀行 交易及綁卡驗證服務** (Mastercard ID Check 3DS).

### Page 1: Choose verification method
Shows transaction summary (merchant = ZOEK INC, amount, date, masked card) and two options:
- ◉ 傳送OTP驗證密碼 ← default, use this
- ○ 玉山Wallet App驗證

Click「下一步」to send the OTP SMS.

### Page 2: Enter OTP
Shows:
- 4 radio button choices for **網頁識別碼** (anti-phishing identifier, e.g. IRNZ / MZIU / OGUC / WDGG)
- OTP input field (驗證密碼)
- 送出 / 取消驗證 / 重送驗證密碼 buttons

**Protocol — always ask the user:**
> "已傳送 OTP 簡訊！請告訴我您簡訊中顯示的**網頁識別碼**，以及收到的 **OTP 驗證碼**。"

After user provides both:
1. Select the matching radio button for the identifier
2. Click the OTP input field and type the 6-digit code
3. Click「送出」

**Never auto-fill or guess the OTP — always wait for the user to provide it.**

### Success signal
URL redirects to:
```
https://www.myfunnow.com/zh-tw/orders/info/<order_id>
```
This is the order confirmation page — booking is complete.

---

## Order confirmation page (`/orders/info/<order_id>`)

Left panel:
- **訂單編號** — order number (e.g. 36717532081582)
- **訂購內容** — product name
- **取消與更改訂單** — cancellation/change links

Right panel:
- Status badge: **預訂完成** (step 1 of 3)
- Shop name and product name
- **時間**: start ~ end datetime
- **地址**: full address with 地圖 link
- **出示兌換碼**: large numeric code (e.g. `108173`) — user must show this at the venue

**Always report the 兌換碼 to the user after successful booking.**

### Cancellation policy
- Free change: once, up to 3 hours before appointment
- Free cancellation: up to 72 hours before appointment start
- 20% fee: 3–72 hours before start
- 100% fee: within 3 hours of start (no-show)

---

## Known failure patterns and recovery

### Time slot carousel — reaching afternoon from morning default
- Default selected slot is usually 11:30 or the earliest available
- Each click on `›` advances by one 15-minute slot
- To reach 14:00 from 11:30 requires ~10 clicks; verify right panel shows the correct time after each group of clicks

### iframe click not registering (SDK status stays at 1)
- Scroll so the payment section is **fully visible** before clicking
- Always re-query iframe positions with `getBoundingClientRect()` after scrolling — positions shift significantly
- First attempt may miss if the page is still settling; retry once with fresh coordinates
- Confirmed working click target: center of each iframe (`x + w/2, y + h/2`)

### Card number field appears blank after input
- Expected TapPay security masking behaviour — not a failure
- Do NOT rely on visual appearance; trust only `getTappayFieldsStatus()`

### FunNow session expired / login redirect mid-checkout
- If page redirects to login during checkout, the booking state is lost
- After re-login, navigate back to the product page and restart from Step 4 (time slot selection)

### OTP expired or not received
- Click「重送驗證密碼」on the bank verification page to resend
- If still not received, advise user to contact 玉山銀行客服: (02)2182-1313 / 0800-30-1313

---

## Quick checklist

**Before clicking 儲存卡片:**
- [ ] Correct product and time slot confirmed in right panel
- [ ] 「新增信用卡」selected
- [ ] Virtual card obtained (amount matches 結帳金額)
- [ ] TPDirect baseline: all fields `1`
- [ ] iframe positions freshly queried
- [ ] `number` status `0` ✅
- [ ] `expiry` status `0` ✅
- [ ] `ccv` status `0` ✅
- [ ] `canGetPrime: true` ✅
- [ ] Cardholder name filled

**After 3DS:**
- [ ] Asked user for identifier code + OTP (never guessed)
- [ ] URL returned to `/orders/info/<id>` = success
- [ ] 兌換碼 reported to user
