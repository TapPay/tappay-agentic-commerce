# Bibian (比比昂) — Japan Daigou Merchant Reference

This file is a merchant-specific reference for `tappay-agentic-commerce`.
Read this **before** any checkout work on `bibian.co.jp`. Read it together
with `tappay-payment-iframe.md`, because Bibian uses TapPay JS SDK iframes
on its credit card form.

> Path: `references/bibian.md`
> Applies to: `bibian.co.jp` (also surfaces as `比比昂` and `Bibian` in
> `discover_shops`)

---

## When to use

Use this reference whenever the active merchant is `bibian.co.jp`. Bibian is
the only `discover_shops` shop in TapPay's prod merchant list whose primary
business is Japanese cross-border proxy buying (代購), so any user request
phrased as "I want to buy this Japanese site item that does not ship to
Taiwan" routes here.

Typical entry conditions:

- User pasted a product URL from a Japanese merchant (Converse Japan, Rakuten,
  ZOZOTOWN, mercari, Rakuma, JDirectItems Auction, ZOZO, beauty / drug
  brand sites, etc.).
- User wants Bibian to source, consolidate, and ship the item to Taiwan.
- User has, or is willing to create, a Bibian account.

Do not use this reference if:

- The user is buying directly from a Taiwan domestic store (eslite, FunNow,
  AsiaYo etc.).
- The merchant is not in `discover_shops` and has no Bibian-style 代購 chain.

---

## Site identity

| Property | Value |
| --- | --- |
| Domain | `https://www.bibian.co.jp` |
| Owner | PChome Bibian Inc. (網家比比昂集團子公司) |
| Backend currency | JPY (¥) on cart inputs; TWD on summaries and payment |
| Conversion rate display | Top-right of cart pages, e.g. `1日円=0.2167元` |
| Payment gateway | TapPay (`喬睿科技 TapPay 金流交易系統`) |
| Card form integration | **TapPay JS SDK iframes** (3 visible iframes for card number / expiry / CCV plus fraud + API iframes hidden) |
| Card brands accepted | Visa, Master, JCB |
| 3DS issuer used by bound JCB | E.Sun Bank (`acs.esunbank.com.tw`, J/Secure) |

---

## Pre-checkout prerequisites

Bibian enforces several profile fields before allowing payment. Check these
before walking the cart through to `payment_p.php`, otherwise the user
will be punted to `member_edit.php?type=basic` and lose flow continuity.

Required member profile fields (`/member_edit.php?type=basic`):

| Field | Required for checkout | Notes |
| --- | --- | --- |
| 中文姓名 | Yes | Used as customs declaration name |
| 性別 | Yes | |
| **身分證字號** | **Yes** | Required for EZWAY 報關. **Agent must NOT auto-fill this.** It is a Taiwan government identifier and falls under PII rules in `user_privacy`. Tell the user to enter it themselves. |
| 出生日期 | Yes | EZWAY required. Ask the user; do not infer. |
| 手機號碼 | Yes | Used by Bibian SMS notifications and 3DS OTP delivery if bank uses cell number for OTP |

Required additional state on Bibian:

- Logged-in Bibian session (`Hi {name}` visible at top right and `登出` link
  present). If not logged in, `cart_new.php` will show items but submitting
  the URL search will trigger a silent login redirect that often hangs the
  renderer (see "Known quirks").
- A configured shipping address (`收貨/進口資訊`). Bibian pre-fills it from
  member profile.
- A pre-configured 全家 store if you plan to use `超商取貨` (default
  recommended). The store is editable inline.

---

## Daigou item entry workflow

Bibian's 代購 flow is centered on URL submission. There are two reliable
entry points and one unreliable one.

### Entry point 1 — `/buy/` 主搜尋框

URL: `https://www.bibian.co.jp/buy/`

Field: `<input name="jp_keyword" placeholder="請貼上日本代買商品網址或輸入關鍵字">`

Submission pattern:

1. Set value via native setter on the input prototype so React-style
   listeners fire:
   ```js
   const input = document.querySelector('input[name="jp_keyword"]');
   const setter = Object.getOwnPropertyDescriptor(
     Object.getPrototypeOf(input), 'value'
   ).set;
   setter.call(input, productUrl);
   input.dispatchEvent(new Event('input',  { bubbles: true }));
   input.dispatchEvent(new Event('change', { bubbles: true }));
   ```
2. Focus the input and press `Enter` via the browser keyboard tool.
   Pressing `Enter` triggers Bibian's JS handler that auto-redirects to
   `cart_new.php` with the item provisionally added.

After submission, the response is **non-deterministic on metadata**:

- Sometimes Bibian's scraper returns a fully populated item: image +
  product name + spec + JPY price.
- Sometimes only the image populates; `name_m[0]`, `style_m[0]`, and
  `price_[0]` come back empty (red-warning state). This depends on
  whether the source URL pattern matches Bibian's whitelist scraper for
  that merchant.

If only the image is captured, fall back to **Entry point 3 (manual fill)**.

### Entry point 2 — Direct cart URL

URL: `https://www.bibian.co.jp/buy/cart_new.php`

Use when navigating in from another tab, after deleting an empty cart row
and starting over, or after the user logs in and you want to resume.

### Entry point 3 — Manual fill of an empty cart row

When Bibian fails to scrape metadata, it still creates an empty row keyed
by index `[0]`. Selectors:

| Field | Selector |
| --- | --- |
| 商品名稱 | `input[name="name_m[0]"]` |
| 規格 | `input[name="style_m[0]"]` |
| 單價 (JPY) | `input[name="price_[0]"]` |
| 數量 | `input[name="num_[0]"]` (number, default 1) |

Authoritative price source: the merchant's product page. If the agent
cannot fetch the merchant page (egress proxy may block specific domains
such as `converse.co.jp`), open the product page in a separate tab via
the Chrome browser tool to read price/title/SKU, then fill the cart.

Always **verify against the live merchant page before filling price**.
Stale price guesses become a source of dispute when Bibian later finds
the actual price differs (the `當商品漲價時` radio decides who handles
the gap, see below).

### Known cart quirks

- Cart can hold up to **15 items**.
- Stale failed entries (e.g., from a prior frozen-renderer attempt) stay
  in the cart with empty fields and `0` price. Delete them via the row's
  `刪除` link before continuing — Bibian otherwise treats them as live
  rows and blocks checkout.
- Each row has a per-row select checkbox `chk_0`. Make sure the row you
  want is checked.

---

## Checkout stage workflow

Once the cart row is valid, click `前往結帳` (link with
`href="javascript:send('<merchantKey>')"` where `<merchantKey>` is a
short identifier Bibian derives from the source domain). The browser
navigates to `/buy/cart_order_new.php`.

### Mandatory radio groups

Coupon selection is gated behind shipping selection — clicking the
coupon `選擇` link before both shipping rows are picked yields a
modal: `請先選擇國際及台灣運送方式`. Set all four radios first.

| Group `name` | Recommended `value` | Meaning |
| --- | --- | --- |
| `partial_bid` | `1` | Continue purchasing other items if some go OOS (推薦) |
| `price_decide` | `price_decide` | Allow ≤ 5% upward price drift to auto-purchase (推薦) |
| `transit_inter_bid` | `air` | International airfreight, ¥150/kg (推薦 for shoes / small items) |
| `tw-delivery` | `超商取貨` | Family Mart pickup, NT$60 (推薦) |

> ⚠️ **Style-only radio quirk** — these visual radios are styled
> wrappers; clicking the visible circle by coordinate or by `ref` does
> **not** flip the underlying `<input>`. You must click the wrapping
> `<label>` element, or call `radio.click()` after `label.click()`
> falls through. The reliable one-shot is:
> ```js
> const r = document.querySelector(`input[type="radio"][name="${n}"][value="${v}"]`);
> (r.closest('label') || r).click();
> ```

### Other fields on the order page

- `備註` — free-text, optional. Used for shopper notes (e.g. "do not
  remove tags", "請賣家避免折到鞋盒").
- `收貨/進口資訊` — already populated from member profile. If the user
  wants a different recipient, click `編輯` and walk them through the
  EZWAY-compliant fields (do not fill ID number or address yourself).

### Service terms checkbox

Checkbox label `我已閱讀並同意以上服務條款` must be checked before
`訂單付款` is clickable. The checkbox is a styled span; click via the
parent `<label>` for reliability.

Service terms (verbatim, source: 2026-04-28):

1. 購物車會自動依照賣家分別成立訂單，並逐筆收取費用。
2. 實際金額將依賣家收費為準（如產生日本當地運費或賣家有折扣優惠等），並更新於第二階段費用。
3. **訂單一、二階段付款方式需相同**。
4. 若賣家分批出貨，將在商品「全數到齊後寄出」。如本訂單中包含現貨與預購商品，需將現貨商品優先寄送，請務必分開下單。

Term 3 is the single most important constraint for TapPay agentic
commerce — see "Two-stage payment model" below.

### Order page resets on re-entry

Returning to `cart_order_new.php` after navigating away (e.g., to fix
member profile fields) **resets all four radios and clears coupon
selection**, but **preserves cart line items**. Always re-pick radios
and re-apply coupons after any side trip.

---

## Discount model

Bibian exposes two distinct discount entry points on the order page:

| Type | Entry | Where coupons live |
| --- | --- | --- |
| 折扣碼 (discount code) | `輸入` link → modal text input | User-supplied promo code |
| 折價券 (coupon) | `選擇` link → modal list | Account-bound coupon wallet at `/member_coupon.php` |

> ⚠️ **Prerequisite — both shipping methods must be selected first.**
> Bibian's order page hard-gates the coupon `選擇` modal on shipping
> state. Clicking `選擇` before **國際運送方式** (`transit_inter_bid`)
> **and** **台灣運送方式** (`tw-delivery`) are both set yields the
> modal: `折價券 — 請先選擇國際及台灣運送方式` and exits without
> showing any coupons. Always finalize the four radio groups
> (`partial_bid`, `price_decide`, `transit_inter_bid`, `tw-delivery`)
> before opening the coupon modal. The same rule applies after any
> side-trip that resets the radios (see "Order page resets on
> re-entry" above) — you must re-pick shipping before re-applying
> coupons.

### Coupon wallet

Full coupon list lives at `https://www.bibian.co.jp/member_coupon.php`,
filterable by `可使用 / 即將上線 / 不可使用`. As of 2026-04 the wallet
shows up to 30 coupons issued for various sub-brands.

Important coupon attributes:

- **限定子品牌** — coupons usually scope to a single Bibian sub-brand
  (`日本代購`, `日本樂天`, `mercari`, `Rakuma`, `JDirectItems Auction`,
  `隨選即買`). Cross-channel use is not allowed.
- **剩餘次數** — `1次` for first-purchase rewards (一次性), `99次`
  for unlimited promotional coupons.
- **抵扣規則** — minimum spend threshold and discount cap. Click the
  coupon's `抵扣規則` to expand inline rules.

### Threshold patterns observed (日本代購, 2026-04)

| Coupon | Min spend (JPY) | Discount |
| --- | --- | --- |
| 【新客體驗禮】日本代購 首次下單 5 折 | (none) | 50% off 代購手續費, capped, 1次 |
| 【日本代購】小單有折 ¥500 隨手結帳金 | ¥10,000 | -¥500, 99次 |
| 【日本代購】8%OFF 封頂券現領現折 | (varies) | 8% OFF |
| 【日本代購】¥3,500 換季應援無限領 | ¥30,000 | 6% OFF, cap ¥3,500, 99次 |

### Critical: failed-payment coupon lock

A coupon selected during checkout is **considered consumed at the
moment it is selected and confirmed in the modal**, not at successful
payment. If the order is created but payment fails or is abandoned, the
1-time coupon stays locked to that order indefinitely (or until Bibian
auto-cancels the unpaid order — typically several days later).

Implications:

- Do not pre-confirm a 1-time coupon unless you are confident the
  payment will go through. For TapPay virtual card flows where SDK
  validation has historically been brittle, prefer to test the payment
  field state first, then go back and apply the coupon.
- If a 1-time coupon goes missing, check `/member_coupon.php` under
  `不可使用` for a "locked to order #" record.

### Modal: "您目前無可使用的折價券"

This message in the order-page coupon modal does not mean the wallet
is empty. It means "no coupon currently in the wallet has minimum-spend
thresholds satisfied by **this** order's JPY total". Cross-check by
opening `/member_coupon.php` directly. A common gotcha is being just
under a threshold (e.g., a coupon needs ¥10,000 minimum spend and the
order total comes in ¥100 below — the modal silently treats the
coupon as unusable for this order).

---

## Payment workflow

### Payment method selector — `/payment_p.php`

Bibian's payment method selector page lists multiple Bibian-side
payment options; this playbook covers only **刷卡付款 → 信用卡**, the
single path compatible with a TapPay Agent Wallet one-time virtual
card.

Click `前往刷卡` → lands on `/payment_p.php?t=creditcard`.

> Bibian's term-3 (`訂單一、二階段付款方式需相同`) means the second-stage
> bill that arrives later must also be paid by credit card — see
> "Two-stage payment model" below for how this interacts with Agent
> Wallet's one-time virtual cards.

### Credit card form — `/payment_p.php?t=creditcard`

Form layout:

| Field | UI | Backing |
| --- | --- | --- |
| 卡號 | input box (looks plain) | **TapPay iframe** (`js.tappaysdk.com/sdk/tpdirect/tappay-field`) |
| 卡片到期日 | input box | **TapPay iframe** |
| 卡片後三碼 | input box | **TapPay iframe** |
| 同意記憶卡號 | checkbox `id=remembermycard` | Plain HTML — leave unchecked for one-time virtual cards |

The page also exposes the global `window.TPDirect` SDK. **There is no
`TPDirect.card.fill()` method**, so you cannot inject card data via
JavaScript. Use the keyboard-typing path documented in
`references/tappay-payment-iframe.md`.

Verification API:
```js
window.TPDirect.card.getTappayFieldsStatus();
// returns { canGetPrime, cardType, hasError, status: { ccv, expiry, number } }
// Status codes: 0 valid / 1 empty / 2 invalid / 3 untouched
```

### TapPay Agent Wallet virtual card behavior at Bibian

Empirically observed (2026-04):

- TapPay Agent Wallet virtual card numbers begin `87xx`. They are not
  Luhn-conformant and `cardType` resolves to `unknown`.
- Earlier in 2026-04 the SDK rejected these cards with `status.number = 2`
  and `canGetPrime = false`, blocking the flow at the merchant frontend
  entirely — the user could not even reach `getPrime()`.
- By 2026-04-28 the same BIN range passes validation:
  `status.number = 0`, `cardType = unknown`, `canGetPrime = true`. The
  SDK appears to whitelist the agent wallet BIN even though Luhn fails.

If you encounter the older blocked behavior, classify as
`blocked_reason: tappay_sdk_rejects_agent_wallet_bin` and stop. Do not
loop, do not type the card multiple times, do not try alternate
formats — it is a server-side SDK whitelist issue.

### 3DS verification

After clicking `確認付款` with a valid prime token, Bibian POSTs to
TapPay backend and is redirected to the issuer ACS page. For users
whose underlying bound card is JCB issued by E.Sun Bank, the URL
becomes:

`https://acs.esunbank.com.tw/acs-auth-web/challenge/brw/J/2.2.0/...`

Page elements:

| Element | Note |
| --- | --- |
| 特約商店 | Always shows `TapPay` (not Bibian) |
| 交易金額 | Must match the order amount in TWD |
| 信用卡號 末四碼 | Must match the user's bound card, not `87xx` virtual |
| Verification options | `傳送 OTP 驗證密碼` (default) / `玉山 Wallet App 驗證` |

OTP page (`/process` URL suffix) presents:

1. **網頁識別碼 (anti-phishing code)** — radio choice between four 4-letter
   codes (e.g., ATIM / JOGM / OSHK / VFBJ). Only one matches the SMS the
   user receives. **The user must pick this themselves**; do not guess.
2. **驗證密碼** — numeric OTP input.
3. **送出** / **取消驗證** / **重送驗證密碼** buttons.

Per `tappay-agentic-commerce` security rules: do **not** auto-fill OTP
or anti-phishing code. Walk the user through what the screen wants and
have them type it themselves, OR have them type and tell you the code,
in which case you may fill `驗證密碼` and submit. Either way, the
identifier-code radio is theirs to pick.

Successful 3DS redirects to:
`https://www.bibian.co.jp/payment_p.php?t=credit_finish`
with body text `訂單付款成功`.

---

## Two-stage payment model (核心限制)

Bibian charges the buyer in two stages:

| Stage | When | What is charged |
| --- | --- | --- |
| 第一階段 | At the time of order placement (this flow) | **Product price only** (JPY → TWD via current rate) |
| 第二階段 | After goods arrive at Bibian's Taiwan warehouse | International freight (¥150/kg airfreight billed by actual weight), domestic Taiwan delivery, customs duty if any, Bibian handling surcharges |

Service term 3 forces both stages to use the **same payment method
class**. Because this playbook restricts stage 1 to credit card, stage 2
must also be paid by credit card. Practical implications for TapPay
Agent Wallet:

- A virtual card is one-time-use and short-lived (~30 minutes from
  `exchange_virtual_card`). It cannot be carried over to stage 2 weeks
  later.
- For stage 2, the user can still satisfy term 3 by paying with **any**
  credit card — including a fresh TapPay virtual card issued via the
  same Agent Wallet at that future time. Bibian's term-3 constraint is
  card *type* (credit card), not card *number*.
- Set this expectation with the user up front so they are not surprised
  by an SMS asking for stage-2 payment 1–3 weeks later.

---

## Order cancellation and recovery

Unpaid orders can be cancelled by the user at:

`/buy/member_buy_finish.php` (我的帳戶 → 購物訂單 → filter by 未付款 / 已取消)

Bibian also auto-cancels long-unpaid orders, typically within ~5 days of
creation. Auto-cancellation behavior:

- The cart entry is gone.
- The order vanishes from `進行中訂單`. It may remain visible under the
  全部 / 已取消 filter.
- 1-time coupons that were locked to that order are **not always
  released**. They may stay in `不可使用` state. Treat 1-time coupons
  as expended after a single failed attempt.

After an order auto-cancels, the user can re-add the same item from
scratch — selectors and flow are identical.

---

## Known quirks summary

| # | Observation | Impact |
| --- | --- | --- |
| 1 | Pressing `Enter` to submit URL without being logged in causes a silent login redirect that locks the renderer (CDP `Runtime.evaluate` timeouts > 45 s). | Always confirm `Hi {name}` visible before URL submission. |
| 2 | Cart row metadata scraping is non-deterministic per source URL pattern. | Have a manual-fill fallback ready. |
| 3 | Visual radios are styled spans, not native `<input>`. | Click the `<label>` ancestor, not the visible circle, to flip state. |
| 4 | All four radios reset on re-entering `cart_order_new.php`. | Re-pick on every entry. |
| 5 | Coupon modal's "no usable coupon" message reflects **threshold filtering**, not wallet emptiness. | Verify against `/member_coupon.php` directly when in doubt. |
| 6 | One-time coupons are consumed at modal-confirm time. | Do not select a 1-time coupon until close to a confirmed-good payment. |
| 7 | Profile fields (身分證字號, 出生日期) being missing causes `訂單付款` to silently redirect to `member_edit.php?type=basic`. | Pre-flight the profile before clicking `訂單付款`. |
| 8 | TapPay Agent Wallet virtual card BIN 87xx historically failed the SDK Luhn/cardType check; this appears resolved as of 2026-04-28. | Always re-verify `getTappayFieldsStatus().canGetPrime` before clicking 確認付款. |
| 9 | Member profile asks for 身分證字號. Agent must NOT fill this. | Hand control to the user explicitly. |
| 10 | `confirm_付款` button gives no visible error when SDK blocks. | Use SDK status, not UI feel, as ground truth. |

---

## Selectors reference (cheat sheet)

```text
# Daigou search input (homepage and /buy/)
input[name="jp_keyword"]

# Cart row index 0
input[name="name_m[0]"]
input[name="style_m[0]"]
input[name="price_[0]"]
input[name="num_[0]"]
input[name="chk_0"]

# Cart submit
a[href="javascript:send('<merchantKey>')"]   # merchantKey from row's source domain

# Order page radio groups
input[name="partial_bid"][value="1"]              # 缺貨繼續購買
input[name="price_decide"][value="price_decide"]  # 漲幅 5% 內可購買
input[name="transit_inter_bid"][value="air"]     # 空運
input[name="tw-delivery"][value="超商取貨"]        # 全家取貨

# Order page agreement
input[type="checkbox"]   # filter by label containing "同意" or "服務條款"

# Order page submit
button:contains("訂單付款")    # use innerText match in JS

# Payment method selector
button:contains("前往刷卡")

# Credit card form
window.TPDirect            # SDK global
window.TPDirect.card.getTappayFieldsStatus()
button:contains("確認付款")

# 3DS (E.Sun)
input[type="radio"][name=...]  # OTP vs Wallet App
input[type="radio"]            # 4-letter identifier codes
input[type="text"]             # OTP password input
button:contains("送出")
button:contains("取消驗證")
```

---

## Output contract addendum

When reporting on a Bibian flow, include these fields in addition to the
shared output contract:

- `bibian_order_id` — the 9-digit order number visible on the payment
  page once `訂單付款` is clicked
- `stage_1_amount_twd`
- `expected_stage_2_components` — list of expected charges (international
  freight, domestic delivery, customs)
- `coupon_applied` and `coupon_state` (`fresh` / `consumed-locked` /
  `unusable-threshold`)
- `pickup_method` (`超商取貨 (全家門市)` / `宅配` / `面交自取`)
- `three_ds_path` (`otp` / `wallet_app` / `none-needed`)