# 專案部署指南 / Project Deployment Guide

本文件說明 **Google Ads 廣告異常監控自動化系統**的完整部署流程，涵蓋 Power Automate Flow 匯入、Custom Connector 建立，以及為什麼本專案採用 JSON MockData 方式進行驗證。

This guide covers the full deployment of the **Google Ads Anomaly Monitoring Automation System**, including Power Automate Flow import, Custom Connector setup, and the reasoning behind the JSON MockData approach used in this project.

---

## 專案組成 / Project Components

本系統由以下三個部分組成：

| 元件 | 說明 | 檔案位置 |
|-----|------|---------|
| Power Automate Flow | 每日排程執行，判斷異常並發送 Email | `flow/GoogleAds_AbnormalMonitor_Daily.json` |
| Custom Connector | 定義 Google Ads API 的呼叫方式（GAQL 查詢） | `connector/apiDefinition.swagger.json` |
| Mock 資料 | 符合 API 回傳格式的測試用 JSON | `mock/*.json` |

> **本 Repo 提供的 Flow 為 MockData 版本**，所有 Google Ads 資料來源已以 Compose Action 內嵌 JSON 取代真實 API 呼叫。原因詳見下方說明。

---

## 為什麼上傳的 Flow 使用 MockData？/ Why does the uploaded Flow use MockData?

### Custom Connector 已完成連線 / Connector is Live

本專案的 Custom Connector 已完成 OAuth 2.0 授權，並使用 Google Ads **測試帳號（Test Account）** 進行開發與連線驗證，確認 Connector 串接、查詢語法、回傳格式皆正常運作。

The Custom Connector has been fully authorized via OAuth 2.0 and verified against a Google Ads **Test Account** during development, confirming the connector wiring, query syntax, and response format all work as expected.

| 開發階段使用方式 | 說明 |
|--------------|------|
| Test Account 連線 | Flow 中保留一段真實的 `GoogleAdsSearch` 呼叫，連線至測試帳號，驗證 Connector 本身串接正常 |
| Test Account 資料不足 | 測試帳號無真實廣告活動資料，無法模擬「花費超標」「CTR 波動」「CPA 異常」等情境，因此監控邏輯改以 MockData 驗證 |

### 上傳 MockData 版的原因：安全性 / Reason for MockData version: Security

> **本 Repo 為公開 Repository，不包含任何真實憑證。**
> Developer Token、OAuth Client Secret、Customer ID、Refresh Token 等敏感資訊不得上傳至公開 Repository。

因此，`flow/` 目錄提供的 Flow 定義採用 **MockData 版本**：

1. 以 **Compose Action**（撰寫）取代 Custom Connector 呼叫
2. 每個 Compose Action 內嵌符合 Google Ads API 實際回傳格式的 JSON 資料
3. 後續所有邏輯（Filter Array、條件判斷、Email 組裝、CSV 產生）行為與串接真實 API 完全一致

MockData 版可直接匯入並執行，無需任何 API 憑證即可驗證完整流程。

**Flow 中的 MockData 步驟對應：**

| Flow 步驟名稱 | 對應監控條件 | 對應 mock 參考檔 |
|------------|------------|----------------|
| `COST_ZERO_MockData` | 花費為零偵測 | `mock/campaign-today.json` |
| `BUDGET_OVERSPEND_MockData` | 累積花費超標 | `mock/campaign-cumulative.json` |
| `CTR_FLUCTUATION_MockData` | CTR 波動偵測（含今日/昨日/前日） | `mock/campaign-yesterday.json` |
| `SEM_HIGH_CPA_MockData` | 關鍵字 CPA 異常 | `mock/keyword-cpa-14days.json` |

---

## 部署步驟一：匯入 Flow / Import the Flow

### 前置條件
- Power Automate Premium 或 Per-Flow 授權
- Office 365 帳戶（用於發送通知 Email）

### 步驟

1. 登入 [Power Automate](https://make.powerautomate.com)
2. 左側選單 → 「我的流程」
3. 上方「匯入」→「匯入套件（舊版）」
4. 上傳 `flow/GoogleAds_AbnormalMonitor_Daily.json`

   > 若系統要求 `.zip` 格式，請先將 JSON 檔包成 zip 再上傳，或改用「從 JSON 建立」選項。

5. 匯入後，開啟 Flow 確認以下設定：

   | 設定項目 | 預期值 |
   |--------|-------|
   | 觸發器類型 | Recurrence（每日） |
   | 觸發時間 | 每日 09:00 UTC+8（JSON 中為 `01:00Z`） |
   | 收件人 Email | 預設為 `your-email@example.com`，請依實際需求修改 |

6. 開啟 `傳送電子郵件_(V2)` 步驟，確認 Office 365 連線已綁定正確帳號

7. 儲存並手動執行一次，確認 Email 正常送出

---

## 部署步驟二：建立 Custom Connector / Setup Custom Connector

Custom Connector 定義 Google Ads API 的呼叫介面，供未來替換 MockData 使用。**目前的 Flow（MockData 版）不需要此步驟即可執行。**

### 匯入 Swagger 定義

1. 登入 [Power Automate](https://make.powerautomate.com)
2. 左側選單 → 「資料」→「自訂連接器」
3. 右上角「新增自訂連接器」→「匯入 OpenAPI 檔案」
4. 名稱：`GoogleAds-Monitor-Connector`
5. 上傳 `connector/apiDefinition.swagger.json`
6. 點擊「繼續」

### 設定 General（一般）

| 欄位 | 值 |
|-----|---|
| Connector name | `GoogleAds-Monitor-Connector` |
| Host | `googleads.googleapis.com` |
| Base URL | `/` |

### 設定 Security（安全性）

**Authentication type：OAuth 2.0**

| 欄位 | 值 |
|-----|---|
| Identity Provider | Google |
| Client ID | GCP 的 OAuth Client ID |
| Client Secret | GCP 的 OAuth Client Secret |
| Authorization URL | `https://accounts.google.com/o/oauth2/auth` |
| Token URL | `https://oauth2.googleapis.com/token` |
| Refresh URL | `https://oauth2.googleapis.com/token` |
| Scope | `https://www.googleapis.com/auth/adwords` |

> OAuth 憑證取得方式詳見 [oauth-setup.md](oauth-setup.md)

### 確認 Definition（動作定義）

切換至「Definition」頁籤，確認以下 Action 已正確載入：

| Operation ID | 端點 |
|-------------|-----|
| `GoogleAdsSearch` | `POST /v23/customers/{customerId}/googleAds:search` |

本 Connector 採**單一端點設計**，以 GAQL 查詢語法決定回傳資料，不需為每種監控條件設立獨立端點。

### 建立 Connection（OAuth 授權）

1. 左側選單 → 「資料」→「連接」→「新增連接」
2. 搜尋 `GoogleAds-Monitor-Connector`
3. 跳出 Google OAuth 授權畫面，登入具備 Google Ads 存取權的帳號
4. 授權完成

---

## 部署步驟三：建立 Foundry GPT API Custom Connector / Setup Foundry GPT API Connector

本專案另外使用一個獨立的 Custom Connector，呼叫 **Microsoft Foundry GPT API**（model: `gpt-5`），在每次監控完成後產生一段 AI 摘要文字，整合進 Email 內文。此 Connector 與 Google Ads Connector 完全獨立，採用 **API Key 驗證**（非 OAuth）。

### 前置條件

- 一個已部署 GPT-5 模型的 **Azure AI Foundry** 資源
- 該資源的 Endpoint（例如 `https://<your-resource>.services.ai.azure.com`）與 API Key

### 建立 Azure AI Foundry 資源與模型部署

1. 登入 [Azure Portal](https://portal.azure.com)，建立或開啟一個 **Azure AI Foundry** 資源
2. 進入該資源的「模型部署」，部署 `gpt-5` 模型
3. 於資源的「金鑰與端點」頁面，記下：
   - Endpoint（去除 `https://` 與結尾路徑後的主機名稱，例如 `your-resource.services.ai.azure.com`）
   - API Key

### 匯入 Custom Connector

1. 登入 [Power Automate](https://make.powerautomate.com)
2. 左側選單 → 「資料」→「自訂連接器」
3. 右上角「新增自訂連接器」→「從空白建立」或「匯入 OpenAPI 檔案」
4. 名稱：`Foundry GPT API`

### 設定 General（一般）

| 欄位 | 值 |
|-----|---|
| Connector name | `Foundry GPT API` |
| Host | `<your-resource>.services.ai.azure.com`（建立連接時填入實際 Azure AI Foundry 資源主機名稱） |
| Base URL | `/` |

### 設定 Security（安全性）

**Authentication type：API Key**

| 欄位 | 值 |
|-----|---|
| Parameter label | `api-key` |
| Parameter name | `api-key` |
| Parameter location | Header |

### 設定 Definition（動作定義）

新增一個 Action：

| 欄位 | 值 |
|-----|---|
| Summary | 傳送訊息 |
| Operation ID | `SendMessage` |
| 路徑 | `POST /openai/v1/responses` |

**Request Body：**

```json
{
  "model": "string",
  "input": "string"
}
```

**Response（預設回應 Schema）：**

```json
{
  "id": "string",
  "object": "string",
  "status": "string",
  "model": "string",
  "output": [
    {
      "type": "string",
      "role": "string",
      "content": [
        { "type": "string", "text": "string" }
      ]
    }
  ],
  "usage": {
    "input_tokens": 0,
    "output_tokens": 0,
    "total_tokens": 0
  }
}
```

### 建立 Connection（API Key 授權）

1. 左側選單 → 「資料」→「連接」→「新增連接」
2. 搜尋 `Foundry GPT API`
3. 貼上前置步驟取得的 API Key
4. 授權完成

### Flow 中的呼叫方式

Flow 在彙整完所有異常後，依序執行：

1. `組合AI請求內容`（Compose）—— 組成提示詞，內容包含監控日期與異常組數
2. `傳送訊息`（呼叫 `Foundry GPT API` Connector 的 `SendMessage`，`body/model` 固定為 `gpt-5`，`body/input` 為提示詞文字）
3. `剖析JSON`（Parse JSON，依上方 Response Schema 解析）
4. `設定變數`（將 `first(last(body('剖析JSON')?['output'])?['content'])?['text']` 寫入 `aiSummary` 變數，供 Email 內文使用）

若 Foundry API 呼叫失敗或逾時，`aiSummary` 變數會維持初始值「AI摘要產生中」，Email 仍會正常寄出，僅 AI 摘要段落顯示初始值，不影響其餘異常資訊的呈現。

---

## 部署步驟四：正式版替換（未來）/ Production Switch

當有實際客戶帳號可串接後，依以下方式將 MockData 替換為真實 API 呼叫：

| MockData Compose 步驟 | 替換為 | GAQL 查詢 |
|----------------------|-------|---------|
| `COST_ZERO_MockData` | `GoogleAdsSearch` Action | `SELECT campaign.id, campaign.name, campaign.start_date, metrics.cost_micros FROM campaign WHERE segments.date DURING YESTERDAY AND campaign.status = 'ENABLED'` |
| `BUDGET_OVERSPEND_MockData` | `GoogleAdsSearch` Action | `SELECT campaign.id, campaign.name, campaign.start_date, campaign.end_date, campaign_budget.amount_micros, metrics.cost_micros FROM campaign WHERE segments.date DURING YESTERDAY AND campaign.status = 'ENABLED'` |
| `CTR_FLUCTUATION_MockData` | `GoogleAdsSearch` Action | `SELECT campaign.id, campaign.name, metrics.ctr, segments.date FROM campaign WHERE segments.date DURING LAST_7_DAYS AND campaign.status = 'ENABLED'` |
| `SEM_HIGH_CPA_MockData` | `GoogleAdsSearch` Action | `SELECT campaign.id, ad_group.id, ad_group.name, ad_group_criterion.keyword.text, ad_group_criterion.keyword.match_type, metrics.cost_per_conversion, metrics.conversions, metrics.cost_micros FROM ad_group_criterion WHERE segments.date DURING LAST_14_DAYS AND ad_group_criterion.type = 'KEYWORD' AND ad_group_criterion.status = 'ENABLED'` |

替換時，後續的 Filter Array、條件判斷、Email 組裝步驟**完全不需修改**，因為 MockData 的 JSON Schema 與真實 API 回傳格式完全一致。

---

## 開發過程遭遇的限制與解法 / Development Limitations & Solutions

以下記錄實際開發過程中遭遇的關鍵問題，供後續維護或正式部署參考。

### 限制 1：Power Automate Expression 不支援 filter() 語法

**問題：** 原本嘗試在 Condition 或 Compose 的 Expression 欄位中，以 `filter()` 函式直接對 API 結果做篩選：

```
filter(outputs('BUDGET_OVERSPEND')?['body/results'],
  greaterOrEquals(int(item()?['metrics']?['costMicros']), int(item()?['campaignBudget']?['amountMicros']))
)
```

Power Automate 的 Expression 引擎雖然支援 `filter()` 語法，但在 `filter()` 的條件參數裡使用 `item()` 參照時，實際執行會出錯或無法正確迭代每一筆資料。

**解法：** 改用 Power Automate 的 **Filter Array（篩選陣列）Action** 或 **Apply to each（套用至各項）+ Condition（條件）** 組合：

| 監控條件 | 改用方式 |
|--------|---------|
| COST_ZERO | 兩層 Filter Array Action 串接（先篩 costMicros = 0，再篩 startDate 在 48 小時內）|
| BUDGET_OVERSPEND | Apply to each 迴圈 + Condition（在迴圈內計算 expectedCumNT 後判斷）|
| SEM_HIGH_CPA | Filter Array Action（Query 類型，使用 `@or(greater(...), and(...))` 條件）|

---

### 限制 2：Custom Connector UI 識別提供者下拉選單無法載入

**問題：** 在 Power Automate 網頁介面建立 Custom Connector 時，「Security」頁籤的「Identity Provider」下拉選單永遠空白，無論使用 Edge、Chrome 或重新整理都無法載入。嘗試從空白建立或用 Swagger Editor 匯入均失敗。

**根本原因：** Microsoft 識別提供者服務在此網路環境無回應，與操作方式無關。

**解法：** 改用 **pac CLI**（Microsoft 官方 Power Platform 開發者工具），同時匯入 `apiDefinition.swagger.json` 和 `apiProperties.json`，繞過故障的 UI：

```powershell
$pac = "C:\Users\User\AppData\Local\Microsoft\PowerAppsCLI\Microsoft.PowerApps.CLI.2.8.1\tools\pac.exe"

# 登入
& $pac auth create --name MyOrg

# 建立 Connector
& $pac connector create `
  --api-definition-file "connector/apiDefinition.swagger.json" `
  --api-properties-file "connector/apiProperties.json"

# 更新（例如更新 Client Secret）
& $pac connector update `
  --connector-id "<connector-id>" `
  --api-definition-file "connector/apiDefinition.swagger.json" `
  --api-properties-file "connector/apiProperties.json"
```

> ⚠️ Client Secret 僅在執行 pac CLI 時暫時寫入 `apiProperties.json`，執行後立刻清除，絕對不 commit 到 GitHub。

---

### 限制 3：policyTemplateInstances 衝突

**問題：** pac CLI 建立 Connector 時出現 `Ambiguous policy sections defined for policy template 'setheader'` 錯誤。

**根本原因：** `apiDefinition.swagger.json` 已將 `developer-token` 定義為 Header 參數，`apiProperties.json` 又同時有 `setheader` policy 設定同一個 header，兩邊衝突。

**解法：** 清空 `apiProperties.json` 的 `policyTemplateInstances` 陣列，改由 Flow 在呼叫 Action 時直接傳入 `developer-token`。

---

### 限制 4：redirect_uri_mismatch

**問題：** 建立 Connection 授權時，Google 回傳 `400: redirect_uri_mismatch`。

**解法：** 在 GCP → Credentials → OAuth Client → Authorized redirect URIs 新增：
```
https://global.consent.azure-apim.net/redirect
https://global.consent.azure-apim.net/redirect/<connector-logical-name>
```

---

## 常見問題 / Troubleshooting

| 問題 | 原因 | 解決方式 |
|-----|------|---------|
| Flow 匯入後 Email 連線顯示錯誤 | Connection 未綁定或帳號不符 | 重新設定 `傳送電子郵件_(V2)` 的 Office 365 連線 |
| Email 沒有收到 | anomalyCount 為 0，條件_4 不觸發 | MockData 中所有條件均已設計為會觸發，確認 Flow 未被修改 |
| Swagger 匯入失敗 | JSON 格式錯誤 | 使用 [Swagger Editor](https://editor.swagger.io) 驗證格式 |
| OAuth 授權後出現 400 錯誤 | Redirect URI 不符 | 確認 GCP 的 Redirect URI 含 `global.consent.azure-apim.net/redirect` |
| API 呼叫回傳 401 `DEVELOPER_TOKEN_INVALID` | 測試帳號的 Developer Token 對真實 Customer ID 查詢時的認證錯誤 | 先使用 MockData 版 Flow 進行驗證，或改用測試帳號的 Customer ID 查詢 |
| GAQL 語法錯誤回傳 400 | 欄位名稱或 FROM 資源不符 | 至 [Google Ads Query Builder](https://developers.google.com/google-ads/api/fields/v23/overview_query_builder) 驗證語法 |
