# 04 · Prompt 结构与上下文裁剪

> 来源：HAR 抓包的完整 `instruction`（见 `captured-samples/full_instruction.sample.txt`）+ `embed` 的 middleOut 实现。

## Prompt 模板（ai-character-chat 实测）

单条 `instruction` 自上而下分五段：

```
Below are some general notes, and then some MESSAGES between several
characters. Use this context to complete the '>>> TASK' which is specified
at the end of this instruction.

Important general notes:
 * IMPORTANT: Avoid broken or robotic sentences. Write in a natural,
   readable style...
 * IMPORTANT: There must be *two* blank lines before each '[[Name]]:' ...
Also:
 * Guidelines for roleplays:  （不破角色、暖色调、避免 purple prose）
 * Guidelines for stories:    （消息 2-4 段、永不收尾、反重复模式：
   长度/结构/措辞动态变化）

---
NOTE: In case it's useful here's a description of the **<角色名>** character:
<角色 brainDump>

STRICT RULES:
- <创作者写的硬规则，第一人称、*动作*用星号、对白用引号、不破第四面墙等>

---
The messages so far:

<MESSAGES>
[[Chloe]]: Hi! 😊

[[Anon]]: <用户消息>
</MESSAGES>

Note: All story characters should speak/act for themselves only...

>>> TASK: Your task is to write the next 3 messages in this chat.
```

## 输出控制三件套（POST body 字段）

| 字段 | 实测值 | 作用 |
|---|---|---|
| `startWith` | `"[[Chloe]]:"` | 强制模型以该前缀开头输出（决定哪个角色说话） |
| `stopSequences` | `["\n\n[[", "\n[["]` | 遇到下一条消息分隔符即停（客户端+服务端双重生效） |
| `generatorName` | `"ai-character-chat"` | 服务端插件标识 |

## 角色卡数据结构（从 prompt 推断）

插件源码本体在 `ai-character-chat-dependencies-v1`（未抓取），从 prompt 可确定字段至少有：
- 名称（`[[Chloe]]`）
- 描述 / brainDump（自由文本）
- STRICT RULES 列表
- 头像（经 text-to-image-plugin / upload-plugin）
- greeting 开场消息
- 用户名（`[[Anon]]`，可自定义）

## 上下文裁剪（middle-out，embed 实现）

```
maxContextTokens = 8000 - 1024        # 有图片附件再扣 IMAGE_TOKEN_COST
                                    # 即约 6976 tokens

算法：
1. fast middle-out：用 approxCharsPerToken（基准 3.9，中文/法语有专门
   估算）字符数估算，若超过 1.6 × maxTokens × charsPerToken 先粗裁
2. 精确裁剪：加载 tokenizer（DeepSeek-R1-0528，huggingface →
   user.uploads.dev 备用 → @xenova/transformers 兜底）计数
3. middleOut()：保留首尾，中间替换为 "\n\n[...]\n\n"
4. 循环：instructionTokenCount + startWithTokenCount 仍超限 →
   对 instruction / startWith 中较大者按 0.95 系数收缩，重复
5. Analytics 记录 didInitialFastMiddleOut / didTokenMiddleOut /
   didMiddleOut
```

请求 body 中对应上报：`instructionTokenCount`、`startWithTokenCount`、`didMiddleOut`、（vision 时）`middleOutSizeAnalytics`。

## 视觉（vision）支持

- `instruction` 可为数组：`[{type:"text", text}, {type:"image", dataUrl, mime}]` → body 字段 `instructionParts`
- `encodeAttachmentBlob()`：canvas 重编码 PNG→JPEG 质量 0.88，最长边 768px / 200-250KB 上限
- 最多 1 张图
