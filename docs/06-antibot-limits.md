# 06 · 反滥用与限流机制

> 来源：`embed` 源码静态分析。阈值为客户端可见字面量，服务端策略为合理推断。

## 客户端可确认的机制

| 机制 | 参数 | 说明 |
|---|---|---|
| Cloudflare Turnstile | sitekey `0x4AAAAAAAJn3pYzPx4ATVOt` | 唯一挑战形式（行为式，通常无感），token 单次使用 |
| 线程槽 | `maxThreadsPerUser = 2` | 每用户最多 2 个并发生成槽，LRU 轮转 |
| 客户端限速 | 500ms | 距上次生成请求 <500ms 强制等待 |
| 生成间隔提示 | `threadGenerationInProgress-<n>` | localStorage 跨标签页提示 |
| key 绑定 | identityId 校验 | `x-new-user-key` 轮换 + `setKey` 的绑定校验使偷来的 key 难跨身份使用 |
| 单次票据 | `createTicket` | 一次性，`too_many_tickets` 配额 |
| 僵尸请求上限 | 480s | 生成请求最长寿命 |
| 流 watchdog | 60s body 停滞 | 超时判定连接死亡 |
| 连接超时 | 60-90s | SSE 建连超时 |
| keepalive stale | 10s 无心跳判 stale；>20 次未知 id 判 `unknown_request_id` | 防伪造 requestId |

## 服务端状态码（反滥用相关）

`too_many_requests` · `too_many_tickets` · `network_busy` · `invalid_key` · `invalid_browser_id` · `client_update_required` · `waiting_for_prev_request_to_finish`

## 行为遥测

- `/api/verificationDebug?status=...` — 验证成败信标
- `/api/waitingButNotWaiting` — 卡等待诊断（recentDebugEvents、promptHashes、`djb2Hash` prompt 哈希）
- `/api/clientPerformanceAnalytics` — 每 2 分钟性能数据
- Cloudflare RUM（`/cdn-cgi/rum` sendBeacon）

## 防调试/防篡改细节

- `encodeAttachmentBlob` 含 **canvas 投毒金丝雀检测**（检测防指纹扩展对 canvas 的干扰）
- `hiddenSafeWait`：用 Blob URL Worker 实现后台标签页不被节流的定时器（隐藏 iframe 中验证不被浏览器限速拖慢）
- postMessage 校验：`e.source === window.parent` 且 `e.origin.endsWith(".perchance.org")`

## 推断（待验证）

- 无 token 验证通道对"低频/正常浏览行为"直接放行；高频或异常行为触发 Turnstile
- 具体限流阈值（每分钟/每天次数）服务端控制，客户端不可见
