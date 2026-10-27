# 05 · API 端点清单

> 来源：`embed` 源码中的全部 fetch 调用 + HAR 实测。同源 `https://text-generation.perchance.org`。身份全部走 URL 查询参数，无 cookie/Authorization。

## 核心端点

### POST /api/generate（核心生成）

Query 参数：`browserId`、`userKey`、`thread`、`requestId`（格式 `aiTextCompletion<随机数>`）、`__cacheBust`

Request body（JSON，`content-type: text/plain;charset=UTF-8`）：

```json
{
  "instruction": "<完整 prompt，字符串或 parts 数组>",
  "startWith": "[[Chloe]]:",
  "stopSequences": ["\n\n[[", "\n[["],
  "generatorName": "ai-character-chat",
  "instructionTokenCount": 1109,
  "startWithTokenCount": 5,
  "didMiddleOut": false,
  "instructionParts": [ { "type": "text", "text": "..." },
                        { "type": "image", "dataUrl": "...", "mime": "image/jpeg" } ]
}
```

响应：**SSE 流**。行协议：
- `t:<JSON 字符串>` — 文本块（自定义协议行）
- `data:<JSON>` — 状态对象/文本块

响应头：`x-new-user-key`（key 轮换）、`x-should-reverify: 1`（要求重验）。

错误响应（JSON 状态对象）：

| status | 含义 |
|---|---|
| `network_busy` | 服务端繁忙 |
| `invalid_key` | userKey 失效 → 需重验 |
| `invalid_browser_id` | browserId 无效 |
| `client_update_required` | 客户端版本过旧 |
| `waiting_for_prev_request_to_finish` | 服务端排队，附 `currentRequestId` / `newRequestId` |
| `invalid_data` | 请求体非法 |

### GET /api/verifyUser（两阶段）

- 第一阶段：`?browserId&thread&__cacheBust`（无 token）
- 第二阶段：`&token=<turnstile token>`
- 响应：`{"status":"success"|"already_verified", "userKey", "attemptedFullVerification"}` 或 `fetch_failure` 等

### GET /api/checkUserVerificationStatus

`?userKey&__cacheBust` → `{"status":"verified"|"need_reverification_soon"|...}`

### POST /api/createTicket

`?browserId&userKey&thread&__cacheBust`，body `{"kind":"generateText","generatorName":"..."}` → `{"status":"success","ticket","expiresAt"}`；错误：`invalid_key` / `need_reverification_soon` / `too_many_requests` / `too_many_tickets` / `client_update_required` / `invalid_browser_id`

### GET /api/awaitExistingGenerationRequest

`?userKey&__cacheBust`，客户端 `AbortSignal.timeout(20s)` → 等待前一个请求完成（长轮询）

### GET /api/checkRequestId

`?requestId&userKey` → `{"stillWorking": boolean}`

## 辅助端点

| 端点 | 方法 | 说明 |
|---|---|---|
| `/api/rateGeneratedText?__cacheBust` | POST | body `{instruction, startWith, generatedText, generatorName, score, reason}` |
| `/api/clientPerformanceAnalytics?__cacheBust` | POST | 性能遥测数组（每 2 分钟，>100 条截断） |
| `/api/waitingButNotWaiting?userKey&thread&...` | GET | "卡等待"诊断上报（recentDebugEvents、promptHashes 等） |
| `/api/verificationDebug?status=...` | GET | 验证调试信标 |
| `perchance.org/api/getCommunityData` | GET | 公开社区统计（可缓存 120s） |
| `perchance.org/api/ss` | POST | 使用统计，纯文本 body：`1\n\n\n20753,ai-character-chat,30` |
| `perchance.org/api/cv?generatorName=...&isFromEmbed=1` | GET | 缓存校验 |

## 外部依赖

| URL | 用途 |
|---|---|
| `huggingface.co/.../DeepSeek-R1-0528/tokenizer.json`（备用 `user.uploads.dev/file/...`） | tokenizer 加载 |
| `cdn.jsdelivr.net`（@xenova/transformers） | tokenizer 库兜底 |
| `challenges.cloudflare.com/turnstile/v0/api.js` | Turnstile 挑战 |
