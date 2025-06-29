# 步驟一：Google Apps Script 客戶回報系統實作

## 1. 表單結構設計

### 1.1 Google Forms 表單欄位設計

#### 1.1.1 基本資訊區塊
```
📋 客戶基本資訊
├── 姓名 * (短答題)
├── 電子郵件 * (短答題，含驗證)
├── 聯絡電話 (短答題)
├── 公司/組織名稱 (短答題)
└── 慣用聯絡方式 * (單選：電子郵件/電話/兩者皆可)
```

#### 1.1.2 問題分類區塊
```
🏷️ 問題分類
├── 回報類型 * (單選)
│   ├── 🐛 Bug 回報
│   ├── 💡 功能需求
│   ├── ❓ 使用問題
│   └── 🔧 技術支援
├── 緊急程度 * (單選)
│   ├── 🔥 緊急 (影響營運)
│   ├── ⚠️ 高 (重要功能異常)
│   ├── 📊 中 (一般問題)
│   └── 📝 低 (建議改善)
└── 影響範圍 * (單選)
    ├── 🌐 全系統
    ├── 👥 多用戶
    ├── 👤 個人帳戶
    └── 🔍 特定功能
```

#### 1.1.3 問題詳述區塊
```
📝 問題詳述
├── 問題標題 * (短答題，限制 100 字)
├── 詳細描述 * (長答題)
│   └── 提示：請詳細描述問題發生的情況、步驟、預期結果等
├── 重現步驟 (長答題)
│   └── 提示：請依序列出重現問題的詳細步驟
├── 使用環境 (短答題)
│   └── 提示：如瀏覽器版本、作業系統、裝置類型等
└── 錯誤訊息 (長答題)
    └── 提示：如有錯誤訊息，請完整複製貼上
```

#### 1.1.4 附件與證明區塊
```
📎 附件與證明
├── 螢幕截圖 (檔案上傳，允許多個檔案)
│   └── 接受格式：.jpg, .jpeg, .png, .gif, .webp
├── 影片證明連結 (短答題)
│   └── 提示：請提供 YouTube、Google Drive、Dropbox 等雲端連結
├── 相關文件連結 (短答題)
│   └── 提示：如有相關文件，請提供可存取的雲端連結
└── 其他補充資料 (長答題)
    └── 提示：任何其他有助於問題解決的資訊
```

### 1.2 表單驗證規則

#### 1.2.1 必填欄位驗證
```javascript
const REQUIRED_FIELDS = {
    'name': '姓名',
    'email': '電子郵件',
    'contact_method': '慣用聯絡方式',
    'report_type': '回報類型',
    'priority': '緊急程度',
    'impact_scope': '影響範圍',
    'title': '問題標題',
    'description': '詳細描述'
};
```

#### 1.2.2 格式驗證規則
```javascript
const VALIDATION_RULES = {
    email: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
    phone: /^[\d\-\+\(\)\s]+$/,
    url: /^https?:\/\/.+/,
    title_max_length: 100,
    file_max_size: 10 * 1024 * 1024, // 10MB
    allowed_file_types: ['jpg', 'jpeg', 'png', 'gif', 'webp']
};
```

## 2. Google Apps Script 實作

### 2.1 主要處理函式

```javascript
/**
 * 表單提交觸發函式
 * @param {GoogleAppsScript.Events.SheetsOnFormSubmit} e 
 */
function onFormSubmit(e) {
    try {
        console.log('=== 表單提交處理開始 ===');
        
        // 1. 資料收集
        const formData = collectFormData(e);
        
        // 2. 資料驗證
        const validationResult = validateFormData(formData);
        if (!validationResult.isValid) {
            logError('資料驗證失敗', validationResult.errors);
            return;
        }
        
        // 3. 資料預處理
        const processedData = preprocessFormData(formData);
        
        // 4. 檔案處理
        const fileData = await processAttachments(formData);
        
        // 5. 整合資料
        const payload = createApiPayload(processedData, fileData);
        
        // 6. 發送到後端 API
        const response = await sendToBackendAPI(payload);
        
        // 7. 處理回應
        handleApiResponse(response, formData);
        
        console.log('=== 表單提交處理完成 ===');
        
    } catch (error) {
        console.error('表單處理錯誤:', error);
        logError('表單處理錯誤', error);
        sendErrorNotification(error, e);
    }
}
```

### 2.2 資料收集函式

```javascript
/**
 * 收集表單資料
 * @param {GoogleAppsScript.Events.SheetsOnFormSubmit} e 
 * @returns {Object} 格式化的表單資料
 */
function collectFormData(e) {
    const range = e.range;
    const values = range.getValues()[0];
    const headers = e.source.getSheets()[0].getRange(1, 1, 1, values.length).getValues()[0];
    
    const formData = {};
    
    // 基本欄位映射
    const fieldMapping = {
        '姓名': 'customerName',
        '電子郵件': 'email',
        '聯絡電話': 'phone',
        '公司/組織名稱': 'company',
        '慣用聯絡方式': 'preferredContact',
        '回報類型': 'reportType',
        '緊急程度': 'priority',
        '影響範圍': 'impactScope',
        '問題標題': 'title',
        '詳細描述': 'description',
        '重現步驟': 'reproductionSteps',
        '使用環境': 'environment',
        '錯誤訊息': 'errorMessage',
        '影片證明連結': 'videoLink',
        '相關文件連結': 'documentLink',
        '其他補充資料': 'additionalInfo'
    };
    
    // 資料映射
    headers.forEach((header, index) => {
        const fieldName = fieldMapping[header];
        if (fieldName) {
            formData[fieldName] = values[index] || '';
        }
    });
    
    // 添加時間戳記和其他元資料
    formData.submissionTime = new Date().toISOString();
    formData.spreadsheetId = e.source.getId();
    formData.rowNumber = range.getRow();
    
    return formData;
}
```

### 2.3 資料驗證函式

```javascript
/**
 * 驗證表單資料
 * @param {Object} formData 表單資料
 * @returns {Object} 驗證結果
 */
function validateFormData(formData) {
    const errors = [];
    
    // 必填欄位檢查
    Object.entries(REQUIRED_FIELDS).forEach(([field, displayName]) => {
        if (!formData[field] || formData[field].trim() === '') {
            errors.push(`${displayName} 為必填欄位`);
        }
    });
    
    // 電子郵件格式檢查
    if (formData.email && !VALIDATION_RULES.email.test(formData.email)) {
        errors.push('電子郵件格式不正確');
    }
    
    // 電話格式檢查
    if (formData.phone && !VALIDATION_RULES.phone.test(formData.phone)) {
        errors.push('電話號碼格式不正確');
    }
    
    // URL 格式檢查
    if (formData.videoLink && formData.videoLink.trim() !== '') {
        if (!VALIDATION_RULES.url.test(formData.videoLink)) {
            errors.push('影片連結格式不正確');
        }
    }
    
    if (formData.documentLink && formData.documentLink.trim() !== '') {
        if (!VALIDATION_RULES.url.test(formData.documentLink)) {
            errors.push('文件連結格式不正確');
        }
    }
    
    // 標題長度檢查
    if (formData.title && formData.title.length > VALIDATION_RULES.title_max_length) {
        errors.push(`問題標題不可超過 ${VALIDATION_RULES.title_max_length} 字`);
    }
    
    return {
        isValid: errors.length === 0,
        errors: errors
    };
}
```

### 2.4 資料預處理函式

```javascript
/**
 * 預處理表單資料
 * @param {Object} formData 原始表單資料
 * @returns {Object} 處理後的資料
 */
function preprocessFormData(formData) {
    const processed = { ...formData };
    
    // 文字資料清理
    ['title', 'description', 'reproductionSteps', 'errorMessage', 'additionalInfo'].forEach(field => {
        if (processed[field]) {
            processed[field] = processed[field].trim();
        }
    });
    
    // 聯絡資訊標準化
    if (processed.email) {
        processed.email = processed.email.toLowerCase().trim();
    }
    
    if (processed.phone) {
        processed.phone = processed.phone.replace(/\s+/g, '');
    }
    
    // 緊急程度映射
    const priorityMapping = {
        '🔥 緊急 (影響營運)': 'critical',
        '⚠️ 高 (重要功能異常)': 'high',
        '📊 中 (一般問題)': 'medium',
        '📝 低 (建議改善)': 'low'
    };
    processed.priorityLevel = priorityMapping[processed.priority] || 'medium';
    
    // 回報類型映射
    const typeMapping = {
        '🐛 Bug 回報': 'bug',
        '💡 功能需求': 'feature_request',
        '❓ 使用問題': 'usage_question',
        '🔧 技術支援': 'technical_support'
    };
    processed.reportTypeCode = typeMapping[processed.reportType] || 'bug';
    
    // 生成唯一識別碼
    processed.ticketId = generateTicketId();
    
    return processed;
}
```

### 2.5 檔案處理函式

```javascript
/**
 * 處理附件檔案
 * @param {Object} formData 表單資料
 * @returns {Object} 檔案處理結果
 */
async function processAttachments(formData) {
    const fileData = {
        screenshots: [],
        totalSize: 0,
        processedCount: 0,
        errors: []
    };
    
    try {
        // 取得表單回應中的檔案
        const form = FormApp.getActiveForm();
        const responses = form.getResponses();
        const latestResponse = responses[responses.length - 1];
        
        const itemResponses = latestResponse.getItemResponses();
        
        itemResponses.forEach(itemResponse => {
            const item = itemResponse.getItem();
            
            // 檢查是否為檔案上傳題目
            if (item.getType() === FormApp.ItemType.FILE_UPLOAD) {
                const fileIds = itemResponse.getResponse();
                
                if (Array.isArray(fileIds)) {
                    fileIds.forEach(fileId => {
                        try {
                            const file = DriveApp.getFileById(fileId);
                            const fileInfo = processScreenshot(file);
                            
                            if (fileInfo.success) {
                                fileData.screenshots.push(fileInfo.data);
                                fileData.totalSize += fileInfo.data.size;
                                fileData.processedCount++;
                            } else {
                                fileData.errors.push(fileInfo.error);
                            }
                        } catch (error) {
                            fileData.errors.push(`檔案處理錯誤: ${error.message}`);
                        }
                    });
                }
            }
        });
        
    } catch (error) {
        console.error('附件處理錯誤:', error);
        fileData.errors.push(`附件處理錯誤: ${error.message}`);
    }
    
    return fileData;
}

/**
 * 處理單一截圖檔案
 * @param {GoogleAppsScript.Drive.File} file 
 * @returns {Object} 處理結果
 */
function processScreenshot(file) {
    try {
        const fileName = file.getName();
        const fileSize = file.getSize();
        const mimeType = file.getBlob().getContentType();
        
        // 檔案大小檢查
        if (fileSize > VALIDATION_RULES.file_max_size) {
            return {
                success: false,
                error: `檔案 ${fileName} 超過大小限制 (${Math.round(fileSize / 1024 / 1024)}MB)`
            };
        }
        
        // 檔案類型檢查
        const fileExtension = fileName.split('.').pop().toLowerCase();
        if (!VALIDATION_RULES.allowed_file_types.includes(fileExtension)) {
            return {
                success: false,
                error: `檔案 ${fileName} 格式不支援`
            };
        }
        
        // 設定檔案權限為可檢視
        file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
        
        return {
            success: true,
            data: {
                id: file.getId(),
                name: fileName,
                size: fileSize,
                mimeType: mimeType,
                url: file.getUrl(),
                downloadUrl: `https://drive.google.com/uc?id=${file.getId()}`,
                thumbnailUrl: file.getThumbnail() ? `https://drive.google.com/thumbnail?id=${file.getId()}` : null
            }
        };
        
    } catch (error) {
        return {
            success: false,
            error: `檔案處理失敗: ${error.message}`
        };
    }
}
```

### 2.6 API 呼叫函式

```javascript
/**
 * 建立 API 請求資料
 * @param {Object} processedData 處理後的表單資料
 * @param {Object} fileData 檔案資料
 * @returns {Object} API 請求負載
 */
function createApiPayload(processedData, fileData) {
    return {
        eventType: 'customer_report_submission',
        timestamp: new Date().toISOString(),
        source: 'google_forms',
        data: {
            // 客戶資訊
            customer: {
                name: processedData.customerName,
                email: processedData.email,
                phone: processedData.phone,
                company: processedData.company,
                preferredContact: processedData.preferredContact
            },
            
            // 問題資訊
            issue: {
                ticketId: processedData.ticketId,
                title: processedData.title,
                description: processedData.description,
                reproductionSteps: processedData.reproductionSteps,
                environment: processedData.environment,
                errorMessage: processedData.errorMessage,
                additionalInfo: processedData.additionalInfo,
                reportType: processedData.reportTypeCode,
                priority: processedData.priorityLevel,
                impactScope: processedData.impactScope
            },
            
            // 媒體資訊
            media: {
                screenshots: fileData.screenshots,
                videoLink: processedData.videoLink,
                documentLink: processedData.documentLink,
                totalFileSize: fileData.totalSize,
                fileProcessingErrors: fileData.errors
            },
            
            // 元資料
            metadata: {
                spreadsheetId: processedData.spreadsheetId,
                rowNumber: processedData.rowNumber,
                submissionTime: processedData.submissionTime,
                processingTime: new Date().toISOString()
            }
        }
    };
}

/**
 * 發送資料到後端 API
 * @param {Object} payload API 請求負載
 * @returns {Object} API 回應
 */
async function sendToBackendAPI(payload) {
    const apiConfig = {
        url: 'https://your-api-domain.com/api/sheets/webhook',
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${getApiKey()}`,
            'X-Source': 'google-apps-script',
            'X-Version': '1.0'
        },
        payload: JSON.stringify(payload)
    };
    
    try {
        console.log('發送 API 請求:', apiConfig.url);
        
        const response = UrlFetchApp.fetch(apiConfig.url, {
            method: apiConfig.method,
            headers: apiConfig.headers,
            payload: apiConfig.payload,
            muteHttpExceptions: true
        });
        
        const responseCode = response.getResponseCode();
        const responseText = response.getContentText();
        
        console.log(`API 回應狀態: ${responseCode}`);
        
        if (responseCode >= 200 && responseCode < 300) {
            return {
                success: true,
                statusCode: responseCode,
                data: JSON.parse(responseText)
            };
        } else {
            return {
                success: false,
                statusCode: responseCode,
                error: responseText
            };
        }
        
    } catch (error) {
        console.error('API 呼叫錯誤:', error);
        return {
            success: false,
            error: error.message
        };
    }
}
```

### 2.7 輔助函式

```javascript
/**
 * 生成工單 ID
 * @returns {string} 唯一工單 ID
 */
function generateTicketId() {
    const timestamp = new Date().getTime();
    const random = Math.floor(Math.random() * 1000).toString().padStart(3, '0');
    return `TICKET-${timestamp}-${random}`;
}

/**
 * 取得 API 金鑰
 * @returns {string} API 金鑰
 */
function getApiKey() {
    return PropertiesService.getScriptProperties().getProperty('API_KEY') || 'your-default-api-key';
}

/**
 * 記錄錯誤
 * @param {string} message 錯誤訊息
 * @param {*} error 錯誤物件
 */
function logError(message, error) {
    const errorLog = {
        timestamp: new Date().toISOString(),
        message: message,
        error: error.toString(),
        stack: error.stack || 'No stack trace available'
    };
    
    console.error('錯誤記錄:', errorLog);
    
    // 可選：寫入到 Google Sheets 進行錯誤追蹤
    // writeErrorToSheet(errorLog);
}

/**
 * 處理 API 回應
 * @param {Object} response API 回應
 * @param {Object} formData 原始表單資料
 */
function handleApiResponse(response, formData) {
    if (response.success) {
        console.log('API 呼叫成功:', response.data);
        
        // 可選：發送確認郵件給客戶
        // sendConfirmationEmail(formData, response.data);
        
    } else {
        console.error('API 呼叫失敗:', response.error);
        
        // 發送錯誤通知給管理員
        sendErrorNotification(response.error, formData);
    }
}

/**
 * 發送錯誤通知
 * @param {*} error 錯誤資訊
 * @param {Object} formData 表單資料
 */
function sendErrorNotification(error, formData) {
    const adminEmail = 'admin@your-company.com';
    const subject = '客戶回報系統處理錯誤';
    const body = `
系統處理客戶回報時發生錯誤：

錯誤資訊：${error}

客戶資訊：
- 姓名：${formData.customerName || 'N/A'}
- 電子郵件：${formData.email || 'N/A'}
- 問題標題：${formData.title || 'N/A'}

請儘速檢查系統狀態。
    `;
    
    try {
        MailApp.sendEmail(adminEmail, subject, body);
    } catch (emailError) {
        console.error('發送錯誤通知失敗:', emailError);
    }
}
```

## 3. 部署設定步驟

### 3.1 Google Apps Script 專案設定

1. **建立新專案**
   - 前往 Google Apps Script (script.google.com)
   - 建立新專案
   - 重新命名為 "客戶回報系統處理器"

2. **權限設定**
   - 執行 > 檢閱權限
   - 授權存取 Google Forms、Drive、Gmail

3. **環境變數設定**
   - 設定 > 指令碼屬性
   - 新增屬性：`API_KEY` = `你的API金鑰`

### 3.2 觸發器設定

```javascript
/**
 * 設定表單提交觸發器
 */
function setupTriggers() {
    // 刪除現有觸發器
    ScriptApp.getProjectTriggers().forEach(trigger => {
        ScriptApp.deleteTrigger(trigger);
    });
    
    // 建立新的表單提交觸發器
    const form = FormApp.getActiveForm();
    ScriptApp.newTrigger('onFormSubmit')
        .create();
}
```

### 3.3 測試流程

1. **單元測試**
   - 測試資料驗證函式
   - 測試檔案處理函式
   - 測試 API 呼叫函式

2. **整合測試**
   - 建立測試表單
   - 模擬各種提交情境
   - 驗證錯誤處理機制

3. **上線前檢查**
   - 檢查所有權限設定
   - 確認 API 金鑰正確
   - 測試通知機制

這個實作方案涵蓋了完整的客戶回報系統需求，包含圖片上傳、連結驗證、聯絡資訊收集等功能。你覺得這個設計如何？有需要調整的地方嗎？
