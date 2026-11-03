# twlegalrag-search

**給 AI 代理使用的台灣法源檢索與引用防偽技能（Agent Skill）。**

讓 Claude Code、Codex、Antigravity 等 AI 代理，在回答台灣法律問題或起草書狀時，先查證真實法源再作答，並在交付前以程式驗收引用是否捏造。涵蓋：

- 各級法院裁判、司法院釋字、憲法法庭憲判字、不當勞動行為裁決（約 2,276 萬筆，截至 2026-10-30）
- 107 個機關的行政函釋與行政規則（含效力狀態查驗）
- 全國法規資料庫現行法條

> [!IMPORTANT]
> 自 **2026-10-30** 起，TLR 服務需要登入才能查詢。安裝後請先依「[申請 TLR 連線金鑰](#-申請-tlr-連線金鑰)」完成設定，否則所有查詢都會回 HTTP 401。

---

## 目錄

- [與開發者版 tw-legal-rag skill 的關係](#-與開發者版-tw-legal-rag-skill-的關係)
- [本技能的強化之處](#-本技能的強化之處)
- [安裝](#-安裝)
- [申請 TLR 連線金鑰](#-申請-tlr-連線金鑰)
- [使用範例](#-使用範例)
- [與 Legal-Pleading-Suite 的關係](#-與-legal-pleading-suite-的關係)
- [授權與免責聲明](#-授權與免責聲明)

---

## 🤝 與開發者版 tw-legal-rag skill 的關係

本技能的檢索引擎是法律偵探 [@aa0101181514](https://github.com/aa0101181514) 開發的開源專案 **[tw-legal-rag](https://github.com/aa0101181514/tw-legal-rag)**：其 `twlegalrag` CLI 與 TLR 服務（`tlr.dr-legal.com.tw`）負責實際檢索與資料，特此致謝。

開發者在同一 repo 內也提供一個 Agent Skill（[`skills/tw-legal-rag`](https://github.com/aa0101181514/tw-legal-rag/tree/main/skills/tw-legal-rag)）。兩者的定位不同：

- **開發者版**：精簡的判決檢索入門技能，以 `pack` 取得判決 bundle，並規定只能引用 bundle 內的裁判。
- **本技能**：建立在同一 CLI 之上的**完整法律研究與書狀引用工作流程**，擴充檢索範圍、加入程式化引用驗收與法律專業防護規則。

兩者共用同一套 CLI 與同一組核心引用原則（只引用 `allowed_citations`、逐字引述須出自 `fulltext_excerpt`、查無即明說），可以並存，不互相衝突。

---

## ✨ 本技能的強化之處

以下對照以 2026-11-04 開發者版 skill 內容與 `twlegalrag` 2.3.1 實測為準。

| 項目 | 開發者版 `tw-legal-rag` skill | 本技能 `twlegalrag-search` |
|---|---|---|
| 檢索範圍 | 法院判決 | 判決、憲法裁判、勞動裁決 **＋ 行政函釋（`ref-search`、`ref --full`）＋ 現行法條（`law`）** |
| 引用驗收 | 依賴 AI 自律 | **`scripts/check.py` 程式驗收**：退出碼 `0` 通過、`1` 疑似捏造、`2` 待人工複核；未通過不得交付 |
| 登入制（2026-10-30 起） | 文件仍寫「no API key is required」 | **完整的認證設定**：前置檢查以實際查詢確認憑證，遇 401 即停止並提示設定金鑰，嚴禁繞道 |
| bundle 欄位 | 要求排除 `unread_candidates`（2.3.1 的 bundle 無此欄位） | 以 2.3.1 實際欄位 `allowed_citations` 為準 |
| 法源性質辨識 | — | 區分憲法裁判、法院裁判、行政函釋、勞動裁決；**嚴禁將函釋或裁決稱為「判決」** |
| 函釋效力 | — | 依 `status` 處置：現行有效可引用；已廢止、停止適用不得作為現行依據；未驗證須明示 |
| 審級與廢棄裁判 | — | 最高法院優先；`case_history` 顯示已廢棄者不得當作現行前例 |
| 查詢詞建構 | — | 「構成要件 ＋ 法律效果」結構、禁止跨越法律效果邊界（終止≠解除、無效≠撤銷）、口語簡寫案號正規化（如 `北院 112訴100`） |
| 正反雙向查證 | — | 支持主張時同步檢索不利見解（`駁回`、`敗訴`） |
| 見解層自查 | 「字號存在不等於支持主張」 | 同左，並逐一核對見解出處、勝敗方向、是否已廢棄 |
| 執行模式 | 單一流程 | **雙模式**：起草書狀一律走 CLI（可接 `check.py` 驗收、本地留痕）；對話探索時優先 MCP，無 MCP 改用 CLI |
| 隱私 | — | 查詢綁定帳號，**查詢詞禁止含當事人姓名、公司名、身分證號、地址等可識別資訊** |
| 失敗處理 | 查無即明說 | 完整失敗處理表：查無字號、401、MCP 重新登入、`check.py` 未通過、裁判已廢棄等各有處置 |

---

## 📦 安裝

### 1. 安裝 CLI（必要）

需要 Python 3.10 以上。

```bash
pip install -U twlegalrag
```

需 **2.3.1 以上**版本，可用 `pip show twlegalrag` 確認。

### 2. 安裝技能

將本 repo 放進 AI 工具的技能資料夾：

| AI 工具 | 技能資料夾 |
|---|---|
| Claude Code | `~/.claude/skills/` |
| Codex | `~/.codex/skills/` |
| Antigravity | `~/.gemini/config/skills/` |

```bash
cd ~/.claude/skills        # 依你的 AI 工具調整
git clone https://github.com/lexchang53/twlegalrag-search.git
```

不熟悉 git 時，也可以在本頁按「Code → Download ZIP」，解壓後將資料夾命名為 `twlegalrag-search` 放入技能資料夾。

---

## 🔑 申請 TLR 連線金鑰

TLR 自 2026-10-30 起改為登入制。CLI 依下列順序讀取憑證：

1. 環境變數 `TWLEGALRAG_TLR_API_KEY`
2. 設定檔 `~/.twlegalrag/config.toml` 的 `[tlr] api_key`

建議使用**連線金鑰**，設定一次即可長期使用。

### 步驟 1：產生金鑰

1. 開啟 <https://dr-legal.com.tw/mcp/keys>，以法律偵探帳號登入（Google 或 email）。
   - 有多個 Google 帳號時，請在 Google 選帳號畫面確認是你要長期使用的帳號；要換帳號就選「使用其他帳戶」。
2. 在「產生新金鑰」欄位先取個名字（例如 `Home`、`Office`），按「產生」。
3. **金鑰只會顯示這一次**，請立即複製（以 `tlr_` 開頭）。

每個帳號最多 5 把有效金鑰。**建議每台電腦各產生一把**，某台電腦有狀況時只要撤銷那一把。

### 步驟 2：寫入設定檔

建立設定檔（資料夾不存在時請先建立）：

| 系統 | 路徑 |
|---|---|
| Windows | `C:\Users\<使用者名稱>\.twlegalrag\config.toml` |
| macOS / Linux | `~/.twlegalrag/config.toml` |

內容如下，將金鑰貼進引號內：

```toml
[tlr]
# 帳號：<登入帳號>｜金鑰名稱：<名稱>｜建立：<日期>
# base_url = "https://tlr.dr-legal.com.tw"
api_key = "tlr_你的金鑰"
```

> [!WARNING]
> **引號內只放金鑰本身。**若照著範本把 `<在此貼上金鑰>` 的 `<` `>` 一起留下，查詢會回 401。

第一行註解建議填上登入帳號與金鑰名稱，日後有多個帳號時才知道用的是哪一個。

### 步驟 3：驗證

```bash
twlegalrag law 民法 184
twlegalrag search "競業禁止 代償措施" -n 2
```

能回傳民法第 184 條條文與裁判清單即完成。

> [!NOTE]
> `twlegalrag health` 是公開端點，**通過不代表能查詢**；請以上面的實際查詢驗證。

### 仍然回 401 時

- 金鑰是否以 `tlr_` 開頭、前後沒有多餘的空白或 `< >`。
- 金鑰是否已在金鑰頁被撤銷。
- 是否另外設定了環境變數 `TWLEGALRAG_TLR_API_KEY`（它的優先順序高於設定檔）。

### 金鑰安全

- 以金鑰發出的請求視為你本人的使用，**不要轉借或公開**。
- 不要把金鑰貼進 AI 對話、寫進技能資料夾，或提交到 git。
- 金鑰外洩時，到金鑰頁按「撤銷」，再產生新的一把。

### 備案：不使用金鑰

無法使用金鑰時，可改用 tw-legal-rag 官方的裝置授權工具取得暫時 token（約 1 小時過期），設為 `TWLEGALRAG_TLR_API_KEY`。詳見 [tw-legal-rag](https://github.com/aa0101181514/tw-legal-rag)。

### MCP（選用）

若要在對話中使用 MCP：在 Claude 的 Connectors 加入 `https://tlr.dr-legal.com.tw/mcp`，以法律偵探帳號登入授權即可，不需要連線金鑰；連續 30 天未使用需重新登入。

---

## 🧭 使用範例

### 起草書狀（統一 CLI 管線）

> 請依 twlegalrag-search 技能，為勞資爭議準備書狀查證「特休未休工資與加班費」的實務見解、勞動部函釋與勞基法第 38 條。

AI 會依序執行：

```bash
twlegalrag law 民法 184                                        # 確認憑證
twlegalrag search "特休未休工資 加班費" -n 5 --read             # 預覽候選
twlegalrag pack "特休未休工資請求權與加班費計算" -o bundle.json -n 6
twlegalrag ref-search "特別休假 未休日數 工資發給" --authority "勞動部"
twlegalrag ref "勞動條2字第1080130382號" --full                # 全文與效力
twlegalrag law 勞基法 38
python scripts/check.py bundle.json pleading_draft.txt          # 引用驗收
```

`check.py` 退出碼為 `0` 才會交付。

### 對話中查函釋

> 財政部對建物管理維護契約是否應貼印花稅票，有什麼函釋？

AI 會先以 `ref-search` 找出字號，再以 `ref --full` 調閱全文並確認效力狀態，註明發布機關、發文字號與是否現行有效。

### 檢查既有文稿的引用

> 用 bundle.json 檢查我這份答辯狀的引用有沒有問題。

AI 會執行 `scripts/check.py`，逐則回報通過、不在 bundle 內（疑似捏造）或待人工複核。

---

## 🔗 與 Legal-Pleading-Suite 的關係

本技能原為 [Legal-Pleading-Suite](https://github.com/lexchang53/Legal-Pleading-Suite)（台灣訴訟書狀 AI 套件）的一部分，2026-11 起獨立為本 repo，可單獨使用。

Legal-Pleading-Suite 的 `legal-research`（法律問題研究與爭點分析）以本技能為**核心依賴**；使用該套件時請一併安裝本技能。

---

## 📄 授權與免責聲明

本專案採用 **CC BY-NC-SA 4.0**，律師、事務所與個別法律從業者於日常業務中使用不受限制；詳見 [LICENSE.md](LICENSE.md)。

本工具僅供檢索與分析參考，**不構成任何法律意見**。檢索資料來自 TLR 服務，引用前請自行核對裁判與函釋全文。

作者：張清浩律師（lex@lex.idv.tw）
