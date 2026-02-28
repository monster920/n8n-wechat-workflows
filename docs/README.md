# n8n 微信公众号自动化 - 技术文档

本文档详细介绍每个工作流的技术实现、节点配置和数据流转逻辑。

## 目录

1. [系统架构](#系统架构)
2. [技术栈](#技术栈)
3. [数据模型](#数据模型)
4. [工作流详解](#工作流详解)
5. [API 集成](#api-集成)
6. [配置说明](#配置说明)
7. [故障排查](#故障排查)

---

## 系统架构

### 整体架构图

```
                    ┌─────────────────┐
                    │   飞书多维表格    │
                    │  (数据存储层)    │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  01_内容生成   │    │  02_图片生成   │    │  03_排版       │
│  (Web API)    │    │  (Web API)    │    │  (Web API)    │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                    │
        └────────────────────┴────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  微信公众号草稿箱 │
                    │   (发布层)       │
                    └─────────────────┘
```

### 层级说明

| 层级 | 组件 | 职责 |
|-----|------|-----|
| 数据存储层 | 飞书多维表格 | 存储文章数据、图片信息、配置参数 |
| 处理层 | n8n 工作流 | 内容生成、图片处理、排版转换 |
| 发布层 | 微信公众号 API | 草稿箱发布 |

---

## 技术栈

### 核心组件

| 组件 | 版本 | 用途 |
|-----|------|-----|
| n8n | 1.0+ | 工作流引擎 |
| DeepSeek | - | AI 内容生成 |
| 通义万相 (wan2.6-t2i) | - | AI 图像生成 |
| 飞书开放 API | v3 | 多维表格数据管理 |
| 微信公众号 API | - | 草稿箱发布 |

### 外部服务

- **DeepSeek API**: https://platform.deepseek.com/
- **阿里云 DashScope**: https://dashscope.aliyuncs.com/
- **飞书开放平台**: https://open.feishu.cn/
- **微信公众平台**: https://mp.weixin.qq.com/

---

## 数据模型

### 飞书多维表格字段

```
| 字段名           | 类型     | 说明                    |
|-----------------|---------|------------------------|
| 文章标题         | 文本     | 公众号文章标题          |
| 文章内容         | 富文本   | 生成的正文内容          |
| 文章摘要         | 文本     | 文章摘要/封面描述       |
| 封面图提示词     | 文本     | 用于生成封面图的 AI 提示词 |
| 配图提示词       | 文本     | 用于生成配图的 AI 提示词   |
| 封面图 URL       | URL      | 生成的封面图链接        |
| 配图 URLs        | 多媒体   | 生成的配图链接列表       |
| 排版后内容       | 富文本   | 排版完成的 HTML 内容    |
| 发布状态         | 单选     | 待生成/待排版/已发布    |
| 记录 ID          | 文本     | 多维表格记录唯一标识     |
```

---

## 工作流详解

### 01_内容生成

**文件**: `01_内容生成.n8n`

#### 功能说明

接收主题输入，使用 DeepSeek AI 生成完整的公众号文章内容。

#### 节点流程

```
Webhook → 设置参数 → 获取信息 → Simple Memory → DeepSeek Chat → 提取内容 → 写入飞书
```

#### 关键节点配置

| 节点名称 | 类型 | 关键配置 |
|---------|------|---------|
| Webhook | webhook | path: wechat-public-account-content |
| 获取信息 | HTTP Request | GET http://172.18.8.176:5678/webhook/information-everyday |
| DeepSeek Chat | LangChain | model: deepseek-reasoner |

#### 数据流转

1. Webhook 接收 POST 请求，参数包含 `file` (飞书多维表格记录 ID)
2. 调用 `获取信息` 节点获取当日信息素材
3. 将信息输入 DeepSeek 生成文章
4. 提取关键内容字段
5. 写入飞书多维表格更新记录

---

### 02_图片生成

**文件**: `02_图片生成.n8n`

#### 功能说明

根据提示词使用阿里云通义万相生成封面图和文章配图。

#### 节点流程

```
Webhook → 设置参数 → 条件检查 → 提取提示词 → 通义万相生成 → 图片二进制 → 上传飞书 → 写入表格
```

#### 关键节点配置

| 节点名称 | 类型 | 关键配置 |
|---------|------|---------|
| Webhook | webhook | path: wechat-public-account-image |
| 条件检查 | IF | 检查标题、摘要是否为空 |
| 通义万相 | HTTP Request | model: wan2.6-t2i |
| 上传飞书 | Feishu | operation: space:upload |

#### API 调用

```bash
# 通义万相 API
POST https://dashscope.aliyuncs.com/api/v1/services/aigc/multimodal-generation/generation
Headers:
  Authorization: Bearer <your-api-key>
  Content-Type: application/json

Body:
{
  "model": "wan2.6-t2i",
  "input": {
    "messages": [{
      "role": "user",
      "content": [{"text": "<提示词>"}]
    }]
  },
  "parameters": {
    "prompt_extend": true,
    "watermark": false,
    "negative_prompt": "低分辨率, 肢体畸形, ..."
  }
}
```

---

### 03_排版

**文件**: `03_排版.n8n`

#### 功能说明

将生成的内容转换为微信公众号支持的 HTML 格式。

#### 节点流程

```
Webhook → 设置参数 → 获取飞书数据 → 内容提取 → HTML转换 → 美化样式 → 写入飞书
```

#### 排版规则

- 标题使用 `<h2>` 标签，加粗居中
- 正文使用 `<p>` 标签，行高 1.75
- 图片使用 `<img>` 标签，最大宽度 100%
- 段落间添加 `<p><br></p>` 间隔

---

### 04_标题内容处理

**文件**: `04_标题内容处理.n8n`

#### 功能说明

处理和优化文章标题，生成适合配图和 SEO 的标题变体。

---

### 05_排版图片生成

**文件**: `05_排版图片生成.n8n`

#### 功能说明

为排版后的文章生成额外的配图素材。

---

### 06_排版图片内容处理

**文件**: `06_排版图片内容处理.n8n`

#### 功能说明

处理排版阶段生成的图片，进行格式转换和优化。

---

### 07_上传至微信草稿

**文件**: `07_上传至微信草稿.n8n`

#### 功能说明

将最终排版完成的内容上传至微信公众号草稿箱。

#### 节点流程

```
手动触发 → 设置参数 → 获取飞书token → Split Out → Loop → 上传图文 → 写入状态
```

#### 关键节点配置

| 节点名称 | 类型 | 关键配置 |
|---------|------|---------|
| 获取飞书token | HTTP Request | POST https://open.feishu.cn/open-apis/auth/v3/tenant_access_token |
| 上传图文 | HTTP Request | POST https://api.weixin.qq.com/cgi-bin/draft/add |

#### 微信 API 调用

```bash
# 获取飞书 Tenant Access Token
POST https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal
Body:
{
  "app_id": "cli_a9d6cb9992b9dbcc",
  "app_secret": "<your-app-secret>"
}

# 上传至微信草稿箱
POST https://api.weixin.qq.com/cgi-bin/draft/add?access_token=<token>
Body:
{
  "articles": [{
    "title": "<标题>",
    "author": "<作者>",
    "content": "<HTML内容>",
    "digest": "<摘要>",
    "content_source_url": "<原文链接>",
    "thumb_media_id": "<封面图Media ID>"
  }]
}
```

---

## API 集成

### 飞书 API

#### Tenant Access Token 获取

```javascript
// n8n HTTP Request 节点配置
{
  "method": "POST",
  "url": "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal",
  "jsonBody": {
    "app_id": "cli_a9d6cb9992b9dbcc",
    "app_secret": "{{$credentials.feishuAppSecret}}"
  }
}
```

#### 多维表格操作

- **读取记录**: `GET /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records/{record_id}`
- **写入记录**: `PUT /open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records/{record_id}`
- **上传图片**: `POST /open-apis/drive/v1/files/upload`

### DeepSeek API

```javascript
// LangChain 节点配置
{
  "model": "deepseek-reasoner",
  "messages": [
    {
      "role": "system",
      "content": "你是一个专业的公众号内容创作者..."
    },
    {
      "role": "user",
      "content": "{{$json.userInput}}"
    }
  ]
}
```

### 通义万相 API

```javascript
// HTTP Request 节点配置
{
  "method": "POST",
  "url": "https://dashscope.aliyuncs.com/api/v1/services/aigc/multimodal-generation/generation",
  "authentication": "genericCredentialType",
  "genericAuthType": "httpBearerAuth",
  "jsonBody": {
    "model": "wan2.6-t2i",
    "input": {
      "messages": [{
        "role": "user",
        "content": [{"text": "{{$json.prompt}}"}]
      }]
    },
    "parameters": {
      "prompt_extend": true,
      "watermark": false,
      "size": "1280x720"
    }
  }
}
```

---

## 配置说明

### 共享配置

所有工作流使用相同的飞书凭证：

```javascript
{
  "飞书token": "SQ4CbAzhfa8rRUsFxRHcOsZ3npd",
  "多维表格ID": "tbl2QoWRiJ6uavHM",
  "viewID": "vewSweWW3T",
  "app_id": "cli_a9d6cb9992b9dbcc"
}
```

### 凭证配置

在 n8n 中配置以下凭证：

| 凭证名称 | 类型 | 字段 |
|---------|------|-----|
| feishuCredentialsApi | 飞书 API | app_id, app_secret |
| httpBearerAuth | Bearer Token | token |
| deepseekCredentials | DeepSeek | api_key |

---

## 故障排查

### 常见问题

#### 1. Webhook 触发失败

**症状**: 返回 404 或超时

**排查步骤**:
1. 确认工作流已激活
2. 检查 webhook URL 是否正确
3. 确认防火墙/网络配置

#### 2. API 调用超时

**症状**: 工作流长时间等待

**解决方案**:
1. 在 HTTP Request 节点增加 timeout 配置
2. 添加重试机制 (retryOnFail)
3. 检查外部服务状态

#### 3. 数据写入失败

**症状**: 飞书/微信 API 返回错误

**排查步骤**:
1. 检查 API 凭证是否有效
2. 确认字段类型匹配
3. 查看错误返回信息

### 日志级别

推荐在生产环境设置日志级别为：

```javascript
{
  "logLevel": "info",
  "executionLogLevel": "manual"
}
```

---

## 更新日志

查看 [CHANGELOG.md](./CHANGELOG.md) 了解版本更新历史。

---

## 相关文档

- [n8n 官方文档](https://docs.n8n.io/)
- [飞书开放平台文档](https://open.feishu.cn/document/)
- [微信公众平台文档](https://developers.weixin.qq.com/doc/)
- [DeepSeek API 文档](https://platform.deepseek.com/docs)
- [阿里云 DashScope 文档](https://dashscope.aliyuncs.com/completion-of-documentation/)
