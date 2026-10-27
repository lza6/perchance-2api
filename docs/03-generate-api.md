# 03 · /api/generate 协议：流式解析与重试矩阵

> 来源：`embed` 源码（`startStreaming` 约 691-972 行、`eventStreamToAsyncIterator` 约 1093-1239 行）。以下阈值为源码字面量（静态确认）；个别分支为合理推断，已标注。

## 请求生命周期总览

```
父页面 startStream {requestId, postData}
  → 附件编码（如有 vision parts）
  → middle-out 裁剪（见 04 文档）
  → startStreaming(postData, requestId)
      ├─ 选线程槽：moveToLeastRecentlyUsedThread()（2 槽 LRU）
      ├─ 限速：lastGenerateRequestTime 距今 <500ms → 等待
      ├─ 若该线程有前请求在跑 → 先等/协调
      └─ POST /api/generate → eventStreamToAsyncIterator()
            ├─ 逐块解析 → postMessage streamData
            ├─ 完成 → streamEnd
            └─ 错误/状态 → 见下方矩阵
```

## SSE 解析细节

- 响应体逐行解析：`t:<JSON 字符串>` 与 `data:<JSON>` 两种行（`t:` 为自定义协议行）
- 每块 yield `{text}`，累积文本经 postMessage `streamData` 转发
- 超时参数：

| 参数 | 值 |
|---|---|
| 建连超时 | 60-90s（AbortSignal.timeout） |
| body 停滞 watchdog | 60s（`threadLastStreamedChunkTime`） |
| 请求最长寿命 | 480s（僵尸请求放弃上限） |
| 超时前探查 | `GET /api/checkRequestId` → `stillWorking` |

## 错误状态处理矩阵

| 服务端 status | embed 的处理 |
|---|---|
| `invalid_key` | 清除本地 key → 触发 verifyUser 重验 → 重验通过后重试请求 |
| `invalid_browser_id` | 上报调试信标；需重新初始化身份（客户端无法自行恢复） |
| `network_busy` | 等待后重试（网络错误类，计入最多 5 次续写重试） |
| `client_update_required` | 终止——客户端脚本版本过旧，提示更新 |
| `waiting_for_prev_request_to_finish` | 记录 `currentRequestId`/`newRequestId`；调 `/api/awaitExistingGenerationRequest`（≤20s 长轮询）或直接采用 newRequestId 续流 |
| `invalid_data` | 终止，`streamError`（请求体非法，重试无意义） |
| 响应头 `x-should-reverify: 1` | 后台低优先级 verifyUser |
| 响应头 `x-new-user-key` | `setKey()` 更新本地 key |

## 断流续传（continuation）

断流后**不从头重试**，而是从已收文本续传：

1. 取已累积的生成文本
2. 构造 continuation postData：原 instruction + 已收文本追加进上下文（token 计数相应更新），`startWith` 调整为续写模式
3. 重新 POST /api/generate
4. 网络错误类续写重试上限 **5 次**

## stop / keepalive

- `stopStream`：requestId 加入 `requestIdsToStop` → 流循环检测到后 abort，错误码 `abort_due_to_stop_stream`
- 父页面需每 <10s 发 `streamKeepAlive`，否则该请求被判 stale 处理；未知 requestId 收到 >20 次 keepalive → 报 `unknown_request_id`

## 并发协调

- 每用户 2 个线程槽（thread 0/1），LRU 选择；`threadGenerationInProgress-<n>` 跨标签页提示
- 同线程前请求未完成时：优先 `awaitExistingGenerationRequest`（服务端排队），避免并发互踩
