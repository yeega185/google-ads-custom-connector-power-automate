# 01_系統規劃文件_GoogleAds異常監控自動化

---

## 1. 專案目標

本專案旨在設計一套以 Microsoft Power Automate Cloud 為核心的 Google Ads 廣告異常監控自動化系統。

系統每日定時觸發，透過自訂連接器（Custom Connector）串接 Google Ads API，自動取得各 Campaign、AdGroup、Keyword 的廣告成效資料，並依據預先定義的監控規則進行異常判斷。一旦偵測到異常，系統將自動彙整所有異常項目，並透過第二個自訂連接器呼叫 Microsoft Foundry 上的 GPT-5 模型，產生一段繁體中文的 AI 摘要說明，連同異常清單、SEM 差勁關鍵字 CSV 附件一併寄送至指定營運人員，實現無人值守的廣告健診流程。

**核心目標如下：**

| 目標面向 | 說明 |
|--------|------|
| 自動化監控 | 每日定時執行，取代人工定期查核 |
| 規則可維護性 | 監控條件與門檻集中設定，無需修改 Flow |
| 通知集中化 | 四種異常條件彙整為同一封 Email |
| AI 摘要說明 | 透過 Foundry GPT API（GPT-5）自動產生易讀的監控摘要文字 |
| 附件產出 | SEM 差勁關鍵字自動產生 CSV |
| 稽核追蹤 | 所有異常紀錄自動寫入資料表 |
| 擴充彈性 | 架構設計支援未來新增平台與條件 |

---

## 2. 需求摘要

### 2.1 功能性需求

| 編號 | 需求描述 |
|-----|---------|
| F-01 | 每日固定時間觸發廣告成效資料擷取流程 |
| F-02 | 透過 Custom Connector 呼叫 Google Ads API |
| F-03 | 支援多客戶、多 Campaign、多 AdGroup、多 Keyword 的監控設定 |
| F-04 | 監控條件與門檻值集中於資料表維護，不寫死於 Flow |
| F-05 | 判斷四種異常條件（詳見第 9 節） |
| F-06 | 異常判斷結果寫入 AbnormalLogs 資料表 |
| F-07 | 四種異常條件彙整為同一封 Email 通知 |
| F-08 | SEM 差勁關鍵字自動產生 CSV 附件並附於 Email |
| F-09 | 依通知群組設定寄送至對應負責人 |
| F-10 | 異常彙整完成後，呼叫 Foundry GPT API（GPT-5）產生繁體中文 AI 摘要，置於 Email 內文 |

### 2.2 非功能性需求

| 編號 | 需求描述 |
|-----|---------|
| NF-01 | 單一客戶 API 失敗不應中斷其他客戶的監控流程 |
| NF-02 | OAuth Token 不得寫死於 Flow 中 |
| NF-03 | 系統執行失敗需有管理者告警機制 |
| NF-04 | 測試階段可使用 Mock 資料驗證流程，與正式環境分離 |
| NF-05 | 架構需支援未來擴充至其他廣告平台 |
| NF-06 | Foundry GPT API 金鑰（api-key）不得寫死於 Flow 中，需透過 Connector Connection 管理 |
| NF-07 | AI 摘要呼叫失敗不應中斷異常通知流程的其餘部分（Email 仍需寄出） |

---

## 3. 系統架構設計

### 3.1 架構總覽

```
┌─────────────────────────────────────────────────────────┐
│                    Google Ads API                        │
│        (Campaign / AdGroup / Keyword Performance)        │
└───────────────────────┬─────────────────────────────────┘
                        │ HTTPS / OAuth 2.0
                        ▼
┌─────────────────────────────────────────────────────────┐
│         Power Automate Custom Connector（Google Ads）     │
│         (封裝 API 呼叫、統一錯誤處理、回傳標準格式)       │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│            Power Automate Cloud Flow（主流程）           │
│                                                          │
│  ① 定時觸發（Recurrence Trigger）                        │
│  ② 讀取 MonitorRules（監控規則設定表）                   │
│  ③ 逐一呼叫 Custom Connector 取得廣告資料               │
│  ④ 判斷四種異常條件                                     │
│  ⑤ 寫入 AbnormalLogs（異常紀錄表）                      │
│  ⑥ 彙整異常清單 → 組合 AI 請求內容                      │
│  ⑦ 呼叫 Foundry GPT API → 取得 AI 摘要文字               │
│  ⑧ 產生 HTML Email 內容（異常表格 + AI 摘要）            │
│  ⑨ 產生 SEM 差勁關鍵字 CSV 附件                         │
│  ⑩ 寄送 Email 通知（含附件）                            │
│  ⑪ 錯誤處理與管理者告警                                 │
└──────────┬──────────────────────────────┬───────────────┘
           │                              │
           ▼                              ▼
┌─────────────────────────┐   ┌───────────────────────────┐
│   Foundry GPT API        │   │  Microsoft Foundry         │
│   Custom Connector       │──▶│  GPT-5（openai/v1/responses）│
│ (api-key 驗證、單一端點) │   └───────────────────────────┘
└─────────────────────────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
   ┌────────────┐ ┌──────────┐ ┌──────────────┐
   │MonitorRules│ │Abnormal  │ │NotifyGroups  │
   │（規則設定） │ │Logs      │ │（通知群組）   │
   └────────────┘ │（異常紀錄）│ └──────────────┘
                  └──────────┘
```

### 3.2 架構設計說明

**為何選用 Power Automate Cloud？**

Power Automate Cloud 提供原生的排程觸發器、內建的 Email 發送 Action、豐富的資料連接器（SharePoint、Dataverse、Excel 等），並支援自訂連接器擴充，適合作為中型規模監控自動化的主控平台，無需自行維護排程伺服器。

**為何需要 Custom Connector？**

Google Ads API 屬於 RESTful API，需要特定的 OAuth 2.0 流程與 Header 格式，Power Automate 的標準 HTTP Action 雖可發出請求，但缺乏結構化的參數定義、版本管理與重用性。透過 Custom Connector，可將 API 呼叫封裝為可複用的 Action，便於團隊維護與後續擴充。

**為何另外建立 Foundry GPT API Connector？**

異常彙整後的純表格資訊（CampaignID、數值、門檻）對非技術背景的營運人員不夠直覺，需要一段口語化的摘要說明。本系統透過第二個 Custom Connector 串接 Microsoft Foundry 上部署的 GPT-5 模型（`openai/v1/responses` 端點），將異常數量與時間組成提示詞（Prompt），由模型生成 2～3 句繁體中文摘要，插入 Email 內文。獨立建立 Connector 而非直接用 HTTP Action 的理由與 Google Ads Connector 相同：統一管理 `api-key` 驗證、結構化輸入輸出、便於日後替換模型或供應商。

**規則設定為何不寫死於 Flow？**

監控門檻（如預算比例、目標 CPA、最低曝光數）因客戶不同而異，且可能隨業務調整。若寫死於 Flow，每次變更都需修改並重新發布 Flow，風險高且不易追蹤。將規則集中於 MonitorRules 資料表，業務人員可直接維護，Flow 每次執行時動態讀取最新規則。

---

## 4. 資料來源與資料流

### 4.1 資料來源

| 資料來源 | 說明 | 存取方式 |
|--------|------|---------|
| Google Ads API | Campaign / AdGroup / Keyword 成效資料 | Custom Connector（OAuth 2.0） |
| MonitorRules | 監控規則與門檻設定 | SharePoint List / Dataverse |
| NotifyGroups | 通知收件人清單 | SharePoint List / Dataverse |
| AbnormalLogs | 異常紀錄（寫入目標） | SharePoint List / Dataverse |

### 4.2 資料流說明

```
[定時觸發]
    │
    ▼
[讀取 MonitorRules] ──> 取得今日需監控的規則清單（IsActive = true）
    │
    ▼
[For Each Rule] ──> 逐條規則執行以下動作：
    │
    ├──> [呼叫 Custom Connector] ──> 取得對應 Campaign/AdGroup/Keyword 資料
    │         │
    │         ├── 若 API 失敗 ──> 記錄錯誤，繼續下一條規則
    │         │
    │         └── 若成功 ──> 進入異常判斷
    │
    ├──> [判斷異常條件] ──> 依 RuleType 選擇對應判斷邏輯
    │         │
    │         ├── 有異常 ──> 寫入 AbnormalLogs + 加入異常彙整清單
    │         │
    │         └── 無異常 ──> 繼續下一條規則
    │
[彙整所有異常結果]
    │
    ├──> [產生 HTML Email 內容]
    ├──> [產生 SEM 差勁關鍵字 CSV]（若條件 4 有異常）
    │
    ▼
[寄送 Email 通知]（含 CSV 附件）
    │
    ▼
[流程結束 / 記錄執行狀態]
```

---

## 5. 監控規則設計

監控規則採「規則表驅動」設計，Flow 本身不包含任何硬編碼的門檻值或客戶名稱。所有規則存放於 MonitorRules 資料表，Flow 每次執行時讀取 `IsActive = true` 的規則並逐條處理。

### 5.1 規則類型定義

| RuleType | 監控項目 | 對應條件 |
|---------|---------|---------|
| `COST_ZERO` | 廣告花費異常（上線後 48 小時無花費） | 條件 1 |
| `BUDGET_OVERSPEND` | 累積花費超出平均預期 10% | 條件 2 |
| `CTR_FLUCTUATION` | CTR 波動異常（± 20% 變化率） | 條件 3 |
| `SEM_HIGH_CPA` | SEM 差勁關鍵字（連續 14 天 CPA 超標） | 條件 4 |

### 5.2 規則新增與維護流程

1. 業務人員或系統管理者進入 MonitorRules 資料表（SharePoint List 或 Dataverse）
2. 新增一筆規則，填寫 CustomerName、CampaignID、RuleType、門檻值等欄位
3. 設定 `IsActive = true` 以啟用規則
4. 下次 Flow 定時觸發時，即自動納入此規則的監控範圍

---

## 6. 資料表設計

### 6.1 MonitorRules：監控規則設定表

| 欄位名稱 | 資料型態 | 說明 | 範例值 |
|--------|---------|------|-------|
| RuleID | String / GUID | 規則唯一識別碼 | `RULE-001` |
| CustomerName | String | 客戶名稱 | `客戶A` |
| Platform | String | 廣告平台 | `Google Ads` |
| CampaignID | String | Campaign 識別碼 | `123456789` |
| CampaignName | String | Campaign 名稱 | `2026_Brand_TW` |
| AdGroupID | String | AdGroup 識別碼（可空） | `987654321` |
| RuleType | String | 規則類型（見 5.1） | `COST_ZERO` |
| ThresholdValue | Number | 門檻絕對值（依規則類型使用） | `48`（小時） |
| ThresholdPercent | Number | 門檻百分比（如 10%、20%） | `0.1` |
| CompareType | String | 比較方式（GT / LT / BOTH） | `GT` |
| TotalBudget | Number | Campaign 總預算（元） | `300000` |
| CampaignStartDate | Date | Campaign 預定上線日 | `2026-05-01` |
| CampaignEndDate | Date | Campaign 預定結束日 | `2026-06-30` |
| TargetCPA | Number | 目標 CPA（元） | `500` |
| MinimumImpressions | Number | 最低曝光數門檻（防誤判） | `100` |
| NotifyGroup | String | 通知群組代碼（對應 NotifyGroups） | `GROUP-SEM-TW` |
| IsActive | Boolean | 是否啟用此規則 | `true` |

---

### 6.2 NotifyGroups：通知群組設定表

| 欄位名稱 | 資料型態 | 說明 | 範例值 |
|--------|---------|------|-------|
| NotifyGroup | String | 群組代碼 | `GROUP-SEM-TW` |
| Email | String | 收件人 Email | `sem-team@company.com` |
| Role | String | 角色描述 | `SEM 主管` |
| IsActive | Boolean | 是否啟用 | `true` |

> 一個 NotifyGroup 可對應多筆 Email，Flow 查詢時取得同群組的所有啟用收件人。

---

### 6.3 AbnormalLogs：異常紀錄表

| 欄位名稱 | 資料型態 | 說明 | 範例值 |
|--------|---------|------|-------|
| LogID | String / GUID | 紀錄唯一識別碼 | `LOG-20260601-001` |
| CheckTime | DateTime | 本次檢查執行時間 | `2026-06-01 09:00:00` |
| RuleID | String | 對應規則 ID | `RULE-001` |
| CustomerName | String | 客戶名稱 | `客戶A` |
| CampaignID | String | Campaign ID | `123456789` |
| AdGroupID | String | AdGroup ID（可空） | `987654321` |
| Keyword | String | 關鍵字（條件 4 使用） | `品牌關鍵字A` |
| MonitorItem | String | 監控項目名稱 | `CTR_FLUCTUATION` |
| ActualValue | Number | 實際偵測值 | `0.025` |
| ThresholdValue | Number | 門檻值 | `0.02` |
| Status | String | 狀態（ABNORMAL / RESOLVED） | `ABNORMAL` |
| Description | String | 異常說明文字 | `CTR 變化率超過 20%，今日差值 0.005，昨日差值 -0.002` |
| EmailSent | Boolean | 是否已寄送通知 | `true` |
| CreatedTime | DateTime | 紀錄建立時間 | `2026-06-01 09:03:22` |

---

## 7. Custom Connector 設計

### 7.1 用途與設計目標

Custom Connector 作為 Power Automate 與 Google Ads API 之間的標準化橋接層，負責：

- 封裝 OAuth 2.0 授權流程，Flow 無需處理 Token 邏輯
- 統一 API 請求格式與回傳資料結構，降低 Flow 複雜度
- 集中版本管理，若 Google Ads API 版本更新，僅需修改 Connector，不影響 Flow 邏輯
- 便於測試與 Mock 替換

### 7.2 為何不直接使用 HTTP Action？

Power Automate 的內建 HTTP Action 可發出任意 HTTP 請求，但缺點包含：

- Token 管理邏輯需在每個 Flow 中重複處理
- 無法提供結構化的輸入／輸出定義，難以維護
- 多個 Flow 共用時，若 API 規格異動需逐一修改
- 缺乏統一的錯誤處理層

Custom Connector 解決以上問題，是正式環境的最佳實踐選擇。

### 7.3 OAuth 2.0 驗證概念

Google Ads API 使用 OAuth 2.0 Authorization Code Flow，驗證流程如下：

```
[Power Automate] ──> [Custom Connector]
      │
      ▼
[Google OAuth 2.0 Endpoint]
      │ 使用 Client ID + Client Secret + Refresh Token
      ▼
[取得 Access Token]（有效期約 1 小時）
      │
      ▼
[帶入 Access Token 呼叫 Google Ads API]
      │
      ▼
[回傳廣告成效資料]
```

Custom Connector 的 OAuth 設定將 Token 刷新邏輯封裝於 Connector 層，Flow 無需關注 Token 生命週期。

### 7.4 正式環境所需憑證資訊

| 憑證項目 | 說明 | 取得方式 |
|--------|------|---------|
| Google Ads Developer Token | API 呼叫授權 Token | Google Ads 帳戶申請 |
| OAuth Client ID | OAuth 應用程式識別碼 | Google Cloud Console |
| OAuth Client Secret | OAuth 應用程式密鑰 | Google Cloud Console |
| Refresh Token | 用於取得 Access Token | OAuth 授權流程產生 |
| Customer ID | 廣告帳戶 ID | Google Ads 帳戶資訊 |
| Login Customer ID | MCC 帳戶 ID（若有多層帳戶） | MCC 帳戶資訊 |

> 所有憑證不得寫死於 Flow，應存放於 Azure Key Vault 或 Power Automate 環境變數中，並透過 Custom Connector Connection 統一管理。

### 7.5 封裝的 API Action（單一端點設計）

本 Connector 採**單一端點設計**，以 GAQL 查詢語法決定回傳的欄位與資料，不需為各監控條件設立獨立 Action。

| Action 名稱 | 端點 | 輸入參數 | 說明 |
|-----------|------|---------|------|
| `GoogleAdsSearch` | `POST /v18/customers/{customerId}/googleAds:search` | `customerId`（路徑）、`developer-token`（Header）、`login-customer-id`（Header，MCC）、`query`（GAQL 字串） | 執行任意 GAQL 查詢，四種監控條件均使用此 Action，以不同 GAQL 字串指定所需欄位與資料範圍 |

**各監控條件對應 GAQL：**

| 監控條件 | SELECT 欄位（摘要） | FROM 資源與 WHERE 條件 |
|--------|-------------------|----------------------|
| `COST_ZERO` | `campaign.id, campaign.name, campaign.start_date, metrics.cost_micros` | `campaign WHERE segments.date DURING YESTERDAY` |
| `BUDGET_OVERSPEND` | `campaign.id, campaign.name, campaign.start_date, campaign.end_date, campaign_budget.amount_micros, metrics.cost_micros` | `campaign WHERE segments.date DURING YESTERDAY` |
| `CTR_FLUCTUATION` | `campaign.id, campaign.name, metrics.ctr, segments.date` | `campaign WHERE segments.date DURING LAST_7_DAYS` |
| `SEM_HIGH_CPA` | `campaign.id, ad_group.id, ad_group.name, ad_group_criterion.keyword.text, ad_group_criterion.keyword.match_type, metrics.cost_per_conversion, metrics.conversions, metrics.cost_micros` | `ad_group_criterion WHERE segments.date DURING LAST_14_DAYS` |

### 7.6 Power Automate 如何呼叫 Connector

在 Power Automate Flow 中，Custom Connector Action 與一般內建 Action 使用方式相同：

1. 在 Flow 設計畫面新增 Action
2. 選擇已建立的 Custom Connector
3. 選擇對應的 Action（本 Connector 僅有 `GoogleAdsSearch` 一個 Action）
4. 填入輸入參數（可使用動態內容綁定規則表欄位）
5. 後續 Action 可直接引用此步驟的輸出欄位

### 7.7 測試階段 Mock 說明

> **重要提醒：測試階段建議先使用 Mock Response 驗證 Power Automate 流程架構的正確性，確認異常判斷邏輯、Email 產生、CSV 附件、資料寫入等環節均運作正常後，再替換為真實 Custom Connector 呼叫。**

Mock 方式建議：
- 在 Flow 中以「撰寫（Compose）」Action 手動輸入符合 API 回傳格式的靜態 JSON 資料
- 或建立一個 Mock Connector 版本，回傳固定測試資料
- 正式導入時，將 Mock 步驟替換為真實 Custom Connector Action 即可

---

## 7A. Foundry GPT API Connector 設計

### 7A.1 用途

此 Connector 為獨立於 Google Ads Connector 之外的第二個 Custom Connector，負責將異常摘要文字送往 Microsoft Foundry 部署的 GPT-5 模型，取得一段自然語言摘要，用於豐富 Email 內文的可讀性。

### 7A.2 驗證方式

與 Google Ads Connector 採用的 OAuth 2.0 不同，Foundry GPT API 採用 **API Key 驗證**：

| 項目 | 說明 |
|-----|------|
| 驗證類型 | API Key（Header） |
| Header 名稱 | `api-key` |
| Key 來源 | Azure AI Foundry 資源（Resource）的金鑰頁面取得 |
| Host | `{your-foundry-resource}.services.ai.azure.com` |

### 7A.3 封裝的 API Action

| Action 名稱 | 端點 | 輸入參數 | 說明 |
|-----------|------|---------|------|
| `SendMessage` | `POST /openai/v1/responses` | `model`（如 `gpt-5`）、`input`（提示詞字串）、`Content-Type` | 傳送提示詞給 GPT-5，回傳生成內容 |

**回傳結構重點：**

```
{
  "id": "...",
  "status": "...",
  "model": "gpt-5",
  "output": [
    {
      "type": "message",
      "role": "assistant",
      "content": [
        { "type": "output_text", "text": "（AI 生成的摘要文字）" }
      ]
    }
  ],
  "usage": { "input_tokens": ..., "output_tokens": ..., "total_tokens": ... }
}
```

Flow 中以 `first(last(body('傳送訊息')?['output'])?['content'])?['text']` 取出最終文字，寫入 `aiSummary` 變數。

### 7A.4 提示詞（Prompt）設計

Prompt 於 Flow 中以 Compose Action 動態組成，固定要求「繁體中文、2～3 句、專業但易懂、附後續確認方向建議」，並帶入當日日期與異常組數：

```
請用繁體中文，根據以下 Google Ads 廣告異常監控結果，寫一段 2 到 3 句話的精簡摘要說明：
今天監控時間為 {yyyyMMdd}，共發現 {anomalyCount} 組異常情況。
請用專業但易懂的語氣描述監測狀況，並建議後續應確認的方向。
```

### 7A.5 失敗時的處理原則

若 Foundry GPT API 呼叫失敗或逾時，不應阻擋異常 Email 的寄送：建議在 `傳送訊息` 與 `剖析JSON` 步驟加上 Configure run after（失敗時略過），`aiSummary` 退回固定的預設文字（例如「AI 摘要產生失敗，請參考下方異常明細」），確保核心通知功能不受影響。

---

## 8. Power Automate Flow 在系統中的角色

Power Automate Cloud 在本系統中扮演**自動化執行引擎**的角色，負責串接各個系統元件、執行業務規則、產生輸出，但**不負責廣告策略的制定或判斷**。

所有監控邏輯與門檻值均由業務人員定義並存放於 MonitorRules 資料表，Power Automate 的職責是忠實地讀取這些規則並自動化執行。具體職責分述如下：

**1. 定時觸發流程**
使用 Recurrence Trigger，每日固定時間（建議設定為廣告資料通常完整更新後，例如早上 09:00）自動啟動流程，無需人工操作。

**2. 讀取監控規則**
連接 MonitorRules 資料來源，篩選 `IsActive = true` 的規則清單，作為本次執行的監控依據。

**3. 呼叫 Custom Connector**
依據每條規則的 CampaignID、AdGroupID 等設定，動態組合 API 請求參數，呼叫對應的 Custom Connector Action。

**4. 解析 API 回傳資料**
使用 Parse JSON Action 解析 Connector 回傳的廣告成效資料，轉換為後續步驟可使用的動態欄位。

**5. 判斷異常條件**
依據規則的 RuleType，執行對應的判斷運算式（使用 Power Automate 運算式函數），比對實際值與門檻值，產生異常旗標。

**6. 彙整異常結果**
將所有規則的判斷結果集中至一個陣列變數，避免分散處理，確保最終 Email 可一次彙整全部異常。

**7. 呼叫 Foundry GPT API 產生 AI 摘要**
組合包含日期與異常組數的提示詞，呼叫 Foundry GPT API Connector 的 `SendMessage` Action，解析回傳結果取出摘要文字，寫入 `aiSummary` 變數，供 Email 內文使用（詳見第 7A 節）。

**8. 產生 HTML Email 內容**
使用字串組合或 HTML 模板，將異常清單與 AI 摘要轉換為格式化的 Email 內文，包含摘要文字、明細表格與處理建議。

**9. 產生 CSV 附件**
將 SEM 差勁關鍵字清單轉換為 CSV 格式字串，編碼後作為 Email 附件。

**10. 寄送 Email**
查詢 NotifyGroups 取得收件人清單，使用 Office 365 Outlook 或 SMTP Connector 發送 Email（含附件）。

**11. 寫入異常紀錄**
將每筆異常資料寫入 AbnormalLogs 資料表，確保可稽核性與後續追蹤。

**12. 執行錯誤通知**
在各關鍵步驟配置 Configure run after（錯誤後執行）邏輯，若發生例外（包含 Foundry GPT API 呼叫失敗），自動發送管理者告警或退回預設摘要文字。

---

## 9. 異常判斷邏輯

### 9.1 條件 1：廣告花費異常（COST_ZERO）

**判斷目的：** 偵測 Campaign 上線後是否在預定時間內開始花費，防止廣告設定錯誤卻未被發現。

**判斷邏輯：**

```
條件成立（判定異常）：
  目前時間 > (CampaignStartDate + 48 小時)
  AND
  當日 Cost = 0
```

**Power Automate 運算式概念：**

```
addHours(item()?['CampaignStartDate'], 48) < utcNow()
AND
item()?['Cost'] == 0
```

**注意事項：**
- CampaignStartDate 需確認時區一致性（建議統一使用 UTC+8）
- Cost = 0 需確認 API 回傳的是當日資料，而非累計資料

---

### 9.2 條件 2：廣告累積花費異常（BUDGET_OVERSPEND）

**判斷目的：** 偵測 Campaign 的實際累積花費是否超出依投放進度預期的合理範圍。

**判斷邏輯：**

```
每日預期花費 = TotalBudget / 投放總天數
已投放天數 = 今日 - CampaignStartDate（天數）
目前預期累積花費 = 每日預期花費 × 已投放天數
異常門檻 = 目前預期累積花費 × 1.1

條件成立（判定異常）：
  實際累積花費 > 異常門檻
```

**Power Automate 運算式概念：**

```
投放總天數 = dateDifference(CampaignStartDate, CampaignEndDate)
已投放天數 = dateDifference(CampaignStartDate, today())
每日預期花費 = TotalBudget / 投放總天數
預期累積花費 = 每日預期花費 × 已投放天數
異常門檻 = 預期累積花費 × 1.1

ActualCumulativeCost > 異常門檻
```

**注意事項：**
- 若 Campaign 尚未開始（已投放天數 ≤ 0），跳過此條判斷
- 若 TotalBudget 為空，跳過此條判斷並記錄警告

---

### 9.3 條件 3：廣告 CTR 成效異常（CTR_FLUCTUATION）

**判斷目的：** 偵測 CTR 趨勢是否出現異常波動，協助識別廣告效果突然下滑或異常衝高的情況。

**判斷邏輯：**

```
今日差值 = 今日 CTR - 昨日 CTR
前日差值 = 昨日 CTR - 前前日 CTR
差異變化率 = (今日差值 - 前日差值) / ABS(前日差值)

條件成立（判定異常）：
  差異變化率 > 0.2  OR  差異變化率 < -0.2
```

**特殊情況處理：**

| 情況 | 處理方式 |
|-----|---------|
| 前日差值 = 0 | 跳過此次判斷（避免除以零），記錄為「樣本不足以計算」 |
| 今日曝光數 < MinimumImpressions | 跳過此次判斷，記錄為「曝光數不足，結果不具參考性」 |
| 昨日或前前日 CTR 資料不存在 | 跳過此次判斷，記錄缺資料原因 |

**Power Automate 取得三日 CTR 資料：**

- 呼叫 `GoogleAdsSearch`，以 GAQL `WHERE segments.date DURING LAST_7_DAYS` 一次取得近 7 天資料
- 在 Flow 中以 **Filter Array** Action 分別篩選出今日、昨日、前前日的資料列，取各日 CTR 值計算波動率

---

### 9.4 條件 4：SEM 差勁關鍵字（SEM_HIGH_CPA）

**判斷目的：** 找出近兩週內持續高於目標 CPA 的關鍵字，提供 SEM 人員優化依據。

**判斷邏輯：**

```
取得近 14 天 Keyword 成效資料（每個 Keyword 的加總 Cost、Conversions）

Keyword CPA = Cost / Conversions

條件成立（列入差勁清單）：
  Keyword CPA > TargetCPA × 1.2
```

**特殊情況處理：**

| 情況 | 處理方式 |
|-----|---------|
| Conversions = 0 | 若 Cost > 0 且 Conversions = 0，視為 CPA = ∞，直接列入差勁清單；若 Cost = 0，跳過 |
| Keyword 曝光數 < MinimumImpressions | 標記為「樣本不足」，仍列入但以備注提醒 |

**CSV 輸出欄位：**

| 欄位名稱 | 說明 | 範例值 |
|--------|------|-------|
| CampaignID | Campaign 識別碼 | `123456789` |
| CampaignName | Campaign 名稱 | `2026_Brand_TW` |
| AdGroupID | AdGroup 識別碼 | `987654321` |
| AdGroupName | AdGroup 名稱 | `品牌關鍵字組` |
| Keyword | 關鍵字文字 | `品牌A 優惠` |
| TargetCPA | 目標 CPA（元） | `500` |
| ActualCPA | 實際 CPA（元） | `720` |
| CPADiffPercent | CPA 超標百分比 | `44%` |
| Cost | 近 14 天花費（元） | `14400` |
| Conversions | 近 14 天轉換數 | `20` |

---

## 10. Email 通知與 CSV 附件設計

### 10.1 Email 主旨格式

```
[Google Ads 異常監控] 2026/06/01 發現 3 組異常
[Google Ads 異常監控] 2026/06/01 今日監控正常，無異常項目
```

### 10.2 是否在無異常時寄信

**建議做法：當日無任何異常時，寄送「每日正常摘要」通知，而非靜默不寄。**

理由：若無異常完全不寄信，業務人員無法判斷是「確實正常」還是「流程執行失敗」。每日一封正常摘要可作為流程存活證明，降低監控盲點風險。

可依客戶需求於 MonitorRules 或系統設定中加入 `SendDailySummary` 旗標，讓部分客戶選擇無異常不通知。

---

### 10.3 Email 範例內容（實際交付版本，含 AI 摘要）

**主旨：** `[Google Ads 異常監控] 2026/06/30 發現 4 組異常`

實際信件內容由「開頭資訊段」「AI 摘要段（Foundry GPT API 生成，無獨立標題，緊接開頭段落）」「單一異常彙整表格」「SEM 差勁關鍵字明細表（如有觸發）」「結尾」依序組成，**不再採用逐條件分段＋個別處理建議的格式**，AI 摘要已取代逐條建議文字，提供一段涵蓋全部異常的綜合說明：

```
最近監控時間為 '20260630'，發現 4 組異常情況。

今日（20260630）監控共偵測到4組異常，顯示部分投放可能出現流量或轉換等關鍵指標的非預期波動，
甚至有投放中斷風險。建議優先檢視帳戶變更紀錄（預算/出價策略/目標鎖定/時段）、廣告審核與拒登
狀態、轉換與GA4/標記是否正常、落地頁可用性與速度，並比對搜尋字詞與競品出價變化，以快速定位
原因並制定修正方案。

┌──────────────────┬─────────────────────────────────────────────┬──────┬──────────┐
│ 監控類別          │ 監控項目                                      │ 狀態 │ 異常程度 │
├──────────────────┼─────────────────────────────────────────────┼──────┼──────────┤
│ 花費異常 COST ZERO │ CampaignID：1111                             │ 異常 │ 花費 = 0 │
│ 廣告累積花費異常   │ CampaignID：2222 累積花費 NT$120000 超出預算門檻 NT$109996 │ 異常 │ +9%      │
│ CTR 成效異常       │ CampaignID：3333                             │ 異常 │ -666%    │
│ SEM 差勁關鍵字     │ CampaignID：2222 / AdGroup：競品關鍵字組      │ 異常 │ 3 筆     │
└──────────────────┴─────────────────────────────────────────────┴──────┴──────────┘

【SEM 差勁關鍵字明細】
┌──────────┬────────────────┬────────────────┬──────────┬──────────┐
│ Campaign │ AdGroup        │ 關鍵字          │ 比對類型 │ CPA (NT$) │
├──────────┼────────────────┼────────────────┼──────────┼──────────┤
│ 2222     │ 競品關鍵字組    │ 品牌A 競品      │ BROAD    │ 720      │
│ 2222     │ 競品關鍵字組    │ 優惠 折扣       │ PHRASE   │ 1000     │
│ 2222     │ 競品關鍵字組    │ 夏季 特賣 限定  │ EXACT    │ 0        │
└──────────┴────────────────┴────────────────┴──────────┴──────────┘

此信件由系統自動產生，請勿回覆。
```

附件：`SEM差勁關鍵字附件.csv`（UTF-8 with BOM，固定檔名，不含日期戳記）

> AI 摘要段落由 Foundry GPT API（GPT-5）依當日日期與異常組數動態生成，每次執行內容略有不同，上例為實際測試輸出之一。

---

### 10.4 CSV 附件產生方式

Power Automate 中，CSV 產生步驟如下：

1. 使用 **Select** Action 將 SEM 差勁關鍵字陣列轉換為指定欄位的物件陣列
2. 使用 **Create CSV table** Action 將陣列轉換為 CSV 格式字串
3. 在 **Send an email** Action 中，將 CSV 字串以 UTF-8 with BOM 編碼後作為附件加入

> 實際交付版本附件檔名固定為 `SEM差勁關鍵字附件.csv`，未隨日期動態命名。

---

## 11. 錯誤處理與例外情境

| 例外情境 | 可能原因 | 處理方式 | 是否通知管理者 |
|--------|---------|---------|------------|
| Google Ads API 呼叫失敗 | 網路逾時、API 服務中斷、請求格式錯誤 | 記錄錯誤訊息至 AbnormalLogs，標記本次規則為執行失敗，繼續下一條規則 | ✅ 是 |
| Custom Connector 驗證失敗 | Refresh Token 過期、OAuth 設定錯誤 | 整個流程停止執行，立即通知管理者重新驗證 Token | ✅ 是 |
| API 回傳資料為空 | 帳戶無資料、查詢條件無符合項目 | 視為正常（無異常），記錄備注「API 回傳空資料」 | ❌ 否（記錄備注） |
| 單一客戶資料異常，其他客戶需繼續 | 個別帳戶 API 錯誤或資料格式異常 | 使用 For Each 搭配 Configure run after，確保單筆失敗不中斷整體迴圈 | ✅ 是（針對失敗客戶） |
| CSV 產生失敗 | 資料格式異常、欄位遺失 | 記錄錯誤，Email 仍寄送但備注「CSV 附件產生失敗」 | ✅ 是 |
| Email 寄送失敗 | SMTP 服務異常、收件人清單為空 | 記錄錯誤至 AbnormalLogs，標記 EmailSent = false | ✅ 是 |
| Power Automate Flow 執行失敗 | 運算式錯誤、Action 逾時、配額超限 | 設定 Flow Run Failure 通知，自動寄送錯誤報告給管理者 | ✅ 是 |
| Google Ads API 資料延遲 | API 資料尚未完整更新（通常發生於深夜至清晨） | 觸發時間設定於資料更新後（建議早上 09:00），若偵測到資料明顯異常少，記錄警告 | ❌ 否（記錄警告） |
| CTR 樣本數過低 | 該 Campaign/AdGroup 當日曝光數過低 | 跳過 CTR 判斷，記錄「曝光數低於門檻（MinimumImpressions），略過本次判斷」 | ❌ 否 |
| CPA 分母 Conversions = 0 | 關鍵字無轉換，但有花費 | 若 Cost > 0，視為 CPA = ∞，列入差勁清單；若 Cost = 0，跳過 | ❌ 否 |
| Foundry GPT API 呼叫失敗 | API Key 失效、Foundry 資源額度用盡、模型逾時 | `aiSummary` 退回固定預設文字，異常 Email 仍正常寄出（不阻擋核心通知） | ❌ 否（建議記錄警告） |
| Foundry GPT API 回傳格式異常 | 模型回傳內容為空或 `output` 結構不符預期 | 剖析 JSON 失敗時略過摘要段落，Email 僅顯示異常表格 | ❌ 否（建議記錄警告） |

---

## 12. 權限與安全性

### 12.1 OAuth Token 管理

- **不得將 Refresh Token、Client Secret 寫死於 Flow 中**
- 正式環境建議將敏感憑證存放於 **Azure Key Vault**，並透過 Power Automate 的 Azure Key Vault Connector 於執行時動態取得
- Custom Connector 的 Connection 資訊由具備管理權限的帳號設定，其他 Flow 使用者僅能執行，無法檢視憑證內容

### 12.2 Google Ads API 金鑰集中管理

- Developer Token 僅申請一組，集中管理於 Custom Connector Connection 設定中
- 若需多帳戶（MCC）管理，透過 Login Customer ID 參數切換，不需申請多組 Token
- 定期檢視 OAuth Token 有效性，設定 Token 過期提醒機制

### 12.2A Foundry GPT API 金鑰管理

- Foundry 資源的 `api-key` 僅在 Connection 建立時輸入一次，由 Power Automate 加密保存，Flow 內僅以參照方式呼叫，不會明文出現於 Flow 定義中
- 建議定期於 Azure AI Foundry 入口輪替（Rotate）金鑰，並同步更新 Connection
- 若 Foundry 額度或預算有限，建議於 Azure 設定用量警示，避免異常呼叫量產生非預期費用

### 12.3 Power Automate Connection 權限控管

- Custom Connector Connection 的擁有者應為服務帳號（Service Account），而非個人帳號，避免人員異動造成 Connection 失效
- Flow 的編輯權限僅開放給系統負責人，使用者不應擁有修改 Flow 的能力
- SharePoint List（MonitorRules / AbnormalLogs）需設定適當的存取權限，僅允許授權人員讀取異常紀錄

### 12.4 異常紀錄資料保護

- AbnormalLogs 可能包含客戶廣告帳戶資訊（CampaignID、Cost 等），需列為機密資料
- SharePoint 或 Dataverse 資料表需設定欄位層級安全性（Column Level Security），避免未授權人員存取
- 定期稽核存取紀錄

### 12.5 GitHub / 版本控管安全

- **絕對不得將以下資訊上傳至 GitHub 或任何公開 Repository：**
  - Refresh Token
  - OAuth Client Secret
  - Google Ads Developer Token
  - Customer ID / Login Customer ID
  - Foundry GPT API 的 `api-key`
  - Azure AI Foundry 資源名稱（Host，如 `{resource}.services.ai.azure.com`）
  - 任何真實客戶資料
- 若使用設定檔（如 `config.json`），需將其加入 `.gitignore`
- 測試用的 Mock 資料需去識別化（如將 CampaignID 替換為 `TEST-CAMPAIGN-001`，Email 替換為 `test@example.com`）

---

## 13. 測試階段與正式導入差異

### 13.1 測試階段（Mock 環境）

| 項目 | 測試階段做法 |
|-----|------------|
| API 資料來源 | 使用靜態 Mock JSON 取代 Custom Connector 呼叫 |
| 通知收件人 | 固定寄送至測試信箱，不寄送至真實客戶負責人 |
| MonitorRules | 使用測試用規則設定，門檻值設定為容易觸發的數值 |
| AbnormalLogs | 寫入測試用資料表，與正式資料隔離 |
| 敏感憑證 | 使用測試帳號，不使用正式 Google Ads 帳戶 |

### 13.2 正式導入步驟

| 步驟 | 說明 |
|-----|------|
| 1 | 完成 Mock 環境驗證，確認所有流程邏輯正確 |
| 2 | 申請 Google Ads Developer Token（正式版需申請，測試版無限制） |
| 3 | 在 Google Cloud Console 建立 OAuth 應用程式，取得 Client ID / Secret |
| 4 | 完成 OAuth 授權流程，取得 Refresh Token |
| 5 | 在 Custom Connector 設定正式 OAuth 憑證 |
| 6 | 將 Mock JSON 步驟替換為真實 Custom Connector Action |
| 7 | 以正式帳戶小量測試（選擇一個真實 Campaign 驗證資料正確性） |
| 8 | 完整功能測試（四種異常條件均觸發一次，確認 Email 與 CSV 格式） |
| 9 | 設定正式排程，交付給業務人員進行 UAT |
| 10 | UAT 通過後正式上線，監控前幾天的執行紀錄 |

### 13.3 測試建議檢查清單

- [ ] Mock 資料涵蓋四種異常條件（各至少一筆觸發 + 一筆不觸發）
- [ ] Email 主旨正確顯示日期與異常數量
- [ ] 四種異常均彙整於同一封 Email
- [ ] CSV 附件欄位與格式正確
- [ ] Conversions = 0 的邊界情境驗證
- [ ] MinimumImpressions 門檻驗證（低於門檻時略過）
- [ ] 單一規則失敗不影響其他規則執行
- [ ] 管理者告警 Email 在模擬錯誤時正確寄出
- [ ] AbnormalLogs 資料正確寫入

---

## 14. 後續擴充方向

| 擴充方向 | 說明 | 預估複雜度 |
|--------|------|---------|
| 支援更多廣告平台 | 新增 Meta Ads、LINE Ads 的 Custom Connector，MonitorRules 的 Platform 欄位已預留此擴充空間 | 中 |
| 支援 Teams 通知 | 在 Email 通知步驟並行加入 Teams Channel 推送，使用 Microsoft Teams Connector | 低 |
| 支援 Power BI Dashboard | 將 AbnormalLogs 作為 Power BI 資料來源，建立異常趨勢儀表板 | 中 |
| 客戶別異常趨勢分析 | 在 AbnormalLogs 基礎上加入週期性分析 Flow，統計各客戶異常頻率與類型分布 | 中 |
| 自動建立待辦事項 | 偵測到異常時，自動在 Planner 或 Jira 建立任務並指派給對應負責人 | 中 |
| 異常分級（Warning / Critical） | 在 MonitorRules 新增 `SeverityLevel` 欄位，Email 主旨與內文依分級標示，Critical 另發 Teams 即時通知 | 低 |
| 管理者審核與處理狀態追蹤 | 在 AbnormalLogs 新增 `HandlerName`、`HandledTime`、`Resolution` 欄位，配合 Power Apps 建立異常管理介面，讓人員標記處理狀態 | 高 |

---

## 15. 結論

本文件針對 Google Ads 廣告異常監控自動化需求，完整規劃了以 Power Automate Cloud 為核心的系統架構，涵蓋 Custom Connector 設計、監控規則資料表、四種異常判斷邏輯、Email 彙整通知、CSV 附件產出、錯誤處理機制與安全性設計。

系統設計遵循以下核心原則：

- **規則驅動，而非寫死邏輯：** 監控條件與門檻集中於資料表維護，確保系統可維護性
- **彈性擴充：** 架構預留多平台、多通知管道、多監控條件的擴充空間
- **穩健執行：** 單一規則或客戶失敗不中斷整體流程，並具備告警回報機制
- **安全合規：** 憑證集中管理，敏感資料不外洩，測試與正式環境明確分離

本文件可作為工程團隊後續接手實作的規劃依據，建議依照第 13 節的步驟，先完成 Mock 環境驗證，再逐步替換為正式 Google Ads API 資料，降低導入風險。

---

*文件版本：v1.2*
*建立日期：2026/06/01*
*最後更新：2026/06/30（新增 Foundry GPT API Connector 設計（第 7A 節），Flow 角色新增 AI 摘要步驟，Email 範例改為實際交付格式並附 AI 摘要段落）*
*文件狀態：系統開發與功能測試皆已完成*
