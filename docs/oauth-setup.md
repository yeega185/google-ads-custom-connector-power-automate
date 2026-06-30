# Google Ads API OAuth 2.0 設定指南

本文件說明如何取得 Google Ads API 所需的 OAuth 憑證，並設定至 Power Automate Custom Connector。

> 本文件僅涵蓋 **Google Ads Custom Connector** 的 OAuth 2.0 設定。Flow 中另外使用的 **Foundry GPT API Custom Connector**（產生 AI 摘要）採用 **API Key** 驗證，不需要 OAuth，設定方式請見 [deployment-guide.md](deployment-guide.md) 的「部署步驟三：建立 Foundry GPT API Custom Connector」。

---

## 前置條件

- 具備 Google Ads 帳戶（或 MCC 管理帳戶）
- 具備 Google Cloud Platform（GCP）帳戶
- 有申請 Google Ads Developer Token 的權限（需帳戶管理員）

---

## 步驟一：申請 Google Ads Developer Token

1. 登入 [Google Ads](https://ads.google.com)
2. 右上角選擇「工具與設定」→「設定」→「API Center」
3. 填寫申請表，說明用途（Internal automation / monitoring）
4. 初次申請會取得「測試帳號 Token」，僅能存取 Test Account（測試帳戶），無需審核
5. 正式申請需提交使用說明，約 5-10 個工作天審核

> **⚠️ Test Developer Token 對真實 Customer ID 查詢時，Google Ads API 會回傳 401 認證錯誤（`DEVELOPER_TOKEN_INVALID`）。**
> 本專案開發與測試階段使用 Google Ads **測試帳號（Test Account）**，Custom Connector 已可正常查詢測試帳號資料。監控邏輯則以本 Repo 的 MockData 版 Flow 完整驗證。

---

## 步驟二：建立 GCP 專案並啟用 Google Ads API

1. 前往 [Google Cloud Console](https://console.cloud.google.com)
2. 建立新專案（例：`sem-automate-monitor`）
3. 左側選單 → 「API 與服務」→「程式庫」
4. 搜尋 **Google Ads API** → 啟用

---

## 步驟三：建立 OAuth 2.0 憑證

1. GCP Console → 「API 與服務」→「憑證」
2. 點擊「建立憑證」→「OAuth 用戶端 ID」
3. 應用程式類型選擇 **Web 應用程式**
4. 名稱填入：`PowerAutomate-GoogleAds-Connector`
5. 授權重新導向 URI 加入以下兩條（缺任何一條都會出現 `redirect_uri_mismatch`）：
   ```
   https://global.consent.azure-apim.net/redirect
   https://global.consent.azure-apim.net/redirect/<connector-logical-name>
   ```
   > `<connector-logical-name>` 可在建立 Connector 後，於 Power Automate 自訂連接器頁面的 URL 中取得。
6. 建立後下載 `client_secret_*.json`

> ⚠️ `client_secret_*.json` 不得上傳至 GitHub。已加入 `.gitignore`。

取得的資訊：
- **Client ID**：格式如 `123456789-abcdefg.apps.googleusercontent.com`
- **Client Secret**：格式如 `GOCSPX-xxxxx`

---

## 步驟四：取得 Refresh Token

使用 [Google OAuth 2.0 Playground](https://developers.google.com/oauthplayground) 取得 Refresh Token：

1. 右上角 ⚙️ 點開設定
2. 勾選「Use your own OAuth credentials」
3. 填入 Client ID 與 Client Secret
4. 左側 Scope 欄位輸入：
   ```
   https://www.googleapis.com/auth/adwords
   ```
5. 點擊「Authorize APIs」→ 登入 Google 帳號並授權
6. 點擊「Exchange authorization code for tokens」
7. 複製 `refresh_token` 的值

> **Refresh Token 不會過期（除非手動撤銷或超過 6 個月未使用）。**

---

## 步驟五：記錄所有憑證資訊

| 憑證項目 | 取得位置 | 範例格式 |
|--------|---------|---------|
| Developer Token | Google Ads API Center | `XXXXXXXX_XXXXXX` |
| Client ID | GCP OAuth 憑證 | `123456-abc.apps.googleusercontent.com` |
| Client Secret | GCP OAuth 憑證 | `GOCSPX-XXXXXXXXX` |
| Refresh Token | OAuth Playground | `1//XXXXXXXXXXXXXXXXXX` |
| Customer ID | Google Ads 帳戶首頁 | `123-456-7890`（去掉 dash 後為 `1234567890`） |
| Login Customer ID | MCC 帳戶首頁 | 同上格式（僅限 MCC 架構需要） |

> **請將上述資訊存放至 Azure Key Vault，不得明文儲存於設定檔或 Flow 中。**

---

## 步驟六：驗證憑證（測試用）

可用 curl 快速驗證 Access Token 是否有效：

```bash
# 1. 用 Refresh Token 取得 Access Token
curl -X POST https://oauth2.googleapis.com/token \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "refresh_token=YOUR_REFRESH_TOKEN" \
  -d "grant_type=refresh_token"

# 2. 用 Access Token 呼叫 Google Ads API
curl -X POST \
  https://googleads.googleapis.com/v23/customers/CUSTOMER_ID/googleAds:search \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "developer-token: YOUR_DEVELOPER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT campaign.id, campaign.name FROM campaign LIMIT 5"}'
```

---

## 常見錯誤

| 錯誤訊息 | 原因 | 解決方式 |
|--------|------|---------|
| `DEVELOPER_TOKEN_NOT_APPROVED` | Developer Token 僅為測試版，無法存取正式帳戶 | 申請正式 Token，或改用 Test Account |
| API 回傳 401 `DEVELOPER_TOKEN_INVALID` | Test Developer Token 查詢真實 Customer ID 時的認證錯誤 | 使用 MockData 版 Flow 驗證邏輯，或改用測試帳號的 Customer ID 查詢 |
| `OAUTH_TOKEN_EXPIRED` | Access Token 已過期 | Refresh Token 重新取得（Custom Connector 自動處理） |
| `CUSTOMER_NOT_FOUND` | Customer ID 錯誤 | 確認 Customer ID（去除 dash，純數字） |
| `USER_PERMISSION_DENIED` | 授權帳號無此帳戶存取權 | 確認 Google Ads 帳戶已授權此 OAuth 帳號 |
| `redirect_uri_mismatch` | GCP 的授權重新導向 URI 設定不完整 | 確認 GCP OAuth 憑證已新增兩條 redirect URI（見步驟三） |
