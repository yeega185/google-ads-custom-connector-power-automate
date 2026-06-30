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
| `GoogleAdsSearch` | `POST /v23/customers/{customerId}/googleAds:search` | 執行 GAQL 查詢，依查詢內容回傳 Campaign / AdGroup / Keyword 成效資料 / Execute GAQL query; returns Campaign, AdGroup, or Keyword metrics depending on the query |

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

### Custom Connector 連線狀態 / Connector Status

本專案的 Custom Connector（`GoogleAdsSearch`）已完成 OAuth 2.0 授權，並使用 Google Ads **測試帳號（Test Account）** 進行開發與連線驗證，確認 Connector 串接、查詢語法、回傳格式皆正常運作。Test Developer Token 對真實 Customer ID 查詢時，Google Ads API 會回傳 401 認證錯誤（`DEVELOPER_TOKEN_INVALID`）。

The Custom Connector (`GoogleAdsSearch`) has been fully authorized via OAuth 2.0 and verified against a Google Ads **Test Account** during development, confirming the connector wiring, query syntax, and response format all work as expected. Querying a real Customer ID with a Test Developer Token returns a 401 authentication error (`DEVELOPER_TOKEN_INVALID`).

### 為什麼上傳的 Flow 使用 MockData？/ Why does the uploaded Flow use MockData?

> **安全性考量：本 Repo 不包含任何真實憑證。**
> Developer Token、OAuth Client Secret、Customer ID、Refresh Token 等敏感資訊不得上傳至公開 Repository。
>
> **Security: This repo contains no real credentials.**
> Sensitive values such as Developer Token, OAuth Client Secret, Customer ID, and Refresh Token must not be committed to a public repository.

因此，`flow/` 目錄提供的 Flow 定義採用 **MockData 版本**——以 Compose Action 內嵌符合 API 回傳格式的 JSON，取代真實 Custom Connector 呼叫。所有監控邏輯、Email 格式、CSV 附件均與正式版完全一致，可直接匯入執行並驗證流程。

The `flow/` directory contains the **MockData version** — Compose Actions with embedded JSON matching the exact API response schema replace live Connector calls. All monitoring logic, email formatting, and CSV output are identical to the production version and can be imported and run immediately without any credentials.

`mock/` 目錄提供 MockData 的獨立 JSON 參考檔，用途為：
The `mock/` directory provides standalone JSON reference files for:

- 快速匯入並執行 Flow 而不需要任何 API 憑證 / Running the Flow immediately without any API credentials
- 新成員熟悉 Google Ads API 回傳格式 / Onboarding new team members to the API response structure
- 正式串接時的 Schema 對照 / Schema reference when switching to live API calls

所有範例資料已去識別化，不含任何真實客戶資訊。Customer ID 均使用虛構數值。
All sample data is anonymized and contains no real client information. Customer IDs use fictional values.

### AI 摘要功能 / AI Summary Feature

Flow 中另外串接 **Foundry GPT API Custom Connector**（model: `gpt-5`），在彙整完所有異常後產生一段繁體中文 AI 摘要文字，整合進監控 Email 內文。此 Connector 採用 **API Key** 驗證，與 Google Ads Connector 完全獨立。詳細設計請見 [01_系統規劃文件](docs/01_系統規劃文件_GoogleAds異常監控自動化.md) 第 7A 節，部署步驟請見 [deployment-guide.md](docs/deployment-guide.md)。

The Flow also calls a separate **Foundry GPT API Custom Connector** (model: `gpt-5`) to generate a short Traditional Chinese AI summary after all anomalies are aggregated, which is embedded into the monitoring email. This connector uses **API Key** authentication and is fully independent from the Google Ads connector. See [01_系統規劃文件](docs/01_系統規劃文件_GoogleAds異常監控自動化.md) Section 7A for design details, and [deployment-guide.md](docs/deployment-guide.md) for setup steps.

---

## 版本紀錄 / Changelog

| 版本 / Version | 日期 / Date | 說明 / Description |
|---|---|---|
| v1.2.0 | 2026-06-30 | 新增 Foundry GPT API Custom Connector（AI 摘要功能），Google Ads API 升級至 v23 / Added Foundry GPT API Custom Connector (AI summary feature), upgraded Google Ads API to v23 |
| v1.1.0 | 2026-06-11 | Custom Connector OAuth 授權完成，以測試帳號（Test Account）驗證連線，更新 MockData 說明為安全性考量而非 Token 限制 / OAuth authorization completed, connection verified against a Test Account, updated MockData rationale to security rather than Token limitation |
| v1.0.0 | 2026-06-01 | 初版，單一端點 GoogleAdsSearch，支援 4 種監控條件，Google Ads API v18 / Initial release, single-endpoint GoogleAdsSearch, supporting 4 monitoring conditions, Google Ads API v18 |

---

## 相關文件 / Related Documents

- [01_系統規劃文件](docs/01_系統規劃文件_GoogleAds異常監控自動化.md)
- [02_流程規劃文件](docs/02_流程規劃文件_GoogleAds異常監控自動化.md)
- [oauth-setup.md](docs/oauth-setup.md)
- [deployment-guide.md](docs/deployment-guide.md)
- [Google Ads API 官方文件 / Official Docs](https://developers.google.com/google-ads/api/docs/start)
