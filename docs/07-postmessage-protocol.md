# 07 · postMessage 协议（embed iframe ↔ 父页面）

> 来源：`embed` 的 `message` 监听器（约 128-599 行）。

安全校验：接收方要求 `e.source === window.parent` 且 `e.origin.endsWith(".perchance.org")`。发送 targetOrigin 为记录的 `window.perchanceGeneratorOrigin`（首条 `embedIsReady` 用 `"*"`）。

## 父页面 → iframe

| type / command | 载荷 | 行为 |
|---|---|---|
| `preload` | — | 低优先级 verifyUser（不显示验证界面） |
| `verifyUser` | — | 完整验证（无限重试，失败 3 次后等 20s） |
| `startStream` | `{requestId, postData}` | 记录 requestId → 附件编码 → middle-out 裁剪 → 最多 5 次网络错误续写重试调 `startStreaming` |
| `stopStream` | `{requestId}` | 加入 `requestIdsToStop`，流内 abort（错误 `abort_due_to_stop_stream`） |
| `streamKeepAlive` | `{requestId}` | 刷新 keepalive；10s 无刷新判 stale；keepalive >20 次仍未知 id 判 `unknown_request_id` |
| `rateGeneratedText` | `{instruction, startWith, generatedText, generatorName, score, reason}` | POST /api/rateGeneratedText |
| `command:"createTicket"` | `{kind:"generateText", generatorName, requestId}` | 回 `ticketCreated` / `ticketError` |

## iframe → 父页面

| type | 载荷 | 时机 |
|---|---|---|
| `embedIsReady` | — | iframe 初始化完成 |
| `verifying` | — | 验证开始（父页面控制广告遮挡层显隐） |
| `verified` | — | 验证通过 |
| `streamData` | `{requestId, value}` | 流式文本块 |
| `streamEnd` | `{requestId}` | 生成完成 |
| `streamError` | `{requestId, status}` | 错误 |
| `middleOut` | `{requestId, 截断统计}` | 触发上下文裁剪时 |
| `ticketCreated` | `{requestId, ticket, expiresAt}` | 票据成功 |
| `ticketError` | `{requestId, error}` | 票据失败 |
