# AsiaYo Playbook

AsiaYo (asiayo.com) 是台灣主要的旅宿訂房平台，支援住宿、露營、包棟等多種類型。
本 playbook 記錄 AsiaYo 的 URL 結構、搜尋方式、checkout 流程及已知的操作細節。

---

## URL 結構

### 首頁
```
https://asiayo.com/zh-tw/
```

### 關鍵字搜尋（含日期）✅ 正確格式
```
https://asiayo.com/zh-tw/search/{keyword}/?adult={人數}&check_in_date={YYYY-MM-DD}&check_out_date={YYYY-MM-DD}&quantity=1
```
範例：
```
https://asiayo.com/zh-tw/search/懶人露營/?adult=2&check_in_date=2026-06-12&check_out_date=2026-06-14&quantity=1
```

> ⚠️ **重要**：`/list/` URL 格式會回傳 404，請務必使用 `/search/`。

### 物件頁面（含日期）
```
https://asiayo.com/zh-tw/view/tw/{county}/{property-id}/?adult={人數}&check_in_date={YYYY-MM-DD}&check_out_date={YYYY-MM-DD}&quantity=1
```
範例：
```
https://asiayo.com/zh-tw/view/tw/yilan-county/40499/?adult=2&check_in_date=2026-06-12&check_out_date=2026-06-14&quantity=1
```

county 格式範例：`nantou-county`, `yilan-county`, `taichung-city`, `taoyuan-city`, `miaoli-county`, `hsinchu-county`, `tainan-city`

### 露營專題頁
```
https://asiayo.com/zh-tw/special/tw_feature_camping/
```
此頁無法帶入日期篩選，建議改用 `/search/` 搜尋並帶日期。

### 付款頁
```
https://asiayo.com/zh-tw/payment/{orderId}/?token={token}&currency=TWD
```
付款頁有 **15 分鐘倒數**，務必在倒數結束前完成付款。

---

## 搜尋流程

### 方法 1：直接用 URL（推薦）
直接構造 `/search/` URL，帶入關鍵字與日期，可立即看到有空房的搜尋結果。

### 方法 2：從首頁搜尋
1. 前往 `https://asiayo.com/zh-tw/`
2. 點擊搜尋欄，輸入關鍵字（如「懶人露營」）
3. 輸入後，搜尋欄會彈出自動完成下拉清單
4. 直接按 `Enter` 搜尋，會跳轉至 `/search/` 結果頁並自動帶入日期（若已有日期 context）

### 搜尋結果頁注意事項
- 頁面顯示「X間住宿（含Y個房間）可供預訂入住」
- 結果使用虛擬捲動（virtual scroll），**只有 viewport 附近的物件才在 DOM 中**
- 若要用 JS 抓取所有物件，需先滾到頁底讓所有項目載入
- 物件連結格式：`a[href*="/zh-tw/view/tw/"]`，配合 `h2/h3` 標題抓名稱
- 頁面載入時可能出現「是否要修改搜尋日期與人數？」彈窗 → 點「略過」關閉，不要點「修改」
- 抓取所有物件建議用 JS：`document.querySelectorAll('a[href*="/zh-tw/view/tw/"]')`，用 Set 去重，讀取 `.innerText` 可取得名稱、評分、價格、位置等完整資訊

---

## 物件頁面

### 房型資訊
- 每個房型區塊包含：名稱、基本入住人數、房型設施、含餐/不含餐、退訂政策、價格、剩餘房數
- 庫存狀態：
  - `僅剩 X 房` / `最後 X 房` → 有空房，可訂
  - `目前無空房` → 無空房，同時會出現「修改日期」按鈕
- 價格單位：TWD，以「1房 / X晚」呈現（非每晚單價）

### 訂房按鈕
- 每個可訂房型右下角有「⚡ 訂房」按鈕
- 使用 `find` tool 搜尋「{房型名稱} 訂房按鈕」可精確定位

---

## Checkout 流程（三步驟）

### Step 1：聯絡資訊
**URL**: `https://asiayo.com/zh-tw/check/?checkInDate=...&checkOutDate=...`

欄位：
- 姓氏、名字（已登入帳號會自動帶入）
- 手機號碼（已登入帳號會自動帶入，顯示「有效的手機號碼」）
- 電子郵件（灰色顯示，自動帶入，無法修改）
- 備用聯絡方式（選填）
- 入住旅客資訊：checkbox「入住是其他人，我只是協助訂房」（預設不勾 = 本人入住）

**操作**：確認資訊正確後，捲動到底部點「下一步」。

### Step 2：詳細資訊

欄位：
- **預計入住時間**（必填）：combobox，用 `form_input` 選擇，格式如 `"15:00"`
- **住客人數**：自動帶入，顯示用
- **特殊需求**（選填）：文字輸入框，可跳過
- **使用優惠**（選填）：「套用優惠」按鈕，可跳過
- **發票/收據**：預設電子發票，checkbox「開立公司統一編號」選填

**注意事項（必勾，共 2 個 checkbox）**：
1. 「我已詳閱並同意旅宿入住守則及相關資訊...」
2. 「我已詳閱並同意服務條款和退訂政策，且同意已年滿18歲。」

操作順序：
1. 選擇預計入住時間（詢問使用者幾點抵達）
2. 勾選兩個 checkbox
3. 點「前往付款」按鈕

> ⚠️ **入住時間 combobox 實測注意**：可選時間**依各旅宿設定而異**，必須當下讀取 combobox 選項才能確認。不可假設固定從某個時間開始。若使用者希望的時間不在選單中，需告知實際可選範圍，請使用者重新選擇。

### Step 3：付款
**URL**: `https://asiayo.com/zh-tw/payment/{orderId}/?token={token}&currency=TWD`

付款方式（依頁面順序）：
- **信用卡支付**：支援 VISA / Mastercard / JCB → **TapPay Agent Wallet 使用此選項**
- LINE Pay（可用 LINE POINTS 折抵）
- iPASS MONEY（一卡通）
- AFTEE先享後付（免綁卡）
- 街口支付（部分情況無法使用，顯示說明連結）
- Apple Pay（部分情況無法使用）

---

## TapPay 信用卡付款流程

> ⚠️ 進入付款頁前，先確認 `exchange_virtual_card` 的金額不超過單筆限額（`check_binding` 的 `singleTransactionLimit`）。

1. 選取「信用卡支付」radio button
2. 頁面展開信用卡輸入欄位
3. **AsiaYo 使用 TapPay JS SDK iframe**（已實測確認），iframe src 為 `js.tappaysdk.com/sdk/tpdirect/tappay-field/html/v5.22.0?...`，依照 `references/tappay-payment-iframe.md` 操作
4. 呼叫 `exchange_virtual_card`（amount = 訂單總金額 TWD）
5. **付款頁額外必填欄位**（在 TapPay iframe 區塊之外）：
   - **持卡人英文姓名**（plain HTML input）：用 `find` 搜尋「持卡人英文姓名 input」或 `form_input` 直接填入
   - **手機號碼**（tel input）：用 `find` 搜尋「手機號碼 input field」填入（不含 +886 國碼）
   - **電子郵件**（text input）：用 `find` 搜尋「電子郵件 email input」填入
6. 填入 TapPay iframe 欄位（卡號、有效期限、CVV）：
   - **關鍵**：需先 scroll 讓 iframe 進入 viewport，再用 `getBoundingClientRect()` 取得最新座標
   - 點擊卡號框中央（不要點邊緣），等藍色 focus 框出現後再 type
   - 每填完一欄立即用 `TPDirect.card.getTappayFieldsStatus()` 確認 status === 0
   - 有效期限格式：MMYY（如 "0526"）
7. 點擊「確認付款」按鈕
8. 若跳轉至銀行 3D 驗證頁，等待使用者完成 OTP

### 付款頁欄位填寫順序（建議）
1. 持卡人英文姓名（plain input，最先填，避免後續 focus 問題）
2. 卡號 iframe → 驗證 status === 0
3. 有效期限 iframe → 驗證 status === 0
4. CVV iframe → 驗證 status === 0 且 canGetPrime === true
5. 手機號碼、電子郵件（plain input）
6. 點「確認付款」

---

## 單筆交易限額注意事項

- TapPay Agent Wallet 預設單筆限額為 **TWD 5,000**
- AsiaYo 住宿訂單常見金額 TWD 4,000–20,000+，**很可能超過限額**
- 建議在進入付款頁之前，先用 `check_binding` 確認 `singleTransactionLimit`
- 若訂單金額 > 限額，告知使用者並建議：
  - 改用其他付款方式（LINE Pay、AFTEE 等）
  - 或聯絡 TapPay 提高額度後再付款

---

## 頁面行為與已知細節

| 情況 | 說明 |
|------|------|
| `/list/` URL | 回傳 404，改用 `/search/` |
| 虛擬捲動 | 搜尋結果只有 viewport 附近的物件在 DOM，需滾底才能取得全部 |
| 搜尋結果彈窗 | 可能出現「是否要修改搜尋日期與人數？」彈窗，點「略過」即可關閉 |
| 登入狀態 | Checkout Step 1 會自動帶入帳號資料（姓名、電話、email） |
| 入住時間 | Step 2 必填，需詢問使用者幾點抵達；可選時間依各旅宿設定而異，需當下讀取 combobox 選項確認 |
| 注意事項 checkbox | Step 2 有 2 個必勾，缺一不能點「前往付款」 |
| 付款倒數 | 進入付款頁後 15 分鐘內需完成，否則預訂記錄不保留 |
| 付款頁信用卡欄位 | 為 TapPay JS SDK iframe（已確認），需依 tappay-payment-iframe.md 操作 |
| 付款頁額外欄位 | iframe 外另有：持卡人英文姓名、手機號碼、電子郵件，皆需填入 |
| iframe focus 技巧 | scroll 讓 iframe 進入 viewport → 取新座標 → 點框中央 → 等藍框出現 → type |
| 3D 驗證（玉山銀行） | URL: `acs.esunbank.com.tw`，第一步選驗證方式（預設 OTP）→ 第二步選網頁識別碼（4選1）並輸入 OTP → 送出 |
| 成功頁 URL | `https://asiayo.com/zh-tw/success/{orderId}/?token=...` |
| 退款政策 | 依房型而異，常見：入住前約 2 週可 100% 退款，入住前 1 週 50%，入住當天不退 |
| 自助入住 | 部分民宿為自助式入住，訂單成立後旅宿會另寄自助入住辦法至信箱 |
| 寢具注意 | 部分露營房型需自備睡袋/枕頭棉被，現場無租借，需提醒使用者 |

---

## 搜尋關鍵字建議

### 民宿 / 一般住宿
| 關鍵字 | 說明 |
|--------|------|
| `{城市}民宿` | 如「台南民宿」、「花蓮民宿」，直接帶入日期搜尋 |
| `{城市}住宿` | 涵蓋範圍更廣，含飯店與民宿 |

### 露營
| 關鍵字 | 說明 |
|--------|------|
| `懶人露營` | 免裝備、附帳篷或小木屋的露營體驗 |
| `豪華露營` | glamping，設備齊全 |
| `露營車` | 特色露營車住宿 |
| `包棟露營` | 整區包場 |
| `親子露營` | 適合家庭親子 |
| `寵物露營` | 寵物友善營區 |

---

*最後更新：2026-04-17，整合台南民宿訂房實測（經典雙人房，信用卡 + 玉山 3D 驗證完整流程）*
