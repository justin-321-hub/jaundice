# 新生兒黃疸居家照護衛教智慧客服小幫手

一個以 Vanilla JavaScript 實作的繁體中文衛教聊天機器人，部署於 GitHub Pages，後端由 n8n 工作流程驅動。

---

## 系統架構總覽

```
┌─────────────────────┐        ┌──────────────────────────────┐        ┌─────────────┐
│   使用者 / App        │        │   前端 (GitHub Pages)         │        │  後端 (n8n) │
│                     │        │                              │        │             │
│  瀏覽器直接開啟  ───────────▶  index.html + app.js + CSS     │        │             │
│                     │        │                              │        │             │
│  App WebView 開啟    │        │  1. 產生 clientId (UUID)      │        │             │
│  ┌───────────────┐  │        │  2. 接收 babyId 注入          │        │             │
│  │ setBabyId()   │──────────▶│  3. 使用者送出訊息             │        │             │
│  │ JS Injection  │  │        │  4. POST /api/chat ──────────────────▶ Webhook      │
│  └───────────────┘  │        │  5. 顯示回覆                  │        │  觸發工作流  │
└─────────────────────┘        └──────────────────────────────┘        │  查詢寶寶資料│
                                                                        │  回傳回覆   │
                                                                        └─────────────┘
```

---

## Baby ID 注入機制（JavaScript Injection）

### 背景說明

當本聊天室網頁被嵌入原生 App 的 WebView 中時，App 可以在頁面載入完成後，透過執行 JavaScript 的方式將 **Baby ID** 注入網頁。網頁收到後會在每次送出訊息時一併傳給 n8n，讓後端得以查詢對應的寶寶資料（例如：病歷、黃疸指數歷史紀錄）。

### 運作流程

```
1. App 啟動 WebView 並載入聊天室網頁 URL
         │
2. WebView 觸發 onPageFinished / didFinish 事件（頁面載入完畢）
         │
3. App 執行 JS 注入：window.setBabyId("BABY_ID_VALUE")
         │
4. 網頁將 babyId 寫入記憶體變數 & localStorage
         │
5. 使用者送出任何訊息
         │
6. 前端 POST body 帶入 babyId ──▶ n8n Webhook
         │
7. n8n 以 babyId 查詢資料庫，生成個人化回覆
```

### 前端實作（app.js）

網頁在 `app.js` 中暴露了一個全域函式，供 App 呼叫：

```javascript
window.setBabyId = function (id) {
  if (id && typeof id === "string" && id.trim()) {
    babyId = id.trim();
    localStorage.setItem("fourleaf_baby_id", babyId);
  }
};
```

- 若 `id` 為空字串或非字串，函式會靜默忽略，不改變現有狀態。
- `babyId` 同時寫入 `localStorage`（key: `babyID`），WebView 頁面重整後仍可恢復。
- 若 App 從未注入（例如直接用瀏覽器測試），`babyId` 維持 `null`，聊天室功能完全正常，n8n 端收到的 `babyId` 欄位為 `null`。

### App 端呼叫方式

**iOS（Swift / WKWebView）**

```swift
// 在 webView(_:didFinish:) 委派方法中執行
webView.evaluateJavaScript("window.setBabyId('\(babyId)')") { result, error in
    if let error = error {
        print("setBabyId failed: \(error)")
    }
}
```

**Android（Kotlin / WebView）**

```kotlin
// 在 WebViewClient.onPageFinished() 中執行
webView.evaluateJavascript("window.setBabyId('$babyId')", null)
```

**React Native（WebView）**

```javascript
// 透過 injectedJavaScript 或 injectJavaScript
webViewRef.current.injectJavaScript(`window.setBabyId('${babyId}');`);
```

---

## API 請求格式

前端每次使用者送出訊息，會向後端發出以下請求：

**Endpoint**

```
POST https://jaundice-server.onrender.com/api/chat
```

**Headers**

| Header | 說明 |
|---|---|
| `Content-Type` | `application/json` |
| `X-Client-Id` | 與 body 中的 `clientId` 相同 |

**Request Body**

```json
{
  "text": "寶寶黃疸指數多少需要照光治療？",
  "clientId": "550e8400-e29b-41d4-a716-446655440000",
  "babyId": "BABY_12345",
  "language": "繁體中文",
  "role": "user"
}
```

| 欄位 | 類型 | 說明 |
|---|---|---|
| `text` | string | 使用者輸入的訊息（已過濾問號並做 XSS 處理） |
| `clientId` | string | 瀏覽器端產生的 UUID，用於區分無登入多使用者 session |
| `babyId` | string \| null | App 注入的寶寶識別碼；未注入時為 `null` |
| `language` | string | 固定為 `"繁體中文"` |
| `role` | string | 固定為 `"user"` |

**Response Body（期望格式）**

```json
{
  "text": "根據您寶寶的紀錄，目前黃疸指數為 ..."
}
```

或

```json
{
  "message": "..."
}
```

---

## n8n 後端工作流程建議

### Webhook 接收節點設定

在 n8n Webhook 節點中，可從 request body 取得以下欄位：

```
{{ $json.body.babyId }}      → 寶寶識別碼
{{ $json.body.text }}        → 使用者訊息
{{ $json.body.clientId }}    → Session 識別碼
```

### 建議工作流程

```
Webhook (POST /api/chat)
    │
    ├─ [If] babyId != null
    │       │
    │       └─▶ 查詢資料庫節點（以 babyId 查寶寶資料）
    │               │
    │               └─▶ AI 節點（帶入寶寶資料作為 context + 使用者問題）
    │                       │
    └─ [Else] babyId == null └─▶ AI 節點（通用衛教回覆）
                                    │
                              Respond to Webhook
                              { "text": "..." }
```

### 錯誤處理

前端內建三層重試機制，n8n 端需注意：

| 前端偵測條件 | n8n 應避免 |
|---|---|
| response body 為空 `{}` | 回傳空物件或空字串 |
| body 含有 `search results` + `html` 字串 | 將原始搜尋結果未處理直接回傳 |
| HTTP 500 / 502 / 503 / 504 | 工作流程中斷未 catch |

每種錯誤前端最多重試一次，第二次失敗才顯示錯誤訊息給使用者。

---

## 本地開發

無需安裝任何套件，直接用 HTTP server 啟動：

```bash
python -m http.server 8080
# 或
npx http-server .
```

開啟 `http://localhost:8080`。

---

## 部署

推送至 `main` branch 後，GitHub Actions（`.github/workflows/static.yml`）自動部署至 GitHub Pages。
