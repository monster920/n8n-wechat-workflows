# n8n WeChat Public Account Automation - Technical Documentation

This document provides detailed technical explanation of each workflow, including node configurations and data flow logic.

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Tech Stack](#tech-stack)
3. [Data Model](#data-model)
4. [Workflow Details](#workflow-details)
5. [API Integration](#api-integration)
6. [Configuration](#configuration)
7. [Troubleshooting](#troubleshooting)

---

## System Architecture

### Architecture Diagram

```
                    ┌─────────────────┐
                    │  Feishu Bitable │
                    │  (Data Storage) │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  01_Content   │    │  02_Image    │    │  03_Typeset   │
│  Generation   │    │  Generation  │    │              │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        └────────────────────┴────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ WeChat Drafts   │
                    └─────────────────┘
```

---

## Tech Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| n8n | 1.0+ | Workflow engine |
| DeepSeek | - | AI content generation |
| Tongyi Wanxiang (wan2.6-t2i) | - | AI image generation |
| Feishu Open API | v3 | Bitable data management |
| WeChat Official Account API | - | Draft publishing |

---

## Data Model

### Feishu Bitable Fields

| Field Name | Type | Description |
|------------|------|-------------|
| Article Title | Text | WeChat article title |
| Article Content | Rich Text | Generated article body |
| Article Summary | Text | Article summary/cover description |
| Cover Image Prompt | Text | AI prompt for cover image |
| Image Prompts | Text | AI prompts for article images |
| Cover Image URL | URL | Generated cover image link |
| Image URLs | Media | Generated image links |
| Typeset Content | Rich Text | Typeset HTML content |
| Publish Status | Single Select | Pending/Typeset/Published |
| Record ID | Text | Unique record identifier |

---

## Workflow Details

### 01_Content Generation

**File**: `01_内容生成.n8n`

Receives topic input and generates complete WeChat articles using DeepSeek AI.

#### Node Flow

```
Webhook → Set Parameters → Get Info → Simple Memory → DeepSeek Chat → Extract Content → Write to Feishu
```

#### Key Node Configuration

| Node Name | Type | Key Config |
|-----------|------|------------|
| Webhook | webhook | path: wechat-public-account-content |
| Get Info | HTTP Request | GET http://172.18.8.176:5678/webhook/information-everyday |
| DeepSeek Chat | LangChain | model: deepseek-reasoner |

---

### 02_Image Generation

**File**: `02_图片生成.n8n`

Generates cover images and article images using Alibaba Cloud Tongyi Wanxiang.

#### Node Flow

```
Webhook → Set Parameters → Condition Check → Extract Prompts → Wanxiang Generation → Image Binary → Upload to Feishu → Write to Table
```

#### Key Node Configuration

| Node Name | Type | Key Config |
|-----------|------|------------|
| Webhook | webhook | path: wechat-public-account-image |
| Condition Check | IF | Check if title/summary is empty |
| Wanxiang | HTTP Request | model: wan2.6-t2i |
| Upload to Feishu | Feishu | operation: space:upload |

---

### 03_Typesetting

**File**: `03_排版.n8n`

Converts generated content to WeChat-compatible HTML format.

#### Typesetting Rules

- Headings: `<h2>` tags, bold and centered
- Body: `<p>` tags, line-height 1.75
- Images: `<img>` tags, max-width 100%
- Paragraphs: `<p><br></p>` spacing

---

### 07_Upload to WeChat Drafts

**File**: `07_上传至微信草稿.n8n`

Uploads typeset content to WeChat official account drafts.

#### Node Flow

```
Manual Trigger → Set Parameters → Get Feishu Token → Split Out → Loop → Upload Article → Write Status
```

---

## API Integration

### Feishu API

```javascript
// Tenant Access Token
POST https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal
{
  "app_id": "cli_a9d6cb9992b9dbcc",
  "app_secret": "{{$credentials.feishuAppSecret}}"
}
```

### DeepSeek API

```javascript
{
  "model": "deepseek-reasoner",
  "messages": [
    {"role": "system", "content": "You are a professional WeChat content creator..."},
    {"role": "user", "content": "{{$json.userInput}}"}
  ]
}
```

### Tongyi Wanxiang API

```javascript
{
  "model": "wan2.6-t2i",
  "input": {
    "messages": [{"role": "user", "content": [{"text": "{{$json.prompt}}"}]}]
  },
  "parameters": {
    "prompt_extend": true,
    "watermark": false,
    "size": "1280x720"
  }
}
```

---

## Configuration

### Shared Configuration

```javascript
{
  "feishu_token": "SQ4CbAzhfa8rRUsFxRHcOsZ3npd",
  "bitable_id": "tbl2QoWRiJ6uavHM",
  "view_id": "vewSweWW3T",
  "app_id": "cli_a9d6cb9992b9dbcc"
}
```

---

## Troubleshooting

### Common Issues

#### 1. Webhook Trigger Failed

- Check if workflow is activated
- Verify webhook URL is correct
- Check firewall/network configuration

#### 2. API Call Timeout

- Increase timeout in HTTP Request node
- Add retry mechanism
- Check external service status

#### 3. Data Write Failed

- Verify API credentials are valid
- Confirm field type matching
- Check error response message

---

## License

MIT License - See [LICENSE](../LICENSE)
