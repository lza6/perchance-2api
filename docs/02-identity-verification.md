# 02 · 身份与验证机制

> 来源：`embed`（verifyUser，约 1270-1468 行）、`generation-identity-v2.js`。

## 身份标识

| 标识 | 格式 | 生成方式 | 存储 |
|---|---|---|---|
| `browserId` | 32 位 hex | `crypto.getRandomValues(Uint8Array(16))` | `localStorage["generation-v2-browser"]` |
| `userKey` | 64 位 hex | 服务端在 verifyUser 成功后颁发 | `localStorage["generation-v2:<browserId>:userKey-<thread>"]` |

要点（`generation-identity-v2.js`）：
- **无指纹采集**——v2 身份就是随机 UUID + localStorage，非 canvas/字体指纹
- 初始化用 `navigator.locks`（Web Locks API）跨标签页互斥，失败回退直接读
- `storage` 是 Proxy：所有键写入 `localStorage["generation-v2:<id>:<key>"]` 命名空间；写入失败（隐私模式）回退内存 Map
- `setKey(thread, key, expectedId)`：要求 identityId 匹配且 key 为 64 hex 才写入，否则丢弃（防伪造/防旧 embed 覆盖）
- **不用 cookie**，运行期验证状态全在内存

## verifyUser 流程

```
verifyUser(thread, opts):
1. 并发去重：同线程已有验证 Promise 则复用；超 60s 强制 location.reload()
2. 已有 key 快速路径：
   - 15s 内查过 → 直接返回
   - GET /api/checkUserVerificationStatus?userKey=...
   - status == "verified"          → 通过
   - status == "need_reverification_soon" → 不通知父页，继续完整验证
3. 无 token 尝试（循环重试，间隔 1-3s）：
   - GET /api/verifyUser?browserId&thread
   - status == "success" | "already_verified" → 通过
     （服务端对"已知浏览器"可免挑战放行——客户端无 token 参与）
   - 失败 → 继续循环，直到距上次尝试 >10s 且
     anotherEmbedIsVerifying_lastActiveTime > 5s
4. Cloudflare Turnstile 挑战（唯一挑战形式，无 PoW/验证码）：
   - 动态加载 challenges.cloudflare.com/turnstile/v0/api.js
   - sitekey: 0x4AAAAAAAJn3pYzPx4ATVOt
   - 渲染到隐藏 div #cfTurnstileCtn-<thread>（0..9 共 10 个槽）
   - token 超时 60s 放弃并上报 /api/verificationDebug?status=client-timeout
   - 连续 2 次超时或 2 次 error 回调 → location.reload()
5. 带 token 验证：
   - GET /api/verifyUser?browserId&token&thread
   - success → generationIdentity.setKey(thread, result.userKey)
              → postMessage {type:"verified"} 给父页面
6. 保活：后台守护循环每 2 分钟对 thread 0 低优先级 verifyUser；
   /api/generate 响应头 x-should-reverify: 1 也会触发
```

## key 轮换

- `/api/generate` 响应头 `x-new-user-key: <新key>` → 客户端 `setKey()` 更新
- 含义：**userKey 是短期会话票据而非长期凭据**，抓包得到的 key 很快失效

## 卡死自愈

- 每 5s 轮询：若 30s 内无验证活动且连续 10 次如此 → 直接向父页发 `verified`（摆脱"卡在 verifying"状态）

## 验证调试上报

`GET /api/verificationDebug?status=...`，status 取值：`fail`、`client-script-load-fail`、`client-timeout`、`error-callback`、`success`。
