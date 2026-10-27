# 01 · 系统全貌与分层架构

> 来源：`embed`（93KB 未压缩 JS）、`ai-character-chat` 页面 HTML、`perchance.org.har` 静态分析。

## 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│ perchance.org/ai-character-chat （编辑器/生成器外壳页 SPA）      │
│  ├─ perchance-engine  — $output 概率列表引擎（与 AI 无关）       │
│  ├─ appExtras.js      — 插件注册表、perchanceEditorClientId     │
│  └─ 实际插件代码在 ai-character-chat-dependencies-v1            │
│     （角色卡数据结构、prompt 拼装、UI 组件都在此包，未抓取）        │
└──────────────┬──────────────────────────────────────────────┘
               │ postMessage：startStream / stopStream /
               │ verifyUser / verified / streamData / streamEnd
┌──────────────▼──────────────────────────────────────────────┐
│ text-generation.perchance.org/embed （隐藏 iframe）            │
│  页面标题 "AI Text Plugin Verifier"                            │
│  ├─ verifyUser()        — Turnstile 挑战 + userKey 获取/保活    │
│  ├─ middleOut()         — token 上下文裁剪（保留首尾，中间删）      │
│  ├─ startStreaming()    — 调 /api/generate 的流式循环+重试矩阵    │
│  ├─ createTicket()      — 服务端插件一次性票据                   │
│  ├─ encodeAttachmentBlob() — 视觉附件重编码（canvas→JPEG）        │
│  └─ maxThreadsPerUser=2、500ms 客户端限速、2min 验证心跳           │
└──────────────┬──────────────────────────────────────────────┘
               │ HTTPS；身份全部走 URL 查询参数，无 cookie
┌──────────────▼──────────────────────────────────────────────┐
│ text-generation.perchance.org 后端                            │
│  /api/verifyUser · /api/generate · /api/createTicket          │
│  /api/checkUserVerificationStatus · /api/awaitExisting...     │
└─────────────────────────────────────────────────────────────┘
```

## 核心调用链

1. **身份初始化**（`generation-identity-v2.js`）：生成 `browserId`（32 位 hex）→ localStorage
2. **验证**：embed iframe 调 `/api/verifyUser`（先无 token，失败加 Turnstile token）→ 获得 `userKey`（64 位 hex）
3. **生成**：插件拼好完整 prompt → postMessage `startStream` → embed 做 middle-out 裁剪 → `POST /api/generate` → SSE 流逐块 postMessage 回父页面
4. **保活**：每 2 分钟后台 verifyUser；响应头 `x-new-user-key` 轮换 key；`x-should-reverify: 1` 触发重验

## 关键结论

- **无 cookie、无付费 API key**——整套体系是 browserId/userKey + Turnstile 门控的匿名配额
- 真正要对接的协议载体是 `embed` iframe（不是 perchance-engine，那是概率列表引擎，全文检索 `textGeneration|stopSequences` 均 0 命中）
- 模型侧客户端可见的线索：tokenizer 为 **DeepSeek-R1-0528**（huggingface 加载，`user.uploads.dev` 备用），上下文上限 `maxContextTokens = 8000 - 1024`（约 6976 tokens）
