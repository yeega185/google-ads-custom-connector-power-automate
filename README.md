# Google Ads Custom Connector for Power Automate

Power Automate 自訂連接器，串接 Google Ads API，以 GAQL（Google Ads Query Language）查詢 Campaign、AdGroup、Keyword 廣告成效資料，作為 Google Ads 廣告異常監控自動化系統的核心元件。

A Power Automate Custom Connector that integrates with the Google Ads API using GAQL (Google Ads Query Language) to query Campaign, AdGroup, and Keyword performance data — the core component of the Google Ads anomaly monitoring automation system.

---

## 專案結構 / Project Structure

```
google-ads-custom-connector/
├── connector/
│   ├── apiDefinition.swagger.json   # Swagger 2.0 連接器定義檔 / Swagger 2.0 connector definition
│   └── apiProperties.json           # 連接器屬性與 OAuth 設定 / Connector properties and OAuth settings
├── mock/
│   ├── campaign-today.json          # 單日 Campaign 成效範例資料 / Sample single-day campaign data
│   ├── campaign-yesterday.json      # 昨日 Campaign 成效範例資料 / Yesterday's campaign data
│   ├── campaign-daybefore.json      # 前前日 Campaign 成效範例資料 / Day-before-yesterday data
│   ├── campaign-cumulative.json     # 累積花費範例資料 / Cumulative cost sample data
│   └── keyword-cpa-14days.json      # 近 14 天 Keyword CPA 範例資料 / 14-day keyword CPA sample
├── docs/
│   ├── 01_系統規劃文件_GoogleAds異常監控自動化.md  # 系統規劃文件 / System design document
│   ├── 02_流程規劃文件_GoogleAds異常監控自動化.md  # 流程規劃文件 / Flow design document
│   ├── oauth-setup.md               # Google OAuth 2.0 憑證設定指南 / Google OAuth 2.0 setup guide
│   └── deployment-guide.md          # 部署至 Power Automate 指南 / Power Automate deployment guide
├── flow/
│   └── GoogleAds_AbnormalMonitor_Daily.json  # Power Automate Flow 定義（Mock 版）/ Flow definition (mock version)
└── README.md
```

---

## 支援的 API 操作 / Supported API Operations

本 Connector 採用**單一端點設計**，透過 GAQL 查詢語法靈活指定所需欄位，不需為每種資料另設端點。
This connector uses a **single-endpoint design** — the GAQL query string determines what data is returned, eliminating the need for separate endpoints per data type.

| Operation ID | 端點 / Endpoint | 說明 / Description |
|---|---|---|
| `GoogleAdsSearch` | `POST /v18/customers/{customerId}/googleAds:search` | 執行 GAQL 查詢，依查詢內容回傳 Campaign / AdGroup / Keyword 成效資料 / Execute GAQL query; returns Campaign, AdGroup, or Keyword metrics depending on the query |

**對應監控條件與 GAQL 範例 / Monitoring Conditions and GAQL Examples:**

| 監控條件 / Condition | SELECT 欄位 / Fields |
|---|---|
| COST_ZERO | `campaign.id, campaign.name, campaign.start_date, metrics.cost_micros` |
| BUDGET_OVERSPEND | `campaign.id, campaign.name, campaign.start_date, campaign.end_date, campaign_budget.amount_micros, metrics.cost_micros` |
| CTR_FLUCTUATION | `campaign.id, campaign.name, metrics.ctr, segments.date` |
| SEM_HIGH_CPA | `campaign.id, ad_group.id, ad_group.name, ad_group_criterion.keyword.text, ad_group_criterion.keyword.match_type, metrics.cost_per_conversion, metrics.conversions, metrics.cost_micros` |

---

## 快速開始 / Quick Start

### 前置需求 / Prerequisites

- Google Ads 帳戶（具 API 存取權限）/ Google Ads account with API access
- Google Cloud Platform 專案 / Google Cloud Platform project
- Power Automate Premium 授權（Custom Connector 需要）/ Power Automate Premium license
- Google Ads Developer Token（[申請說明](docs/oauth-setup.md) / [Application guide](docs/oauth-setup.md)）

### 安裝步驟 / Installation

1. 取得 OAuth 憑證，詳見 [docs/oauth-setup.md](docs/oauth-setup.md)
   Get OAuth credentials — see [docs/oauth-setup.md](docs/oauth-setup.md)

2. 匯入 Connector 定義，詳見 [docs/deployment-guide.md](docs/deployment-guide.md)
   Import connector definition — see [docs/deployment-guide.md](docs/deployment-guide.md)

3. 建立 Connection（綁定 OAuth 憑證）
   Create a Connection by binding your OAuth credentials

4. 在 Power Automate Flow 中使用此 Connector
   Use this Connector in your Power Automate Flow

---

## 安全性注意事項 / Security

> ⚠️ **請勿將以下資訊提交至此 Repo / Never commit the following to this repo:**

- `client_secret.json` / OAuth client secret file
- Refresh Token
- Google Ads Developer Token
- Customer ID / Login Customer ID
- 任何含真實客戶資料的設定檔 / Any configuration files containing real client data

上述資訊請使用 **Azure Key Vault** 或 **Power Automate 環境變數**管理。
Store all credentials in **Azure Key Vault** or **Power Automate Environment Variables**.

`.gitignore` 已預設排除常見的憑證檔案格式。
`.gitignore` is pre-configured to exclude common credential file formats.

---

## 關於 Mock 資料設計 / About the Mock Data Design

### 為什麼使用自建 JSON 而非直接呼叫 API？/ Why self-built JSON instead of live API calls?

> **Google Ads API 不提供測試用 Token 或沙箱環境。**
> 即使申請到 Developer Token，在通過 Google 審核前僅能存取測試帳戶（Test Account），且測試帳戶的資料極為有限，無法模擬真實的 Campaign 成效狀態（如累積花費超標、CTR 劇烈波動、關鍵字 CPA 異常等）。
>
> **Google Ads API does not provide test tokens or a sandbox environment.**
> Developer Tokens are restricted to Test Accounts before passing Google's review process. Test Account data is too limited to simulate realistic anomaly scenarios (e.g., budget overspend, CTR fluctuation, high-CPA keywords).

因此，本 Flow 在測試階段以 **Compose Action** 內嵌符合 API 回傳格式的 JSON 資料（即 `*_MockData` 步驟），取代實際的 Custom Connector API 呼叫，完整驗證所有監控邏輯與 Email 通知格式。

The Flow therefore uses **Compose Actions with embedded JSON** (`*_MockData` steps) that match the exact API response schema, replacing live Custom Connector calls during testing. This allows complete validation of all monitoring logic and email notification formatting.

`mock/` 目錄提供這些 MockData 的獨立 JSON 參考檔，用途為：
The `mock/` directory provides standalone JSON reference files for these MockData values, used for:

- 開發階段的 Flow 邏輯驗證 / Flow logic validation during development
- 新成員熟悉 Google Ads API 回傳格式 / Onboarding new team members to understand API response structure
- 未來正式串接 API 時的 Schema 對照 / Schema reference when connecting to the live API

所有範例資料已去識別化，不含任何真實客戶資訊。Customer ID 均使用虛構數值。
All sample data is anonymized and contains no real client information. Customer IDs use fictional values.

---

## 版本紀錄 / Changelog

| 版本 / Version | 日期 / Date | 說明 / Description |
|---|---|---|
| v1.0.0 | 2026-06-01 | 初版，單一端點 GoogleAdsSearch，支援 4 種監控條件，Google Ads API v18 / Initial release, single-endpoint GoogleAdsSearch, supporting 4 monitoring conditions, Google Ads API v18 |

---

## 相關文件 / Related Documents

- [01_系統規劃文件](docs/01_系統規劃文件_GoogleAds異常監控自動化.md)
- [02_流程規劃文件](docs/02_流程規劃文件_GoogleAds異常監控自動化.md)
- [Google Ads API 官方文件 / Official Docs](https://developers.google.com/google-ads/api/docs/start)
