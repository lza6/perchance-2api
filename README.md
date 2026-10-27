# perchance-2api

对 Perchance 文本生成服务（`text-generation.perchance.org`）的**纯静态逆向分析**资料库。

> ⚠️ **仅供学习研究**。本仓库不含任何可运行代码、不含任何可用凭据（抓包样本已全部脱敏），仅包含协议分析文档与抓包快照。请勿用于违反 Perchance 服务条款的用途。

## 这是什么

Perchance 的 AI 聊天（ai-character-chat）没有官方 API，但其网页客户端通过一组公开 HTTP 端点完成验证与文本生成。本仓库通过静态分析网页源码快照 + HAR 抓包，完整还原了：

- 用户验证体系（`browserId` / `userKey` / Cloudflare Turnstile）
- 文本生成协议（`/api/generate` SSE 流式接口）
- prompt 拼装模板与上下文裁剪算法（middle-out）
- 全部已观察到的 API 端点、错误码、限流机制
- `postMessage` iframe 桥接协议（embed ↔ 父页面）

## 仓库结构

```
docs/
  01-overview.md            # 系统全貌与分层架构
  02-identity-verification.md # 身份与验证机制（browserId/userKey/Turnstile）
  03-generate-api.md        # /api/generate 协议、SSE 流、错误码、重试矩阵
  04-prompt-structure.md    # prompt 模板、stopSequences/startWith、token 裁剪
  05-api-endpoints.md       # 全部端点清单与参数表
  06-antibot-limits.md      # 反滥用机制与限流参数
  07-postmessage-protocol.md # embed iframe 与父页面消息协议
  08-file-inventory.md      # 抓包文件身份速查表
captured-source/            # 抓包的公开网页源码快照（.txt 后缀）
captured-samples/           # 脱敏 HAR + 完整 prompt 样本
```

## 免责声明

- `captured-source/` 内文件为通过浏览器正常访问公开网页时获取的源码快照，版权归原作者（Perchance / Cloudflare / Google / 各开源库）所有，仅作研究存档。
- 分析文档基于静态阅读，服务端行为（验证放行策略、限流阈值等）为合理推断，未实测。
- 使用本项目信息产生的任何后果由使用者自行承担。
