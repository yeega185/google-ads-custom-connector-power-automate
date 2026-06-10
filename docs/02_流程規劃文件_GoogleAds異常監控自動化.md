# 02_流程規劃文件_GoogleAds異常監控自動化

---

> **版本說明**
>
> 本文件分為兩個部分：
> - **第 1–11 節：系統設計規劃**——描述完整的目標架構，包含 SharePoint 規則管理、多客戶支援、AbnormalLogs 寫入等。Connector 操作採正確的單一端點設計（`GoogleAdsSearch`）。
> - **第 12 節：實際交付版本（MockData 版）**——由於 Google Ads Developer Token 在審核前僅限存取 Test Account，對真實 Customer ID 呼叫回傳 404，無法以正式 API 進行測試。因此實際交付的 Flow 改以 Compose Action 內嵌 JSON MockData 取代 API 呼叫，完整驗證所有監控邏輯與 Email 格式。MockData 結構與真實 API 回傳格式完全一致，正式環境只需將 MockData 步驟替換為 `GoogleAdsSearch` Action 即可。

---

## 1. 文件目的

本文件詳細說明 Power Automate Cloud Flow 的設計規劃，作為工程團隊實作的依據。內容涵蓋觸發器設計、流程變數、各步驟 Action 設計、異常條件判斷邏輯、Email 組裝、CSV 附件產生、錯誤處理，以及完整的 Flow 結構示意。

---

## 2. 流程概覽

| 項目 | 說明 |
|-----|------|
| Flow 名稱 | `GoogleAds_AbnormalMonitor_Daily` |
| 觸發方式 | 排程（每日 09:00 UTC+8） |
| 資料來源 | Google Ads API（透過 Custom Connector） |
| 規則來源 | SharePoint List：`MonitorRules` |
| 輸出 | Email 通知（含 CSV 附件）+ AbnormalLogs 寫入 |
| 預估執行時間 | 5～15 分鐘（依規則數量與帳戶數量） |

---

## 3. 觸發器設計

**Action 類型：** Recurrence Trigger

| 設定項目 | 值 |
|--------|---|
| Interval | 1 |
| Frequency | Day |
| Time Zone | (UTC+08:00) Taipei |
| Start Time | 09:00:00 |

> **注意：** Google Ads 廣告資料通常於當日凌晨至上午 8:00 完成前一日資料更新，設定 09:00 觸發可確保資料完整性。若需監控當日即時數據，需評估 API 資料延遲風險。

---

## 4. 流程變數設計

所有變數於流程開始時初始化，作用域為整個 Flow Run。

| 變數名稱 | 型態 | 初始值 | 用途 |
|--------|------|-------|------|
| `varTodayDate` | String | `formatDateTime(utcNow(), 'yyyy-MM-dd')` | 今日日期，作為 API 查詢參數 |
| `varYesterdayDate` | String | `formatDateTime(addDays(utcNow(), -1), 'yyyy-MM-dd')` | 昨日日期 |
| `varDayBeforeDate` | String | `formatDateTime(addDays(utcNow(), -2), 'yyyy-MM-dd')` | 前前日日期 |
| `var14DaysAgoDate` | String | `formatDateTime(addDays(utcNow(), -14), 'yyyy-MM-dd')` | 14 天前日期（SEM CPA 用） |
| `varAbnormalList` | Array | `[]` | 儲存所有異常項目（供 Email 彙整用） |
| `varSEMKeywords` | Array | `[]` | 儲存 SEM 差勁關鍵字清單（供 CSV 用） |
| `varAbnormalCount` | Integer | `0` | 異常總數 |
| `varErrorLog` | Array | `[]` | 執行中發生的錯誤紀錄 |
| `varEmailBody` | String | `''` | 組裝中的 Email HTML 內容 |

---

## 5. 主流程步驟

### 步驟總覽

```
[01] Trigger: Recurrence (09:00 daily)
 │
[02] Initialize Variables (7 個變數)
 │
[03] Get Items: MonitorRules (filter: IsActive eq true)
 │
[04] Apply to each: MonitorRule
 │    ├─ [04-A] Switch: RuleType
 │    │    ├─ COST_ZERO        → [Sub-A] 廣告花費異常判斷
 │    │    ├─ BUDGET_OVERSPEND → [Sub-B] 累積花費異常判斷
 │    │    ├─ CTR_FLUCTUATION  → [Sub-C] CTR 波動異常判斷
 │    │    └─ SEM_HIGH_CPA     → [Sub-D] SEM 差勁關鍵字判斷
 │    │
 │    └─ [04-B] Configure run after: 若 Connector 呼叫失敗
 │         └─ Append to varErrorLog + 繼續下一筆
 │
[05] Set Variable: varAbnormalCount = length(varAbnormalList)
 │
[06] Condition: varAbnormalCount > 0
 │    ├─ Yes:
 │    │   ├─ [07] Compose: HTML Email Body
 │    │   ├─ [08] Condition: length(varSEMKeywords) > 0
 │    │   │        ├─ Yes: [09] Create CSV Table → [10] Base64 encode
 │    │   │        └─ No:  skip
 │    │   ├─ [11] Get Items: NotifyGroups (filter by NotifyGroup)
 │    │   └─ [12] Send Email (with/without attachment)
 │    │
 │    └─ No (無異常):
 │        └─ [13] Send daily normal summary (依設定決定是否寄出)
 │
[14] Apply to each: varAbnormalList
 │    └─ Create item: AbnormalLogs
 │
[15] Condition: length(varErrorLog) > 0
 │    └─ Yes: Send error notification to admin
 │
[END]
```

---

### 步驟 03：讀取監控規則

**Action：** `SharePoint - Get items`

| 設定項目 | 值 |
|--------|---|
| Site Address | `https://[company].sharepoint.com/sites/ADOps` |
| List Name | `MonitorRules` |
| Filter Query | `IsActive eq 1` |
| Top Count | `500`（避免 SharePoint 預設 100 筆限制） |

> **設計說明：** 若規則數量超過 500，需實作分頁（`@odata.nextLink`）邏輯。

---

### 步驟 04：逐條規則處理

**Action：** `Control - Apply to each`

- 輸入：步驟 03 的 `value` 輸出
- **Concurrency 設定：** 關閉（Sequential 執行），確保 `varAbnormalList` append 不發生 Race Condition

> **重要：** Power Automate Apply to each 的並行執行（Concurrency ON）會造成陣列變數寫入衝突，必須設為 Sequential。

---

### 步驟 12：寄送 Email

**Action：** `Office 365 Outlook - Send an email (V2)`

| 設定項目 | 值 |
|--------|---|
| To | `join(varNotifyEmails, ';')` |
| Subject | `[Google Ads 異常監控] {varTodayDate} 發現 {varAbnormalCount} 組異常` |
| Body | `varEmailBody`（HTML 格式） |
| Is HTML | `true` |
| Attachments Name | `SEM差勁關鍵字_{varTodayDate}.csv`（條件式加入） |
| Attachments Content | Base64 編碼後的 CSV 字串 |

---

## 6. 異常條件子流程設計

### 6.1 Sub-A：廣告花費異常（COST_ZERO）

**觸發條件：** `RuleType = 'COST_ZERO'`

#### 步驟

```
[A-1] Custom Connector: GoogleAdsSearch
       輸入：
         - customerId:    Rule.CustomerID
         - developer-token: （環境變數）
         - query (GAQL):
             SELECT campaign.id, campaign.name, campaign.start_date,
                    metrics.cost_micros
             FROM campaign
             WHERE segments.date DURING YESTERDAY
               AND campaign.status = 'ENABLED'

[A-2] Parse JSON：解析 API 回傳結果

[A-3] Compose：取得 cost（API 回傳單位為 micros，需除以 1,000,000）
       costValue = div(int(body('Parse_JSON')?['results'][0]['metrics']['costMicros']), 1000000)

[A-4] Compose：計算上線後經過時間（小時）
       hoursElapsed = div(
         sub(ticks(utcNow()), ticks(Rule.CampaignStartDate)),
         36000000000
       )

[A-5] Condition：
       hoursElapsed > Rule.ThresholdValue（預設 48）
       AND costValue == 0

       → True：
           Append to varAbnormalList：
           {
             "RuleID": Rule.RuleID,
             "CustomerName": Rule.CustomerName,
             "CampaignID": Rule.CampaignID,
             "CampaignName": Rule.CampaignName,
             "MonitorItem": "COST_ZERO",
             "ActualValue": 0,
             "ThresholdValue": Rule.ThresholdValue,
             "Description": "Campaign 上線 {hoursElapsed} 小時後仍無花費",
             "Severity": "Critical"
           }

       → False：略過
```

---

### 6.2 Sub-B：廣告累積花費異常（BUDGET_OVERSPEND）

**觸發條件：** `RuleType = 'BUDGET_OVERSPEND'`

#### 步驟

```
[B-1] Custom Connector: GoogleAdsSearch
       輸入：
         - customerId:    Rule.CustomerID
         - developer-token: （環境變數）
         - query (GAQL):
             SELECT campaign.id, campaign.name, campaign.start_date,
                    campaign.end_date, campaign_budget.amount_micros,
                    metrics.cost_micros
             FROM campaign
             WHERE segments.date DURING YESTERDAY
               AND campaign.status = 'ENABLED'

[B-2] Parse JSON：取得累積花費
       actualCumCost = div(costMicros, 1000000)

[B-3] Compose：計算各中間值
       totalDays    = dateDifference(Rule.CampaignStartDate, Rule.CampaignEndDate)
       elapsedDays  = dateDifference(Rule.CampaignStartDate, varTodayDate)
       dailyExpected = div(Rule.TotalBudget, totalDays)
       expectedCum   = mul(dailyExpected, elapsedDays)
       threshold     = mul(expectedCum, add(1, Rule.ThresholdPercent))
                       ← 例：ThresholdPercent = 0.1 → ×1.1

[B-4] Condition：actualCumCost > threshold

       → True：
           Append to varAbnormalList：
           {
             "MonitorItem": "BUDGET_OVERSPEND",
             "ActualValue": actualCumCost,
             "ThresholdValue": threshold,
             "Description": "累積花費 {actualCumCost} 超出預期 {expectedCum} 達 {overspendPct}%",
             "Severity": "Warning"
           }
```

**Edge Case：**
- `totalDays = 0`：跳過，記錄「Campaign 起訖日設定異常」
- `elapsedDays ≤ 0`：Campaign 尚未開始，跳過
- `TotalBudget` 為空：跳過，記錄警告

---

### 6.3 Sub-C：CTR 成效異常（CTR_FLUCTUATION）

**觸發條件：** `RuleType = 'CTR_FLUCTUATION'`

#### 步驟

```
[C-1] Custom Connector: GoogleAdsSearch（單次查詢，一次取回多日資料）
       輸入：
         - customerId:    Rule.CustomerID
         - developer-token: （環境變數）
         - query (GAQL):
             SELECT campaign.id, campaign.name, metrics.ctr,
                    metrics.impressions, segments.date
             FROM campaign
             WHERE segments.date DURING LAST_7_DAYS
               AND campaign.status = 'ENABLED'

       > 使用 LAST_7_DAYS 一次取回多日資料，避免重複呼叫 API，
       > 後續由 Flow 的 Filter Array 依 segments.date 篩出今日、昨日、前前日各自的資料。

[C-2] Filter Array：從結果中篩出今日 / 昨日 / 前前日各一筆
       todayResult     = filter by segments.date = varTodayDate
       yesterdayResult = filter by segments.date = varYesterdayDate
       dayBeforeResult = filter by segments.date = varDayBeforeDate

[C-3] Parse JSON（或直接取值）：
       ctrToday     = first(todayResult)?['metrics']?['ctr']
       ctrYesterday = first(yesterdayResult)?['metrics']?['ctr']
       ctrDayBefore = first(dayBeforeResult)?['metrics']?['ctr']
       impressionsToday = first(todayResult)?['metrics']?['impressions']

[C-4] Condition：impressionsToday < Rule.MinimumImpressions
       → True：跳過，Append 警告到 varErrorLog「曝光數不足，略過 CTR 判斷」
       → False：繼續

[C-5] Compose：計算差值與變化率
       diffToday    = sub(ctrToday, ctrYesterday)
       diffYesterday = sub(ctrYesterday, ctrDayBefore)

[C-6] Condition：diffYesterday == 0
       → True：跳過，記錄「前日差值為 0，無法計算變化率」
       → False：繼續

[C-7] Compose：
       changeRate = div(sub(diffToday, diffYesterday), abs(diffYesterday))

[C-8] Condition：
       changeRate > Rule.ThresholdPercent（預設 0.2）
       OR changeRate < mul(Rule.ThresholdPercent, -1)

       → True：
           Append to varAbnormalList：
           {
             "MonitorItem": "CTR_FLUCTUATION",
             "ActualValue": changeRate,
             "ThresholdValue": Rule.ThresholdPercent,
             "Description": "CTR 變化率 {changeRatePct}%（今日差值 {diffToday}，昨日差值 {diffYesterday}）",
             "Severity": "Warning"
           }
```

---

### 6.4 Sub-D：SEM 差勁關鍵字（SEM_HIGH_CPA）

**觸發條件：** `RuleType = 'SEM_HIGH_CPA'`

#### 步驟

```
[D-1] Custom Connector: GoogleAdsSearch
       輸入：
         - customerId:    Rule.CustomerID
         - developer-token: （環境變數）
         - query (GAQL):
             SELECT campaign.id, campaign.name,
                    ad_group.id, ad_group.name,
                    ad_group_criterion.keyword.text,
                    ad_group_criterion.keyword.match_type,
                    metrics.cost_per_conversion,
                    metrics.conversions, metrics.cost_micros
             FROM ad_group_criterion
             WHERE segments.date DURING LAST_14_DAYS
               AND ad_group_criterion.type = 'KEYWORD'
               AND ad_group_criterion.status = 'ENABLED'

[D-2] Parse JSON：取得 Keyword 列表

[D-3] Apply to each: keyword in results

   [D-3-1] Condition：keyword.metrics.conversions == 0
            → True AND cost > 0：
                 actualCPA = 999999（視為 ∞）
                 備注 = "Conversions = 0，CPA 視為無限大"
            → True AND cost == 0：跳過此關鍵字
            → False：
                 actualCPA = div(keyword.metrics.cost, keyword.metrics.conversions)

   [D-3-2] Compose：
            targetThreshold = mul(Rule.TargetCPA, add(1, 0.2))  ← 超標 20%

   [D-3-3] Condition：actualCPA > targetThreshold

            → True：
                Append to varSEMKeywords：
                {
                  "CampaignID":    keyword.campaign.id,
                  "CampaignName":  keyword.campaign.name,
                  "AdGroupID":     keyword.adGroup.id,
                  "AdGroupName":   keyword.adGroup.name,
                  "Keyword":       keyword.adGroupCriterion.keyword.text,
                  "TargetCPA":     Rule.TargetCPA,
                  "ActualCPA":     actualCPA,
                  "CPADiffPercent": div(sub(actualCPA, Rule.TargetCPA), Rule.TargetCPA),
                  "Cost":          keyword.metrics.cost,
                  "Conversions":   keyword.metrics.conversions
                }

                Append to varAbnormalList（摘要用）：
                {
                  "MonitorItem": "SEM_HIGH_CPA",
                  "Description": "AdGroup {adGroupId} 中有差勁關鍵字，詳見 CSV",
                  "Severity": "Warning"
                }
```

---

## 7. Email 彙整與發送設計

### 7.1 HTML Email 結構

Email 採用 HTML 格式，於步驟 07 的 Compose Action 中組裝。

**組裝邏輯：**

```
varEmailBody = 
  "<html><body>"
  + "<h2>Google Ads 異常監控報告</h2>"
  + "<p>最近監控時間：{varTodayDate}，發現 {varAbnormalCount} 組異常情況</p>"
  + "<table border='1'>..."
  + ForEach varAbnormalList → 組裝 <tr> 行
  + "</table>"
  + "<p>AI 洞察：...</p>"
  + "</body></html>"
```

### 7.2 Email 主旨規則

| 情況 | 主旨格式 |
|-----|---------|
| 有異常 | `[Google Ads 異常監控] {date} 發現 {N} 組異常` |
| 無異常 | `[Google Ads 異常監控] {date} 今日監控正常，無異常項目` |
| 系統錯誤 | `[Google Ads 異常監控][系統錯誤] {date} Flow 執行異常，請確認` |

### 7.3 Email 範例內容（依題目格式）

```
最近監控時間為「2026/06/01」，發現 2 組異常情況，
分別為廣告 CTR 成效異常（CampaignID: 1111）以及 SEM 差勁關鍵字（CampaignID: 2222）。
建議 SEM 操作人員確認廣告素材與出價策略，並檢視附件關鍵字清單。

┌──────────────────┬───────────────────────────────┬──────────────┬──────────────────────┐
│ 監控類別         │ 監控項目                      │ 狀態         │ 嚴重程度             │
├──────────────────┼───────────────────────────────┼──────────────┼──────────────────────┤
│ 廣告 CTR 成效異常 │ CampaignID: 1111              │ 異常 -30%   │ Warning              │
│ SEM 差勁關鍵字   │ CampaignID: 2222, AdGroup: 3333│ 異常        │ 詳見附件 CSV         │
└──────────────────┴───────────────────────────────┴──────────────┴──────────────────────┘

附件：SEM差勁關鍵字詳情_20260601.csv
```

---

## 8. CSV 附件產生設計

### 步驟說明

```
[08-1] Condition：length(varSEMKeywords) > 0

[08-2] Action: Select
       輸入：varSEMKeywords
       對應（Map）：
         "Campaign"       → item()?['CampaignID']
         "AdGroup"        → item()?['AdGroupID']
         "Keyword"        → item()?['Keyword']
         "目標CPA"        → item()?['TargetCPA']
         "實際CPA"        → item()?['ActualCPA']
         "14日平均目標CPA差" → concat(string(mul(item()?['CPADiffPercent'], 100)), '%')
         "Cost"           → item()?['Cost']
         "Conversions"    → item()?['Conversions']

[08-3] Action: Create CSV table
       輸入：步驟 08-2 的 Select 輸出
       Columns：Custom（依上方順序定義欄位名稱）

[08-4] Action: Compose（Base64 encode for attachment）
       公式：base64(outputs('Create_CSV_table'))

[08-5] 將 08-4 輸出作為 Send Email 的 Attachments Content
```

> **編碼注意：** 若 CSV 包含中文字元，需確認 Power Automate Create CSV table 輸出為 UTF-8，必要時加上 BOM（`﻿`）確保 Excel 正確開啟。

---

## 9. 錯誤處理設計

### 9.1 各層級錯誤處理策略

```
┌─────────────────────┬────────────────────────────────────────────────────┐
│ 層級                │ 處理策略                                           │
├─────────────────────┼────────────────────────────────────────────────────┤
│ Custom Connector 失敗│ Configure run after → has failed / has timed out  │
│                     │ → Append 錯誤訊息到 varErrorLog                   │
│                     │ → 繼續下一筆規則（不中斷整體流程）                  │
├─────────────────────┼────────────────────────────────────────────────────┤
│ Parse JSON 失敗     │ Configure run after → has failed                   │
│                     │ → 記錄「API 回傳格式異常」，略過此條規則            │
├─────────────────────┼────────────────────────────────────────────────────┤
│ CSV 產生失敗        │ Configure run after → has failed                   │
│                     │ → Email 仍寄出，但備注「CSV 附件產生失敗」          │
├─────────────────────┼────────────────────────────────────────────────────┤
│ Email 寄送失敗      │ Configure run after → has failed                   │
│                     │ → 標記 EmailSent = false 寫入 AbnormalLogs        │
│                     │ → 傳送告警給管理者                                  │
├─────────────────────┼────────────────────────────────────────────────────┤
│ Flow 整體失敗       │ Flow Run 失敗通知（設定 Flow owner 接收）           │
│                     │ → 管理者收到系統錯誤 Email                          │
└─────────────────────┴────────────────────────────────────────────────────┘
```

### 9.2 管理者告警觸發條件

於流程末端步驟 [15]：

```
Condition: length(varErrorLog) > 0
  → True:
      Send Email to admin:
        Subject: [系統告警] GoogleAds 監控 Flow 發生 {N} 個執行錯誤
        Body: 列出 varErrorLog 所有錯誤項目，包含 RuleID、錯誤訊息、發生時間
```

---

## 10. Flow 完整結構示意

```
Flow: GoogleAds_AbnormalMonitor_Daily
│
├─ [Trigger] Recurrence - 每日 09:00 UTC+8
│
├─ [Initialize] varTodayDate, varYesterdayDate, varDayBeforeDate
├─ [Initialize] var14DaysAgoDate, varAbnormalList, varSEMKeywords
├─ [Initialize] varAbnormalCount, varErrorLog, varEmailBody
│
├─ [SharePoint] Get Items: MonitorRules (IsActive=true)
│
└─ [Apply to each] MonitorRule
     │
     ├─ [Switch] RuleType
     │   ├─ COST_ZERO
     │   │   ├─ [Connector] GetCampaignPerformance (today)
     │   │   ├─ [Parse JSON]
     │   │   ├─ [Compose] costValue, hoursElapsed
     │   │   ├─ [Condition] hoursElapsed>48 AND cost=0
     │   │   └─ [Append] varAbnormalList
     │   │
     │   ├─ BUDGET_OVERSPEND
     │   │   ├─ [Connector] GetCampaignPerformance (cumulative)
     │   │   ├─ [Parse JSON]
     │   │   ├─ [Compose] totalDays, elapsedDays, expectedCum, threshold
     │   │   ├─ [Condition] actualCost > threshold
     │   │   └─ [Append] varAbnormalList
     │   │
     │   ├─ CTR_FLUCTUATION
     │   │   ├─ [Connector] GetCampaignPerformance ×3 (today/yesterday/day before)
     │   │   ├─ [Parse JSON] ×3
     │   │   ├─ [Condition] impressions < MinimumImpressions → skip
     │   │   ├─ [Compose] diffToday, diffYesterday, changeRate
     │   │   ├─ [Condition] changeRate > 0.2 OR < -0.2
     │   │   └─ [Append] varAbnormalList
     │   │
     │   └─ SEM_HIGH_CPA
     │       ├─ [Connector] GetKeywordCPAReport (14 days)
     │       ├─ [Parse JSON]
     │       ├─ [Apply to each] keyword
     │       │   ├─ [Condition] conversions = 0 (handle ∞ CPA)
     │       │   ├─ [Compose] actualCPA, threshold
     │       │   ├─ [Condition] actualCPA > targetCPA × 1.2
     │       │   ├─ [Append] varSEMKeywords
     │       │   └─ [Append] varAbnormalList (summary)
     │       └─ ─
     │
     └─ [Configure run after: failed] → Append varErrorLog, continue
│
├─ [Set Variable] varAbnormalCount = length(varAbnormalList)
│
├─ [Condition] varAbnormalCount > 0
│   ├─ Yes:
│   │   ├─ [Compose] HTML Email Body
│   │   ├─ [Condition] length(varSEMKeywords) > 0
│   │   │   ├─ Yes:
│   │   │   │   ├─ [Select] Map SEM fields
│   │   │   │   ├─ [Create CSV table]
│   │   │   │   └─ [Compose] Base64 encode
│   │   │   └─ No: skip
│   │   ├─ [SharePoint] Get Items: NotifyGroups
│   │   └─ [Outlook] Send Email (with/without attachment)
│   │
│   └─ No:
│       └─ [Outlook] Send daily normal summary
│
├─ [Apply to each] varAbnormalList
│   └─ [SharePoint] Create item: AbnormalLogs
│
└─ [Condition] length(varErrorLog) > 0
    └─ Yes: [Outlook] Send admin error notification
```

---

## 11. 測試驗證清單

### 11.1 Mock 資料測試（測試階段）

| 測試項目 | 測試方式 | 預期結果 |
|--------|---------|---------|
| COST_ZERO 觸發 | Mock cost = 0，上線時間 > 48 小時 | 加入 varAbnormalList，Email 出現條件 1 |
| COST_ZERO 不觸發 | Mock cost > 0 | 不加入清單 |
| BUDGET_OVERSPEND 觸發 | Mock actualCost > expectedCum × 1.1 | 加入 varAbnormalList，Email 出現條件 2 |
| CTR_FLUCTUATION 觸發 | changeRate = -50%（超過 20% 門檻） | 加入 varAbnormalList |
| CTR 曝光不足略過 | Mock impressions < MinimumImpressions | 不加入清單，varErrorLog 有警告記錄 |
| SEM_HIGH_CPA 觸發 | actualCPA > targetCPA × 1.2 | 加入 varSEMKeywords，CSV 正確產生 |
| Conversions = 0 | Mock conversions = 0，cost > 0 | 視為 CPA = ∞，列入差勁清單 |
| 單條規則 API 失敗 | 模擬 Connector 回傳 500 | varErrorLog 有記錄，其他規則繼續執行 |
| 無異常情況 | 所有規則正常 | 寄出正常摘要 Email，AbnormalLogs 無新增 |
| CSV 格式驗證 | SEM 有異常 | CSV 欄位正確、UTF-8 編碼、可用 Excel 開啟 |
| Email 彙整驗證 | 四種異常各觸發一組 | 同一封 Email 包含所有異常，格式正確 |

### 11.2 正式導入後驗證

| 驗證項目 | 說明 |
|--------|------|
| 資料正確性 | 比對 Custom Connector 回傳值與 Google Ads 後台數值是否一致 |
| 時區正確性 | 確認 API 日期區間與實際廣告投放日期對齊（UTC vs UTC+8） |
| 多客戶隔離 | 不同客戶規則獨立執行，異常不互相影響 |
| AbnormalLogs 寫入 | 確認每筆異常均正確寫入，欄位完整 |
| Email 收件驗證 | 確認收件人依 NotifyGroup 正確分配 |

---

---

## 12. 實際交付版本（MockData 版）

### 12.1 背景：為何改為 MockData 實作

Google Ads API 不提供測試 Token 或沙箱環境。Developer Token 在通過 Google 人工審核前，僅限存取 Test Account（測試帳戶）。實際測試時，對真實 Customer ID 發出 API 呼叫，Google Ads API 回傳 **404**（資源不存在），而非正常資料。

在等待 Developer Token 審核期間，為完整驗證所有監控邏輯與 Email 格式，採用以下替代方案：在 Flow 中以 **Compose Action 內嵌 JSON**（`*_MockData` 步驟）取代 Custom Connector 呼叫，所有後續的 Filter Array、條件判斷、Email 組裝、CSV 產生步驟均維持不變。

MockData 的 JSON 結構與真實 Google Ads API 回傳格式完全一致，正式環境只需將各 `*_MockData` Compose 步驟替換為 `GoogleAdsSearch` Action 即可，無需修改任何下游邏輯。

---

### 12.2 實際交付 Flow 架構

**Flow 名稱：** `GoogleAds_AbnormalMonitor_Daily`
**檔案位置：** `flow/GoogleAds_AbnormalMonitor_Daily.json`

```
[Trigger] Recurrence（每日 01:00 UTC = 09:00 UTC+8）
 │
[Initialize] alertContent（String，初始值為 HTML 表格標題列）
[Initialize] anomalyCount（Integer，初始值 0）
 │
[Compose] COST_ZERO_MockData       ← 模擬 COST_ZERO API 回傳
[Filter Array] 篩選陣列             ← costMicros = 0
[Filter Array] 篩選陣列_1           ← startDate 在 48 小時內
[Condition] 條件                   ← length > 0 → 異常
  └─ True: IncrementVariable anomalyCount
           AppendToStringVariable alertContent（加入 COST_ZERO 異常列）
 │
[Compose] BUDGET_OVERSPEND_MockData ← 模擬 BUDGET_OVERSPEND API 回傳
[Apply to each] 套用至各項           ← 逐筆 Campaign 計算
  ├─ [Compose] campaignDays / expectedDaily / actualCumCostNT / elapsedDays / expectedCumNT
  └─ [Condition] 條件_1             ← actualCumCostNT > expectedCumNT × 1.1
       └─ True: IncrementVariable / AppendToStringVariable（BUDGET_OVERSPEND 列）
 │
[Compose] CTR_FLUCTUATION_MockData  ← 模擬 CTR_FLUCTUATION API 回傳（含今日/昨日/前日）
[Filter Array] todayResults         ← segments.date = 今日
[Filter Array] yesterdayResults     ← segments.date = 昨日
[Filter Array] dayBeforeResults     ← segments.date = 前前日
[Condition] 條件_2                  ← today_ctr > yesterday_ctr × 1.2
                                       OR today_ctr < yesterday_ctr × 0.8
  └─ True: IncrementVariable / AppendToStringVariable（CTR_FLUCTUATION 列，含百分比）
 │
[Compose] SEM_HIGH_CPA_MockData     ← 模擬 SEM_HIGH_CPA API 回傳
[Compose] targetCPA = 500
[Filter Array] highCPAKeywords      ← costPerConversion > targetCPA × 1.2
                                       OR (conversions < 1 AND costMicros > 0)
[Condition] 條件_3                  ← length(highCPAKeywords) > 0
  └─ True:
       IncrementVariable / AppendToStringVariable（SEM 摘要列）
       [Select] 選取                ← 整理關鍵字欄位
       [Create CSV table] 建立_CSV_資料表
       AppendToStringVariable 附加至字串變數_5
         ← 關閉主表格，開啟 SEM 明細表格（含欄位標題）
       [Apply to each] 套用至各項_1 ← 逐筆關鍵字加入明細表格列
 │
AppendToStringVariable 關閉明細表   ← 關閉最後一個表格（</table>）
 │
[Condition] 條件_4                  ← anomalyCount > 0
  └─ True: Send Email (V2)
           收件人：your-email@example.com
           主旨：[Google Ads 異常監控] yyyy/MM/dd 發現 N 組異常
           內文：HTML 格式，含異常摘要表格與 SEM 明細表格
           附件：SEM差勁關鍵字附件.csv（UTF-8 with BOM）
```

---

### 12.3 實際交付版與設計規劃版的對照

| 項目 | 設計規劃版（第 1–11 節）| 實際交付版（MockData）|
|-----|----------------------|-------------------|
| 資料來源 | Custom Connector `GoogleAdsSearch` | Compose MockData（相同 Schema）|
| 規則管理 | SharePoint `MonitorRules` | Flow 內硬寫門檻值（targetCPA=500）|
| 流程變數 | 9 個（含日期、清單、錯誤日誌） | 2 個（`alertContent`、`anomalyCount`）|
| 主流程結構 | Apply to each MonitorRule → Switch | 依序 MockData → 條件判斷 |
| 結果寫入 | AbnormalLogs SharePoint List | 無（只寄 Email）|
| 錯誤處理 | varErrorLog + 管理者告警 | 無（MockData 不會失敗）|
| Email 附件 | Base64 encode | UTF-8 BOM + Create CSV table |
| 多客戶支援 | ✅（由 MonitorRules 驅動）| ❌（單一固定 Customer ID）|

---

*文件版本：v1.1*
*建立日期：2026/06/01*
*更新日期：2026/06/10（新增第 12 節實際交付版本說明，修正 Connector 操作設計）*
*對應文件：01_系統規劃文件_GoogleAds異常監控自動化.md*
