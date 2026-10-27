# 08 · 抓包文件身份速查表

> `captured-source/` 内为浏览器访问公开网页时的源码快照（加 `.txt` 后缀便于在线查看），版权归原作者所有。

| 快照文件 | 原始身份 | 说明 |
|---|---|---|
| `embed.js.txt` | `text-generation.perchance.org/embed` | **最关键文件**：AI Text Plugin Verifier iframe——验证 + 生成代理协议的完整实现 |
| `generation-identity-v2.js.txt` | 同名 | browserId/userKey 本地身份存储层（`navigator.locks` + localStorage 命名空间 Proxy） |
| `ai-character-chat.page.html.txt` | `perchance.org/ai-character-chat` | 生成器编辑器外壳页 SPA（CodeMirror 编辑器、预览 iframe、保存/版本管理）。插件源码本体在 `ai-character-chat-dependencies-v1`（未抓取） |
| `perchance-engine-491bf81418aa4b69.js.txt` | Perchance 引擎 | 概率列表引擎运行时（`window.PERCH`/`PERCH2`，树构建/求值）。**不含 AI 逻辑** |
| `appExtras.js.txt` | perchance.org 页面脚本 | 插件注册表（ai-text-plugin、text-to-image-plugin 等）、编辑器设备 ID |
| `comlink.js.txt` | Google Comlink | Worker/MessageChannel RPC 库（Apache-2.0），无 Perchance 业务逻辑 |
| `tokenizers-0.1.2.js.txt` | HuggingFace tokenizers JS 绑定 | WASM 封装的 BPE/WordPiece/Unigram tokenizer |
| `index.min.mjs.txt` | esm.sh 产物 | Unicode 字符属性/区间数据 bundle（tokenizer 依赖） |
| `editors.bundle.min.js.txt` | CodeMirror 6 bundle | 在线编辑器内核 |
| `eslint-linter-browserify-v9.14.0.mjs.txt` | ESLint linter（esm.sh 打包） | 浏览器内代码校验 |
| `htmlparser2-v9.1.0.mjs.txt` | htmlparser2（esm.sh 打包） | HTML 解析器（附 emoji 表数据混入） |
| `cloudflare-rum.js.txt` | Cloudflare RUM | Real User Monitoring beacon |

## captured-samples/

| 文件 | 说明 |
|---|---|
| `perchance.org.redacted.har` | HAR 抓包（**已脱敏**：userKey/browserId 替换为 `<USER_KEY_REDACTED>`/`<BROWSER_ID_REDACTED>`）。含 checkUserVerificationStatus、两条 /api/generate（POST body 完整保存）、getCommunityData、ss、cv 共 6 个 entries |
| `full_instruction.sample.txt` | 实测的完整 `/api/generate` `instruction` prompt（通用指导 + 角色描述 + STRICT RULES + `<MESSAGES>` + TASK） |

## 未抓取的关键材料

- `ai-character-chat-dependencies-v1` — 插件源码本体（角色卡数据结构、prompt 拼装代码、UI 组件）
- 获取方式：在 ai-character-chat 页面的预览 iframe 中查看其源码文本，或抓取对应生成器资源
