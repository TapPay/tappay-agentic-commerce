# tappay-agentic-commerce

> An AI agent skill for end-to-end shopping with TapPay Agent Wallet — works with Claude Code, Claude Cowork, OpenClaw, Manus, and more.  
> 讓 AI Agent 代替你完成整個購物流程 — 從選品、找商家、到結帳付款，一氣呵成。支援 Claude Code、Claude Cowork、OpenClaw、Manus 等主流 Agent 平台。

---

## What is this? / 這是什麼？

`tappay-agentic-commerce` is a **prompt skill** that teaches any AI agent how to complete an entire online shopping task autonomously — finding a product, navigating a supported merchant site, filling in shipping information, and completing payment using a **TapPay Agent Wallet** one-time virtual card.

The skill is distributed as a `.zip` file containing a structured set of prompt files. Any agent platform that can read and follow prompt instructions can use it.

`tappay-agentic-commerce` 是一份 **prompt skill**，讓 AI Agent 學會自動完成整個網路購物任務 — 包含找商品、導航到支援的商家網站、填寫寄送資訊，以及使用 **TapPay Agent Wallet** 的一次性虛擬卡完成付款。

Skill 以 `.zip` 格式發布，內含一組結構化的 prompt 檔案，任何能夠讀取並遵循 prompt 指令的 Agent 平台都能使用。

---

## Requirements / 前置需求

### 1. TapPay Agent Wallet MCP Server

The skill communicates with TapPay's wallet service through an MCP server. Add the following entry to your agent's MCP configuration:

此 skill 透過 MCP server 與 TapPay 錢包服務溝通，請在 Agent 的 MCP 設定中加入以下設定：

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

### 2. Browser control capability

The agent needs to be able to open and control a web browser to navigate merchant checkout pages. See the platform-specific installation section below for details.

Agent 需要能夠開啟並控制網頁瀏覽器，才能操作商家的結帳頁面。請參閱下方各平台的安裝說明。

### 3. TapPay Agent Wallet account

You need a TapPay Agent Wallet account with a bound credit/debit card. The skill will guide your agent through the login flow on first use.

你需要一個已綁定信用卡/金融卡的 TapPay Agent Wallet 帳戶。第一次使用時，skill 會引導 Agent 完成登入流程。

---

## Installation / 安裝方式

### Download / 下載

Download the latest `.zip` from the [Releases](../../releases) page and unzip it. You will get a folder named `tappay-agentic-commerce/` containing the skill files.

從 [Releases](../../releases) 頁面下載最新的 `.zip` 檔案並解壓縮，你會得到一個名為 `tappay-agentic-commerce/` 的資料夾，其中包含所有 skill 檔案。

---

### Claude Cowork

1. Rename the unzipped folder so it ends with `.skill` (e.g. `tappay-agentic-commerce.skill`) — or use the pre-built `.skill` file from Releases if provided.  
   將解壓縮後的資料夾重新命名，使其以 `.skill` 結尾（例如 `tappay-agentic-commerce.skill`）— 或直接使用 Releases 中提供的 `.skill` 檔案。

2. Open **Claude Cowork** desktop app and start a new conversation.  
   開啟 **Claude Cowork** 桌面版並開始新對話。

3. Drag and drop the `.skill` file into the conversation — Claude will offer a **Save Skill** button.  
   將 `.skill` 檔案拖曳進對話視窗 — Claude 會出現 **儲存 Skill** 的按鈕。

4. Click **Save Skill** to install.  
   點選 **儲存 Skill** 完成安裝。

5. Install the official **Claude in Chrome** extension for browser control:  
   安裝官方 **Claude in Chrome** 擴充功能以啟用瀏覽器控制：  
   👉 [chrome.google.com/webstore — Claude](https://chromewebstore.google.com/detail/claude/fcoeoabgfenejglbffodgkkbkcdhcgfn)

> ⚠️ Make sure you install the **official Anthropic extension**, not any third-party alternative.  
> ⚠️ 請確認安裝的是 **官方 Anthropic 擴充功能**，而非第三方替代品。

---

### Claude Code

1. Place the unzipped `tappay-agentic-commerce/` folder inside your project's `.claude/skills/` directory:  
   將解壓縮後的 `tappay-agentic-commerce/` 資料夾放入專案的 `.claude/skills/` 目錄中：

   ```
   your-project/
   └── .claude/
       └── skills/
           └── tappay-agentic-commerce/
               ├── SKILL.md
               └── references/
   ```

2. Add the TapPay MCP server to your `claude_mcp_config.json` or project MCP settings.  
   將 TapPay MCP server 加入 `claude_mcp_config.json` 或專案的 MCP 設定中。

3. In a Claude Code session, reference the skill by name or include the `SKILL.md` content in your prompt:  
   在 Claude Code 工作階段中，透過名稱引用此 skill 或在 prompt 中包含 `SKILL.md` 的內容：

   ```
   Use the tappay-agentic-commerce skill to buy [product] from a supported merchant.
   ```

4. For browser control, ensure your Claude Code environment has a browser automation tool available (e.g. Playwright MCP, Puppeteer, or similar).  
   若需要瀏覽器控制，請確認你的 Claude Code 環境中有可用的瀏覽器自動化工具（例如 Playwright MCP、Puppeteer 等）。

---

### OpenClaw

OpenClaw uses the same `SKILL.md` format. Skills are placed in `~/.openclaw/skills/`.

1. Unzip and move the skill folder to the OpenClaw skills directory:  
   解壓縮後，將 skill 資料夾移至 OpenClaw 的 skills 目錄：

   ```bash
   mv tappay-agentic-commerce ~/.openclaw/skills/
   ```

2. Add the TapPay MCP server to your OpenClaw config (`~/.openclaw/config.json` or via `openclaw config`):  
   在 OpenClaw 設定中加入 TapPay MCP server（`~/.openclaw/config.json` 或透過 `openclaw config`）：

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

3. OpenClaw will automatically discover the skill. Start a new task:  
   OpenClaw 會自動偵測到此 skill，接著開啟新任務：

   ```
   Use tappay-agentic-commerce to buy [product].
   ```

---

### Manus

1. Upload the unzipped `tappay-agentic-commerce/` folder to your Manus agent workspace.  
   將解壓縮後的 `tappay-agentic-commerce/` 資料夾上傳到你的 Manus agent workspace。

2. Configure the TapPay MCP server in Manus's tool/MCP settings panel.  
   在 Manus 的工具 / MCP 設定面板中設定 TapPay MCP server。

3. Start a task by pointing the agent at `SKILL.md` as the primary instruction file.  
   建立任務時，將 `SKILL.md` 指定為主要指令檔案，讓 Agent 開始執行。

---

## How to use / 使用方式

Once installed, just describe what you want to buy. The agent handles the rest.

安裝完成後，只需告訴 Agent 你想買什麼，其餘步驟由 Agent 自動處理。

**Example prompts / 範例指令：**

```
我想買一些幫助睡眠的商品
```
```
幫我在誠品網路書店買一本關於設計思考的書
```
```
I'd like to buy a desk lamp from a supported merchant using TapPay.
```

The agent will:
1. Discover supported merchants and suggest options
2. Verify your TapPay wallet is ready (login if needed)
3. Open the merchant site in a browser and find the product
4. Fill in your shipping information
5. Select credit card payment and obtain a one-time virtual card
6. Complete checkout, including 3D Secure OTP verification if required

Agent 會：
1. 搜尋支援的商家並提供選項
2. 確認 TapPay 錢包就緒（如需要會引導你登入）
3. 在瀏覽器開啟商家網站並找到商品
4. 填寫寄送資訊
5. 選擇信用卡付款並取得一次性虛擬卡
6. 完成結帳，包含必要時的 3D 安全驗證 OTP

---

## Supported merchants / 支援商家

Supported merchants are those registered in the **TapPay Agent Wallet** platform. The agent will discover and show you the current list during a shopping session.

支援的商家為已在 **TapPay Agent Wallet** 平台上登記的商家，Agent 會在購物過程中顯示目前可用的商家清單。

The following merchants have detailed checkout playbooks included in this skill:

以下商家在此 skill 中包含完整的結帳流程 playbook：

| | Merchant | Description | Domain |
|---|----------|-------------|--------|
| 📖 | **誠品 eslite** | 挑選最有質感的週末讀物與生活良品 | [eslite.com](https://www.eslite.com) |
| 🇯🇵 | **比比昂 Bibian** | 零時差代標日本最夯的限量模型或露營裝備 | [bibian.co.jp](https://www.bibian.co.jp) |
| ⛺️ | **AsiaYo** | 訂好下一次的森林系露營或特色旅宿 | [asiayo.com](https://asiayo.com/zh-tw/) |
| 🍻 | **FunNow** | 即時預訂今晚下班後的按摩或居酒屋聚會 | [myfunnow.com](https://www.myfunnow.com/zh-tw) |

For other TapPay-supported merchants not listed above, the agent will apply a generic checkout strategy and learn from each run.

對於上表以外的 TapPay 支援商家，Agent 會使用通用結帳策略，並從每次執行中學習。

---

## Credential persistence / 登入狀態保存

To avoid logging in to TapPay every session, the skill saves wallet credentials to a local file:

為了避免每次工作階段都需要重新登入 TapPay，skill 會將錢包憑證儲存到本機檔案：

```
<your workspace folder>/.secrets/tappay-agent-wallet.json
```

This file contains your `agentUuid`, `accessToken`, and `refreshToken`. It is **never** uploaded anywhere — it stays on your machine. Keep it private and do not commit it to version control (add it to `.gitignore`).

此檔案包含你的 `agentUuid`、`accessToken` 和 `refreshToken`。此檔案**永遠不會**上傳到任何地方，只存在你的電腦上。請妥善保管此檔案，不要將它提交到版本控制系統（建議加入 `.gitignore`）。

---

## Security notes / 安全注意事項

- **Virtual cards are one-time use.** Each checkout uses a freshly generated virtual card that expires in 30 minutes and can only be used for a single transaction.  
  **虛擬卡是一次性的。** 每次結帳都會產生一張全新的虛擬卡，有效期 30 分鐘且只能用於單筆交易。

- **The agent never stores your card number.** The virtual card is used immediately and then discarded from memory.  
  **Agent 不會儲存你的卡號。** 虛擬卡使用後立即從記憶體中清除。

- **3D Secure OTP is always handled by you.** When a merchant requires 3D Secure verification, the agent will ask you to provide the OTP code — it never tries to intercept it.  
  **3D 安全 OTP 一律由你自行輸入。** 當商家需要 3D 安全驗證時，Agent 會請你提供 OTP 驗證碼，不會嘗試自動攔截。

---

## Skill file layout / Skill 檔案結構

```
tappay-agentic-commerce/
├── README.md                         # This file / 本文件
├── SKILL.md                          # Main skill prompt / 主要 skill 指令
└── references/
    ├── integrated-checklist.md       # Pre-flight checklist for every run
    ├── tappay-payment-iframe.md      # TapPay iframe field protocol
    └── playbook/                     # Merchant-specific checkout playbooks
        ├── eslite.md                 # eslite.com
        ├── bibian.md                 # bibian.co.jp
        ├── asiayo.md                 # asiayo.com
        └── funnow.md                 # myfunnow.com
```

---

## Troubleshooting / 常見問題

**Q: The agent says "wallet tooling not found".**  
**Q: Agent 說「找不到錢包工具」。**  
A: The TapPay MCP server is not configured. Add the MCP server entry shown in the Requirements section above.  
A: TapPay MCP server 尚未設定，請依照前置需求章節加入 MCP server 設定。

**Q: The agent cannot control the browser.**  
**Q: Agent 無法控制瀏覽器。**  
A: Make sure a browser automation tool is installed and active for your platform. For Claude Cowork, install the official Claude in Chrome extension.  
A: 請確認你的平台已安裝並啟用瀏覽器自動化工具。若使用 Claude Cowork，請安裝官方 Claude in Chrome 擴充功能。

**Q: The transaction failed (交易失敗).**  
**Q: 交易失敗了。**  
A: This usually means the virtual card was rejected. The skill will automatically guide you through recovery: re-adding items to cart and obtaining a fresh virtual card. Never reuse the old virtual card.  
A: 通常表示虛擬卡被拒絕。Skill 會自動引導你進行復原流程：重新加入商品到購物車並取得全新的虛擬卡。請勿重複使用舊的虛擬卡。

---

## License / 授權

MIT License. This skill is provided as-is for personal use. The author is not responsible for any transactions made using this skill.

MIT 授權。本 skill 以現況提供，僅供個人使用。作者對於使用本 skill 所產生的任何交易不負任何責任。

---

## Contributing / 貢獻

Pull requests are welcome! If you have a checkout playbook for a new TapPay-supported merchant, add a file under `references/<domain>.md` following the pattern in `references/eslite.md`.

歡迎提交 Pull Request！如果你有新的 TapPay 支援商家的結帳流程，請依照 `references/eslite.md` 的格式，在 `references/<domain>.md` 新增一份商家 playbook。
