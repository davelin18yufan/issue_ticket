# Google Sheets 至 Discord 整合系統設計文件

## 1. 系統概述

### 1.1 目標
建立一個自動化系統，當使用者在 Google Sheets 中提交資料後，透過 OpenAI API 處理資料，然後發送到 Discord 頻道，並整合現有的內部 issue 系統。

### 1.2 核心流程
```
Google Sheets 表單提交 → Apps Script 觸發 → HTTP API 呼叫 → .NET 後端處理 → OpenAI 資料整合 → 建立 Issue → Discord 通知
```

## 2. 架構設計

### 2.1 系統組件

#### 2.1.1 Google Apps Script 層
- **職責**：資料收集、預處理、API 呼叫
- **觸發條件**：表單提交或工作表變更
- **輸出**：標準化的 JSON 資料

#### 2.1.2 .NET Framework API 層
- **職責**：接收請求、資料驗證、流程編排
- **端點**：`POST /api/sheets/webhook`
- **處理模式**：非同步處理

#### 2.1.3 業務邏輯層
- **職責**：OpenAI 整合、Issue 建立、Discord 通知
- **模式**：Background Service 或 Task.Run

### 2.2 資料流向圖
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Google Sheets  │───▶│  Apps Script    │───▶│   .NET API      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                                               ┌────────▼────────┐
                                               │ Background      │
                                               │ Processing      │
                                               └─────────────────┘
                                                        │
                        ┌─────────────────┐            │
                        │   Discord API   │◀───────────┤
                        └─────────────────┘            │
                                                        │
                        ┌─────────────────┐            │
                        │   OpenAI API    │◀───────────┤
                        └─────────────────┘            │
                                                        │
                        ┌─────────────────┐            │
                        │ Internal Issue  │◀───────────┘
                        │    System       │
                        └─────────────────┘
```

## 3. 技術規格

### 3.1 Google Apps Script

#### 3.1.1 觸發器設定
```javascript
// 觸發器類型選擇
- onFormSubmit: 表單提交時觸發
- onChange: 工作表內容變更時觸發
```

#### 3.1.2 資料處理邏輯
- 資料驗證與清理
- 格式標準化
- 敏感資訊過濾

#### 3.1.3 HTTP 請求規格
- **方法**：POST
- **URL**：`https://your-api-domain.com/api/sheets/webhook`
- **Content-Type**：`application/json`
- **驗證**：API Key 或 Bearer Token

### 3.2 .NET Framework API

#### 3.2.1 端點設計
```csharp
[HttpPost]
[Route("api/sheets/webhook")]
public async Task<IActionResult> HandleSheetsWebhook([FromBody] SheetsWebhookRequest request)
```

#### 3.2.2 請求模型
```csharp
public class SheetsWebhookRequest
{
    public string EventType { get; set; }
    public DateTime Timestamp { get; set; }
    public string SpreadsheetId { get; set; }
    public Dictionary<string, object> FormData { get; set; }
    public string UserEmail { get; set; }
    public string ApiKey { get; set; }
}
```

#### 3.2.3 回應模型
```csharp
public class SheetsWebhookResponse
{
    public bool Success { get; set; }
    public string Message { get; set; }
    public string ProcessingId { get; set; }
    public Dictionary<string, object> Metadata { get; set; }
}
```

### 3.3 非同步處理架構

#### 3.3.1 處理流程
1. **立即回應**：快速返回 202 Accepted
2. **背景處理**：使用 Task.Run 或 BackgroundService
3. **狀態追蹤**：記錄處理進度
4. **錯誤處理**：重試機制和失敗通知

#### 3.3.2 背景服務設計
```csharp
public class SheetsProcessingService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // 處理佇列中的請求
        // 1. OpenAI API 呼叫
        // 2. Issue 建立
        // 3. Discord 通知
        // 4. 狀態更新
    }
}
```

## 4. 詳細實作流程

### 4.1 階段一：Google Apps Script 實作

#### 4.1.1 環境設置
- 建立新的 Google Apps Script 專案
- 設定適當的觸發器
- 配置必要的權限

#### 4.1.2 核心功能開發
```javascript
function onFormSubmit(e) {
    // 1. 取得表單資料
    // 2. 資料驗證與格式化
    // 3. 呼叫 .NET API
    // 4. 錯誤處理
}
```

#### 4.1.3 錯誤處理機制
- 網路請求失敗重試
- 日誌記錄到 Google Sheets
- 管理員通知機制

### 4.2 階段二：.NET API 端點實作

#### 4.2.1 Controller 開發
- 請求驗證
- 資料綁定
- 非同步處理啟動

#### 4.2.2 安全性實作
- API Key 驗證
- IP 白名單（可選）
- 請求速率限制
- 輸入資料驗證

#### 4.2.3 日誌與監控
- 結構化日誌記錄
- 效能指標收集
- 錯誤追蹤

### 4.3 階段三：業務邏輯實作

#### 4.3.1 OpenAI 整合
```csharp
public class OpenAIService
{
    public async Task<string> ProcessFormData(Dictionary<string, object> formData)
    {
        // 1. 資料格式化為 prompt
        // 2. 呼叫 OpenAI API
        // 3. 處理回應
        // 4. 錯誤處理與重試
    }
}
```

#### 4.3.2 Issue 系統整合
- 呼叫現有的 Issue 建立 API
- 資料映射與轉換
- 建立結果驗證

#### 4.3.3 Discord 通知
```csharp
public class DiscordService
{
    public async Task SendWebhook(string processedData, string issueId)
    {
        // 1. 格式化 Discord 訊息
        // 2. 發送 Webhook
        // 3. 處理回應
    }
}
```

## 5. 設定檔與環境變數

### 5.1 必要設定項目
```json
{
  "OpenAI": {
    "ApiKey": "sk-...",
    "BaseUrl": "https://api.openai.com/v1",
    "Model": "gpt-3.5-turbo",
    "MaxTokens": 1000,
    "Temperature": 0.7
  },
  "Discord": {
    "WebhookUrl": "https://discord.com/api/webhooks/...",
    "DefaultChannel": "#issues"
  },
  "SheetsIntegration": {
    "ApiKey": "your-api-key",
    "AllowedOrigins": ["script.google.com"],
    "MaxRequestSize": 1048576
  }
}
```

### 5.2 環境變數
- `OPENAI_API_KEY`
- `DISCORD_WEBHOOK_URL`
- `SHEETS_API_KEY`
- `ASPNETCORE_ENVIRONMENT`

## 6. 測試策略

### 6.1 單元測試
- OpenAI 服務測試
- Discord 服務測試
- 資料驗證邏輯測試

### 6.2 整合測試
- Apps Script 到 API 的完整流程
- API 到外部服務的整合
- 錯誤情境測試

### 6.3 端到端測試
- 從 Google Sheets 提交到 Discord 通知的完整流程
- 效能測試
- 負載測試

## 7. 部署與維運

### 7.1 部署清單
- [ ] .NET API 部署到生產環境
- [ ] Google Apps Script 發佈
- [ ] 設定檔案配置
- [ ] 監控系統設置
- [ ] 日誌收集配置

### 7.2 監控指標
- API 回應時間
- OpenAI API 呼叫成功率
- Discord 通知發送成功率
- 系統錯誤率
- 處理佇列長度

### 7.3 維護計劃
- 定期日誌檢查
- API 金鑰輪換
- 效能調優
- 安全性審查

## 8. 風險評估與緩解

### 8.1 技術風險
| 風險 | 影響 | 機率 | 緩解措施 |
|------|------|------|----------|
| OpenAI API 限制 | 高 | 中 | 實作重試機制、備用方案 |
| Google Apps Script 限制 | 中 | 低 | 優化執行時間、分批處理 |
| Discord API 限制 | 低 | 低 | 速率限制、錯誤處理 |

### 8.2 業務風險
| 風險 | 影響 | 機率 | 緩解措施 |
|------|------|------|----------|
| 資料遺失 | 高 | 低 | 完整日誌、備份機制 |
| 處理延遲 | 中 | 中 | 非同步處理、狀態通知 |
| 安全性問題 | 高 | 低 | 多層驗證、定期審查 |

## 9. 時程規劃

### 9.1 開發階段（預估 2-3 週）
- **Week 1**：Google Apps Script + .NET API 端點
- **Week 2**：業務邏輯實作（OpenAI + Discord）
- **Week 3**：測試、優化、部署

### 9.2 里程碑
- [ ] MVP 功能完成
- [ ] 整合測試通過
- [ ] 生產環境部署
- [ ] 使用者驗收測試

## 10. 後續擴展

### 10.1 可能的功能增強
- 支援多種表單格式
- 自訂 AI 處理規則
- Discord 訊息格式自訂
- 處理狀態儀表板

### 10.2 效能優化
- 快取常用資料
- 批次處理機制
- 資料庫連線池優化
- CDN 整合
