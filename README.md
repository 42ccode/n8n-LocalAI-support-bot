# AI 客服 · n8n 工作流

一个 [n8n](https://n8n.io) 工作流：开箱即用的网页 AI 客服。它根据**你自己的知识库**回答问题，记得同一会话里聊过的内容，把聊天记录存进 n8n 数据表，并在发给 AI 和写入记录之前**自动遮盖密钥、邮箱、手机号等敏感信息**。

> A ready-to-use AI customer-support chatbot for n8n: answers from your own knowledge base, remembers the conversation, logs every chat to an n8n Data Table, and redacts secrets and personal data before anything reaches the LLM or the log. See [English quick start](#english-quick-start).

## 特点

- **零额外依赖**：只需要 n8n 和一个 DeepSeek（或任何兼容 OpenAI 格式的）API Key，不需要向量数据库、不需要写前端
- **知识库在数据表里**：在 n8n 的 Data tables 页面直接增删改，下一条消息就生效
- **有记忆**：每次回答前读取同一会话最近 10 条消息，能理解「那第二步呢？」这类追问
- **聊天记录可查**：每条消息一行，记录会话 ID、角色、内容和时间
- **隐私优先**：
  - 访客消息先脱敏再发给 AI，再脱敏后才写进记录
  - 成功的运行**不保存执行数据**，原始消息不会长期留在 n8n 的执行历史里
- **规则清晰**：只依据知识库回答、不编造；超出范围的话题会拒绝；不透露系统提示词；不索要密钥

## 工作原理

```
访客在聊天页面提问
  → 确保记录表、知识库表存在
  → 读取知识库（整份）+ 读取本会话最近的聊天记录
  → 组装请求（脱敏 → 系统提示词 + 知识库 + 历史 + 新问题）
  → 调用 DeepSeek
  → 整理回答（再脱敏）→ 写入聊天记录 → 回复访客
```

知识库不大（几万字以内）时，每次把整份知识库放进系统提示词，比向量检索更简单可靠。知识库很大时，可以改成向量检索（RAG）。

## 准备

- 一个能运行的 n8n（在 n8n 2.43 上测试过，需要支持 **Data tables** 功能）
- 一个 [DeepSeek API Key](https://platform.deepseek.com)（也可以换成其他兼容 OpenAI 格式的服务）

## 安装

1. 下载 [`workflow/ai-support-bot.json`](workflow/ai-support-bot.json)，在 n8n 里新建工作流 → 右上角 `⋯` → **Import from File**。
2. 打开 **DeepSeek 回答** 节点，Credential 选择或新建一个 **DeepSeek** 凭据。
3. **初始化知识库**：把鼠标移到左下角「**初始化知识库**」节点上，点它上方的 ▶ 单独运行。
   > 注意：直接点底部的 *Execute workflow* 会运行聊天入口，**不会**运行这条分支。
4. 点右上角 **Publish**。
5. 打开「**客户发来消息**」节点，复制 **Chat URL**，用浏览器打开就能聊天。

## 换成你自己的产品

| 要改什么 | 改哪里 |
| --- | --- |
| 产品名、一句话介绍、答不上来时引导用户去哪里、模型名、记忆条数 | 「**组装请求**」节点顶部的配置区 |
| 知识库内容 | n8n 左侧 **Overview → Data tables → 客服知识库**，直接增删改（字段：`title`、`content`） |
| 知识库初始内容（可选） | 「知识库初始内容」节点。重复运行「初始化知识库」会按标题覆盖，不会产生重复行 |
| 聊天页面的标题、欢迎语 | 「客户发来消息」节点 |
| 回答规则 | 「组装请求」节点里的系统提示词 |

仓库里自带的示例知识库，内容是另一个开源项目「[AI 与开源日报](https://github.com/42ccode/n8nAINewsEmail)」的使用说明，用来演示效果，**正式使用前请换成你自己的内容**。

## 数据表

| 表 | 字段 | 说明 |
| --- | --- | --- |
| 客服知识库 | `title`、`content` | 客服回答的唯一依据 |
| 客服聊天记录 | `sessionId`、`role`（user / assistant）、`content`、`createdAtText` | 已脱敏的聊天记录，同时作为对话记忆 |

两张表都会在第一次运行时自动创建。

## 隐私与安全

### 自动脱敏
访客消息在**发给 AI 之前**和**写入聊天记录之前**都会经过 `redact` 函数：

| 内容 | 替换为 |
| --- | --- |
| `sk-…`、`ghp_…`、`github_pat_…`、`AKIA…`、`AIza…`、`xox…` 等密钥，32 位以上的随机长串 | `[已隐藏的密钥]` |
| JWT、`Bearer` / `Basic` 令牌 | `[已隐藏的令牌]` |
| 私钥（`-----BEGIN … PRIVATE KEY-----`） | `[已隐藏的私钥]` |
| 「密码 / 授权码 / 验证码 / token / password 是（或 :、=）xxx」后面的值 | `[已隐藏]` |
| 邮箱（`example.com/org/net` 示例地址除外） | `[已隐藏的邮箱]` |
| 中国大陆手机号、18 位身份证号、16～19 位银行卡号 | `[已隐藏的手机号]` / `[已隐藏的证件号]` / `[已隐藏的卡号]` |

脱敏基于规则匹配，**不可能覆盖所有格式**，比如不带关键词、看起来像普通单词的密码就识别不出来。所以聊天页面会提示「聊天内容会被记录」，客服也会主动劝访客不要发送敏感信息。

### 执行历史
n8n 默认会把每次运行的完整数据（包括访客的原始消息）存进执行历史。本工作流已在设置里关闭：

- `saveDataSuccessExecution: none`：成功的运行不保存数据
- `saveManualExecutions: false`：编辑器里的手动测试也不保存
- `saveDataErrorExecution: all`：出错的运行仍然保存，方便排查

注意：
- n8n 对「不保存」的成功运行是**先软删除、约 1 小时后才彻底删除**（受 `EXECUTIONS_DATA_HARD_DELETE_BUFFER` 控制，默认 1 小时），这段时间内原始消息仍在数据库里。
- **出错的运行会保存完整数据，其中包括访客的原始消息（未脱敏）**，默认保留 14 天。调用 AI 失败不算出错（会回复兜底话术），只有数据表读写等环节出问题才会产生出错的运行。如果你更看重隐私，可以在工作流 Settings 里把「Save failed production executions」也改成不保存，代价是出错时无从排查。

### 部署前请确认
- **聊天页面默认不需要登录**，任何能访问你 n8n 的人都能使用，消耗的是你的 API 额度。对外公开前，建议放在有访问控制或限流的反向代理后面；只想自己测试时，可以在「客户发来消息」节点里把 Authentication 改成 Basic Auth。
- **跨域默认放开**：「客户发来消息」节点的 *Allowed Origins (CORS)* 默认是 `*`，意味着任何网站都能在页面里嵌入、调用你的客服。如果要嵌入到自己的网站，请改成你自己的域名。
- 每次提问最多取 2000 字，每次回答最多 1500 token，避免单次请求消耗过多额度；但这不能代替限流。
- 如果 n8n 端口对局域网或公网开放，请确认这是你想要的。
- **知识库会原样发给 AI**，不要把密钥、内部信息、个人数据写进知识库。
- 会话 ID 由聊天页面随机生成；长度不足 16 位或格式异常的会话 ID 不会读取历史记录，避免被猜中后看到别人的对话。
- 聊天记录属于访客的个人数据，请按你所在地区的法规告知用户并定期清理。

## 常见问题

**客服说「知识库是空的」**：还没运行「初始化知识库」，或者运行的是聊天入口。见安装第 3 步。

**用命令行或 API 导入后，聊天地址返回 404**
为了不让所有人的聊天地址都一样，仓库里的工作流没有带 `webhookId`。用编辑器的 *Import from File* 导入时，n8n 会自动生成一个；但用 `n8n import:workflow` 命令或 API 导入时不会，这时 n8n 会用中文节点名拼出地址，导致 404。解决办法：导入前在 JSON 里给「客户发来消息」节点加一个 `"webhookId": "<随机 UUID>"`，或者改用编辑器导入。

**为什么不用 n8n 自带的 AI Agent 和 Memory 节点？**
用 HTTP Request 直接调用接口更透明，方便在发送前脱敏；聊天记录存在数据表里，既能查看，又能在 n8n 重启后保留记忆。

**能换成别的模型吗？**
可以。改「组装请求」里的 `MODEL`，以及「DeepSeek 回答」节点里的接口地址和凭据，换成任何兼容 OpenAI `/chat/completions` 格式的服务即可。

## English quick start

1. Import [`workflow/ai-support-bot.json`](workflow/ai-support-bot.json) into n8n (`⋯` → **Import from File**). Requires n8n with **Data tables** (tested on 2.43).
2. In the **DeepSeek 回答** node, select or create a **DeepSeek** credential.
3. Hover the **初始化知识库** (init knowledge base) node and click its ▶ button to seed the example knowledge base. (*Execute workflow* at the bottom runs the chat entry instead.)
4. Click **Publish**, then open the **Chat URL** from the **客户发来消息** node.
5. Edit the config block at the top of the **组装请求** node and replace the knowledge base (Data tables → 客服知识库) with your own content.

The bot answers in Chinese by default; change the system prompt in **组装请求** to switch languages. The chat page is public by default; put it behind access control or rate limiting before exposing it.

## License

[MIT](LICENSE)
