# OpenAI 整理與 Discord 發送實作

## 1. 整體處理流程

```
表單資料收集 → OpenAI API 整理 → 格式化 Discord 訊息 → 發送 Webhook → 錯誤處理
```

## 2. OpenAI 整理服務

### 2.1 OpenAI API 配置

```javascript
/**
 * OpenAI API 設定
 */
const OPENAI_CONFIG = {
    apiKey: getOpenAIApiKey(),
    baseUrl: 'https://api.openai.com/v1',
    model: 'gpt-3.5-turbo',
    maxTokens: 1000,
    temperature: 0.3,
    timeout: 30000
};

/**
 * 取得 OpenAI API 金鑰
 */
function getOpenAIApiKey() {
    return PropertiesService.getScriptProperties().getProperty('OPENAI_API_KEY') || '';
}
```

### 2.2 資料整理主函式

```javascript
/**
 * 使用 OpenAI 整理客戶回報資料
 * @param {Object} formData 表單資料
 * @param {Object} fileData 檔案資料
 * @returns {Object} 整理後的資料
 */
async function processWithOpenAI(formData, fileData) {
    try {
        console.log('開始 OpenAI 資料整理...');
        
        // 1. 建立 AI 提示詞
        const prompt = createAIPrompt(formData, fileData);
        
        // 2. 呼叫 OpenAI API
        const aiResponse = await callOpenAI(prompt);
        
        // 3. 解析 AI 回應
        const parsedResponse = parseAIResponse(aiResponse);
        
        // 4. 建立整理結果
        const processedResult = {
            original: formData,
            aiSummary: parsedResponse,
            processingTime: new Date().toISOString(),
            success: true
        };
        
        console.log('OpenAI 整理完成');
        return processedResult;
        
    } catch (error) {
        console.error('OpenAI 處理錯誤:', error);
        
        // 回傳備用格式
        return {
            original: formData,
            aiSummary: createFallbackSummary(formData),
            processingTime: new Date().toISOString(),
            success: false,
            error: error.message
        };
    }
}
```

### 2.3 AI 提示詞建立

```javascript
/**
 * 建立 OpenAI 提示詞
 * @param {Object} formData 表單資料
 * @param {Object} fileData 檔案資料
 * @returns {string} 完整提示詞
 */
function createAIPrompt(formData, fileData) {
    const hasImages = fileData.screenshots && fileData.screenshots.length > 0;
    const hasVideo = formData.videoLink && formData.videoLink.trim() !== '';
    const hasDocuments = formData.documentLink && formData.documentLink.trim() !== '';
    
    const prompt = `
請協助整理以下客戶回報資訊，並以繁體中文回應。請依照指定格式輸出結構化的摘要：

## 客戶資訊
- 姓名：${formData.customerName}
- 公司：${formData.company || '未提供'}
- 聯絡方式：${formData.email}${formData.phone ? ` / ${formData.phone}` : ''}
- 慣用聯絡：${formData.preferredContact}

## 問題詳情
- 類型：${formData.reportType}
- 緊急程度：${formData.priority}
- 影響範圍：${formData.impactScope}
- 標題：${formData.title}
- 詳細描述：${formData.description}
- 重現步驟：${formData.reproductionSteps || '未提供'}
- 使用環境：${formData.environment || '未提供'}
- 錯誤訊息：${formData.errorMessage || '未提供'}
- 其他補充：${formData.additionalInfo || '未提供'}

## 附件資訊
- 截圖：${hasImages ? `${fileData.screenshots.length} 個檔案` : '無'}
- 影片連結：${hasVideo ? '有提供' : '無'}
- 文件連結：${hasDocuments ? '有提供' : '無'}

請你：
1. 將問題描述重新整理成簡潔明瞭的摘要
2. 識別問題的關鍵重點和可能原因
3. 評估問題的嚴重性和建議處理優先級
4. 提供初步的建議解決方向

回應格式請使用以下 JSON 結構：
{
  "summary": "問題簡要摘要 (50字內)",
  "keyPoints": ["關鍵點1", "關鍵點2", "關鍵點3"],
  "severity": "critical|high|medium|low",
  "category": "技術問題|使用者體驗|功能需求|其他",
  "suggestedActions": ["建議行動1", "建議行動2"],
  "estimatedComplexity": "simple|moderate|complex",
  "requiresImmediate": true|false,
  "notes": "額外備註說明"
}

請確保回應是有效的 JSON 格式。
`;

    return prompt.trim();
}
```

### 2.4 OpenAI API 呼叫

```javascript
/**
 * 呼叫 OpenAI API
 * @param {string} prompt 提示詞
 * @returns {string} API 回應內容
 */
async function callOpenAI(prompt) {
    const requestBody = {
        model: OPENAI_CONFIG.model,
        messages: [
            {
                role: "system",
                content: "你是一個專業的客戶服務分析師，擅長整理和分析客戶回報的問題。請以專業、客觀的角度分析問題，並提供實用的建議。"
            },
            {
                role: "user",
                content: prompt
            }
        ],
        max_tokens: OPENAI_CONFIG.maxTokens,
        temperature: OPENAI_CONFIG.temperature,
        response_format: { type: "json_object" }
    };
    
    const requestOptions = {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${OPENAI_CONFIG.apiKey}`,
            'Content-Type': 'application/json'
        },
        payload: JSON.stringify(requestBody),
        muteHttpExceptions: true
    };
    
    try {
        console.log('發送 OpenAI API 請求...');
        
        const response = UrlFetchApp.fetch(
            `${OPENAI_CONFIG.baseUrl}/chat/completions`,
            requestOptions
        );
        
        const statusCode = response.getResponseCode();
        const responseText = response.getContentText();
        
        if (statusCode !== 200) {
            throw new Error(`OpenAI API 錯誤 (${statusCode}): ${responseText}`);
        }
        
        const responseData = JSON.parse(responseText);
        
        if (!responseData.choices || responseData.choices.length === 0) {
            throw new Error('OpenAI API 回應格式錯誤');
        }
        
        return responseData.choices[0].message.content;
        
    } catch (error) {
        console.error('OpenAI API 呼叫失敗:', error);
        throw error;
    }
}
```

### 2.5 AI 回應解析

```javascript
/**
 * 解析 OpenAI 回應
 * @param {string} aiResponse AI 原始回應
 * @returns {Object} 解析後的結構化資料
 */
function parseAIResponse(aiResponse) {
    try {
        const parsed = JSON.parse(aiResponse);
        
        // 驗證必要欄位
        const required = ['summary', 'keyPoints', 'severity', 'category'];
        const missing = required.filter(field => !parsed[field]);
        
        if (missing.length > 0) {
            throw new Error(`AI 回應缺少必要欄位: ${missing.join(', ')}`);
        }
        
        // 標準化資料格式
        return {
            summary: String(parsed.summary).substring(0, 100), // 限制長度
            keyPoints: Array.isArray(parsed.keyPoints) ? parsed.keyPoints.slice(0, 5) : [],
            severity: validateSeverity(parsed.severity),
            category: String(parsed.category),
            suggestedActions: Array.isArray(parsed.suggestedActions) ? parsed.suggestedActions.slice(0, 3) : [],
            estimatedComplexity: validateComplexity(parsed.estimatedComplexity),
            requiresImmediate: Boolean(parsed.requiresImmediate),
            notes: String(parsed.notes || '').substring(0, 200)
        };
        
    } catch (error) {
        console.error('AI 回應解析錯誤:', error);
        throw new Error('AI 回應格式無效');
    }
}

/**
 * 驗證嚴重性等級
 */
function validateSeverity(severity) {
    const validLevels = ['critical', 'high', 'medium', 'low'];
    return validLevels.includes(severity) ? severity : 'medium';
}

/**
 * 驗證複雜度等級
 */
function validateComplexity(complexity) {
    const validLevels = ['simple', 'moderate', 'complex'];
    return validLevels.includes(complexity) ? complexity : 'moderate';
}

/**
 * 建立備用摘要（當 AI 處理失敗時使用）
 */
function createFallbackSummary(formData) {
    return {
        summary: `${formData.reportType}: ${formData.title}`,
        keyPoints: [
            formData.priority,
            formData.impactScope,
            '需要人工審核'
        ],
        severity: mapPriorityToSeverity(formData.priority),
        category: mapReportTypeToCategory(formData.reportType),
        suggestedActions: ['指派給相關團隊', '進行初步分析'],
        estimatedComplexity: 'moderate',
        requiresImmediate: formData.priority.includes('緊急'),
        notes: 'AI 處理失敗，使用備用格式'
    };
}
```

## 3. Discord 發送服務

### 3.1 Discord 配置

```javascript
/**
 * Discord Webhook 設定
 */
const DISCORD_CONFIG = {
    webhookUrl: getDiscordWebhookUrl(),
    username: '客戶回報系統',
    avatarUrl: 'https://your-domain.com/avatar.png',
    timeout: 10000
};

/**
 * 取得 Discord Webhook URL
 */
function getDiscordWebhookUrl() {
    return PropertiesService.getScriptProperties().getProperty('DISCORD_WEBHOOK_URL') || '';
}
```

### 3.2 Discord 訊息建立

```javascript
/**
 * 建立 Discord 訊息內容
 * @param {Object} processedData OpenAI 處理後的資料
 * @param {Object} fileData 檔案資料
 * @returns {Object} Discord Webhook 負載
 */
function createDiscordMessage(processedData, fileData) {
    const { original, aiSummary } = processedData;
    
    // 選擇適當的顏色和表情符號
    const severityConfig = getSeverityConfig(aiSummary.severity);
    const typeEmoji = getTypeEmoji(original.reportTypeCode);
    
    // 建立主要 Embed
    const mainEmbed = {
        title: `${typeEmoji} ${original.title}`,
        description: aiSummary.summary,
        color: severityConfig.color,
        timestamp: new Date().toISOString(),
        fields: [
            {
                name: "🎯 AI 分析重點",
                value: aiSummary.keyPoints.map(point => `• ${point}`).join('\n') || '無',
                inline: false
            },
            {
                name: "👤 客戶資訊",
                value: `**${original.customerName}**\n${original.company ? `${original.company}\n` : ''}📧 ${original.email}${original.phone ? `\n📱 ${original.phone}` : ''}`,
                inline: true
            },
            {
                name: "📊 問題分類",
                value: `**類型:** ${aiSummary.category}\n**緊急度:** ${severityConfig.label}\n**影響:** ${original.impactScope}\n**複雜度:** ${getComplexityLabel(aiSummary.estimatedComplexity)}`,
                inline: true
            }
        ]
    };
    
    // 添加建議行動
    if (aiSummary.suggestedActions && aiSummary.suggestedActions.length > 0) {
        mainEmbed.fields.push({
            name: "💡 建議處理方式",
            value: aiSummary.suggestedActions.map(action => `• ${action}`).join('\n'),
            inline: false
        });
    }
    
    // 添加技術詳情（如果有）
    const technicalDetails = buildTechnicalDetails(original);
    if (technicalDetails) {
        mainEmbed.fields.push({
            name: "🔧 技術詳情",
            value: technicalDetails,
            inline: false
        });
    }
    
    // 建立附件資訊 Embed
    const attachmentEmbed = buildAttachmentEmbed(original, fileData);
    
    // 組合完整訊息
    const discordPayload = {
        username: DISCORD_CONFIG.username,
        avatar_url: DISCORD_CONFIG.avatarUrl,
        embeds: [mainEmbed],
        components: [
            {
                type: 1, // Action Row
                components: [
                    {
                        type: 2, // Button
                        style: aiSummary.requiresImmediate ? 4 : 1, // Red if urgent, Blue otherwise
                        label: aiSummary.requiresImmediate ? "🚨 立即處理" : "📋 查看詳情",
                        custom_id: `ticket_${original.ticketId}`,
                        disabled: false
                    }
                ]
            }
        ]
    };
    
    // 添加附件 Embed（如果有）
    if (attachmentEmbed) {
        discordPayload.embeds.push(attachmentEmbed);
    }
    
    // 添加備註（如果有）
    if (aiSummary.notes && aiSummary.notes.trim() !== '') {
        discordPayload.embeds.push({
            title: "📝 AI 分析備註",
            description: aiSummary.notes,
            color: 0x95a5a6
        });
    }
    
    // 緊急情況添加 @everyone
    if (aiSummary.requiresImmediate && aiSummary.severity === 'critical') {
        discordPayload.content = "🚨 **緊急客戶回報** - 需要立即關注！";
    }
    
    return discordPayload;
}
```

### 3.3 輔助函式

```javascript
/**
 * 取得嚴重性等級配置
 */
function getSeverityConfig(severity) {
    const configs = {
        critical: { color: 0xe74c3c, label: '🔥 緊急', priority: 4 },
        high: { color: 0xf39c12, label: '⚠️ 高', priority: 3 },
        medium: { color: 0x3498db, label: '📊 中等', priority: 2 },
        low: { color: 0x95a5a6, label: '📝 低', priority: 1 }
    };
    return configs[severity] || configs.medium;
}

/**
 * 取得類型表情符號
 */
function getTypeEmoji(reportType) {
    const emojis = {
        bug: '🐛',
        feature_request: '💡',
        usage_question: '❓',
        technical_support: '🔧'
    };
    return emojis[reportType] || '📋';
}

/**
 * 取得複雜度標籤
 */
function getComplexityLabel(complexity) {
    const labels = {
        simple: '🟢 簡單',
        moderate: '🟡 中等',
        complex: '🔴 複雜'
    };
    return labels[complexity] || labels.moderate;
}

/**
 * 建立技術詳情內容
 */
function buildTechnicalDetails(formData) {
    const details = [];
    
    if (formData.environment) {
        details.push(`**環境:** ${formData.environment}`);
    }
    
    if (formData.reproductionSteps) {
        details.push(`**重現步驟:**\n${formData.reproductionSteps}`);
    }
    
    if (formData.errorMessage) {
        details.push(`**錯誤訊息:**\n\`\`\`\n${formData.errorMessage}\n\`\`\``);
    }
    
    return details.length > 0 ? details.join('\n\n') : null;
}

/**
 * 建立附件資訊 Embed
 */
function buildAttachmentEmbed(formData, fileData) {
    const attachments = [];
    
    // 截圖資訊
    if (fileData.screenshots && fileData.screenshots.length > 0) {
        attachments.push(`📸 **截圖:** ${fileData.screenshots.length} 個檔案`);
        
        // 顯示第一張圖片
        const firstImage = fileData.screenshots[0];
        return {
            title: "📎 附件資訊",
            fields: [
                {
                    name: "檔案列表",
                    value: fileData.screenshots.map((file, index) => 
                        `${index + 1}. [${file.name}](${file.url}) (${Math.round(file.size / 1024)}KB)`
                    ).join('\n'),
                    inline: false
                }
            ],
            image: {
                url: firstImage.downloadUrl
            },
            color: 0x9b59b6
        };
    }
    
    // 連結資訊
    if (formData.videoLink || formData.documentLink) {
        const links = [];
        if (formData.videoLink) links.push(`🎥 [影片連結](${formData.videoLink})`);
        if (formData.documentLink) links.push(`📄 [文件連結](${formData.documentLink})`);
        
        return {
            title: "🔗 相關連結",
            description: links.join('\n'),
            color: 0x3498db
        };
    }
    
    return null;
}
```

### 3.4 Discord 發送函式

```javascript
/**
 * 發送訊息到 Discord
 * @param {Object} processedData 處理後的資料
 * @param {Object} fileData 檔案資料
 * @returns {Object} 發送結果
 */
async function sendToDiscord(processedData, fileData) {
    try {
        console.log('準備發送 Discord 訊息...');
        
        // 建立 Discord 訊息
        const discordMessage = createDiscordMessage(processedData, fileData);
        
        // 發送 Webhook
        const response = await sendDiscordWebhook(discordMessage);
        
        console.log('Discord 訊息發送成功');
        return {
            success: true,
            messageId: response.id,
            timestamp: new Date().toISOString()
        };
        
    } catch (error) {
        console.error('Discord 發送失敗:', error);
        return {
            success: false,
            error: error.message,
            timestamp: new Date().toISOString()
        };
    }
}

/**
 * 發送 Discord Webhook
 * @param {Object} payload Discord 訊息負載
 * @returns {Object} Discord API 回應
 */
async function sendDiscordWebhook(payload) {
    const requestOptions = {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        payload: JSON.stringify(payload),
        muteHttpExceptions: true
    };
    
    try {
        const response = UrlFetchApp.fetch(DISCORD_CONFIG.webhookUrl, requestOptions);
        const statusCode = response.getResponseCode();
        const responseText = response.getContentText();
        
        if (statusCode === 204) {
            // Discord Webhook 成功回傳 204 No Content
            return { success: true };
        } else if (statusCode >= 200 && statusCode < 300) {
            return JSON.parse(responseText);
        } else {
            throw new Error(`Discord API 錯誤 (${statusCode}): ${responseText}`);
        }
        
    } catch (error) {
        console.error('Discord Webhook 發送失敗:', error);
        throw error;
    }
}
```

## 4. 主要整合函式

```javascript
/**
 * 主要處理函式：OpenAI 整理 + Discord 發送
 * @param {Object} formData 表單資料
 * @param {Object} fileData 檔案資料
 * @returns {Object} 完整處理結果
 */
async function processAndSendToDiscord(formData, fileData) {
    const result = {
        ticketId: formData.ticketId,
        timestamp: new Date().toISOString(),
        steps: {
            aiProcessing: { success: false },
            discordSending: { success: false }
        }
    };
    
    try {
        // 步驟 1: OpenAI 處理
        console.log('步驟 1: 開始 OpenAI 資料整理');
        const processedData = await processWithOpenAI(formData, fileData);
        result.steps.aiProcessing = {
            success: processedData.success,
            data: processedData.aiSummary,
            error: processedData.error
        };
        
        // 步驟 2: Discord 發送
        console.log('步驟 2: 發送到 Discord');
        const discordResult = await sendToDiscord(processedData, fileData);
        result.steps.discordSending = {
            success: discordResult.success,
            messageId: discordResult.messageId,
            error: discordResult.error
        };
        
        // 整體結果
        result.success = result.steps.aiProcessing.success && result.steps.discordSending.success;
        result.message = result.success ? '處理完成' : '部分步驟失敗';
        
        return result;
        
    } catch (error) {
        console.error('整體處理失敗:', error);
        result.success = false;
        result.error = error.message;
        return result;
    }
}
```

## 5. 環境變數設定

在 Google Apps Script 的「設定 > 指令碼屬性」中添加：

```
OPENAI_API_KEY = sk-your-openai-api-key
DISCORD_WEBHOOK_URL = https://discord.com/api/webhooks/your-webhook-url
```

## 6. 使用方式

將原本的 `sendToBackendAPI` 函式替換為：

```javascript
// 在 onFormSubmit 函式中替換
const result = await processAndSendToDiscord(processedData, fileData);
handleProcessingResult(result, formData);
```

這個實作方案提供了：
- 智能的 AI 資料分析和摘要
- 豐富的 Discord 訊息格式
- 完整的錯誤處理機制
- 彈性的配置選項
- 視覺化的優先級和類別標示

你覺得這個設計如何？有需要調整的地方嗎？