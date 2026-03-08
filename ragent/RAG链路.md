# RAG链路

![无法获取该图片](https://oss.open8gu.com/ragent-architecture.svg)

## 组装提示词上下文

下面按「原理 → 面试会怎么问 → 怎么答」说清楚「组装提示词上下文」这件事。

---

### 1.1 整体位置（在 RAG 链路里）

```
  检索完成 → RetrievalContext (kbContext, mcpContext, intentChunks)
                    |
                    v
  RAGChatServiceImpl.streamLLMResponse()
    构建 PromptContext → promptBuilder.buildStructuredMessages() → List<ChatMessage>
                    |
                    v
  ChatRequest.builder().messages(messages).build() → LLMService.streamChat()
```

也就是说：**检索结果**先变成 **PromptContext**，再交给 **RAGPromptService** 拼成发给 LLM 的 **messages**（system + evidence + history + user）。

---

### 1.2 两段「上下文」从哪来（检索侧已经格式化好）

**KB 上下文** 和 **MCP 上下文** 不是在 RAGPromptService 里从零拼的，而是在**检索阶段**就按固定结构格式化好了： 

```
  RetrievalEngine 内：
  · formatKbContext(kbIntents, intentChunks, topK)   → ctx.getKbContext()
  · formatMcpContext(responses, mcpIntents)          → ctx.getMcpContext()
```

- **KB**：单意图时 = 「回答规则（promptSnippet）+ 知识库片段（chunk text）」；多意图 = 合并多意图的规则和片段，并做去重、截断。
- **MCP**：按工具分组，每组 = 「意图规则（promptSnippet）+ 动态数据片段（工具返回文本）」；失败的工具会标出错误信息。

所以「组装提示词上下文」里的「上下文」= 已经格式化好的 **kbContext 字符串** 和 **mcpContext 字符串**，再加上**选哪套 system 模板、怎么排消息顺序**。

---

### 1.3 组装流程（字符画）

```
  PromptContext (question, kbContext, mcpContext, kbIntents, mcpIntents, intentChunks)
                    |
                    v
  RAGPromptService.buildStructuredMessages(context, history, question, subQuestions)
                    |
  ┌─────────────────┴─────────────────┐
  │ 1. 选场景、选 System 模板 (plan)      │
  │    hasMcp && !hasKb → MCP_ONLY       │
  │    !hasMcp && hasKb → KB_ONLY        │
  │    hasMcp && hasKb  → MIXED          │
  │    对应模板：answer-chat-mcp /       │
  │    answer-chat-kb / answer-chat-     │
  │    mcp-kb-mixed.st                   │
  └─────────────────┬───────────────────┘
                    v
  ┌─────────────────────────────────────┐
  │ 2. 单/多意图下是否用节点自定义模板     │
  │    KB_ONLY: planPrompt(kbIntents,    │
  │    intentChunks) → 有命中且单意图    │
  │    且节点配了 promptTemplate 则用    │
  │    节点模板，否则用默认 KB 模板       │
  └─────────────────┬───────────────────┘
                    v
  ┌─────────────────────────────────────┐
  │ 3. buildSystemPrompt(context)        │
  │    → 一条 SYSTEM 消息：模板内容      │
  │    （角色、原则、约束、禁止行为等）   │
  └─────────────────┬───────────────────┘
                    v
  ┌─────────────────────────────────────┐
  │ 4. 按顺序拼 messages：               │
  │    [0] SYSTEM: 基模板（上面）        │
  │    [1] SYSTEM: "## 动态数据片段"     │
  │        + mcpContext (若有)           │
  │    [2] USER:   "## 文档内容"         │
  │        + kbContext (若有)            │
  │    [3] ... 会话历史 history          │
  │    [4] USER: 当前问题                 │
  │        (多子问题时：编号 1. 2. 3.…)   │
  └─────────────────────────────────────┘
                    |
                    v
  List<ChatMessage> → 交给 LLM
```

要点：

- **System**：先按场景（仅 KB / 仅 MCP / 混合）选模板；KB 场景还会根据意图命中情况决定用「节点自定义模板」还是默认 KB 模板。
- **证据**：MCP 用一条 **SYSTEM** 消息（"## 动态数据片段" + mcpContext），KB 用一条 **USER** 消息（"## 文档内容" + kbContext），这样和模板里「唯一信息来源」「块级引用」等约束对应。
- **历史**：原样 append 在证据之后、当前问题之前。
- **当前问题**：单问一句就一条 user；多子问题则一条 user，内容为「请基于上述文档内容，回答以下问题：\n1. xxx\n2. xxx…」，降低漏答。

---

### 1.4 消息顺序小结（面试可直接说）

| 顺序 | 角色   | 内容                                                         |
| ---- | ------ | ------------------------------------------------------------ |
| 1    | SYSTEM | 基模板（按 KB_ONLY / MCP_ONLY / MIXED 选，可能用意图节点自定义模板） |
| 2    | SYSTEM | 若有 MCP：`## 动态数据片段` + mcpContext                     |
| 3    | USER   | 若有 KB：`## 文档内容` + kbContext                           |
| 4    | …      | 会话历史 history（摘要 + 最近 N 轮）                         |
| 5    | USER   | 当前问题（多子问题时为编号列表）                             |

---

### 面经

**Q1：你们 RAG 里 prompt 是怎么组装的？检索出来的内容怎么塞给模型？**

- **答**：我们有一套 **RAGPromptService** 专门负责组装。检索完成后会得到 **kbContext** 和 **mcpContext** 两段已经格式化好的字符串（在检索侧用 **ContextFormatter** 按意图、按 chunk 拼好）。组装时先根据**有没有 KB、有没有 MCP** 选三种场景之一：仅 KB、仅 MCP、混合，每种场景对应不同的 **system 模板**（角色、原则、唯一信息来源、禁止行为等）。然后按固定顺序拼 **messages**：第一条是 system 基模板，接着如果有 MCP 就加一条 system 的「动态数据片段」，如果有 KB 就加一条 user 的「文档内容」，再拼**会话历史**，最后一条 user 是**当前问题**；如果是多个子问题，会把这几个问题编号成 1、2、3… 放在最后一条 user 里，减少模型漏答。这样模型看到的结构是：角色与约束 → 证据（MCP/KB）→ 历史 → 当前问句。

**Q2：为什么 KB 用 user 消息、MCP 用 system 消息装证据？**

- **答**：我们设计是：**文档内容** 当作「用户提供的材料」放在 **USER** 里，和模板里「文档中标注为【文档内容】的是唯一信息来源」对应，方便模型区分「用户给的文档」和「系统给的规则」；**MCP 工具返回** 当作「系统侧提供的动态数据」放在 **SYSTEM** 里，和「动态数据片段」的说明一致，也便于在 prompt 里约束「只用这些数据回答」。这样角色分工清晰，也利于后续改模板、加约束。

**Q3：多意图时 system 模板怎么选？会不会每个意图一个模板？**

- **答**：不会每个意图各发一个模板。我们是**先按场景**选一个基模板（仅 KB / 仅 MCP / 混合）。在**仅 KB** 场景下，会再根据意图和检索结果做一次 **planPrompt**：只保留「有命中 chunk」的意图；若是**单意图且该节点配置了 promptTemplate**，就用**该节点的自定义模板**，否则用默认的 KB 模板。多意图时统一用默认 KB 模板，不在 prompt 里再按意图拆多条 system。

**Q4：子问题多了，怎么保证模型每个都答到？**

- **答**：在组装最后一条 user 时，如果 **subQuestions 有多条**，我们不只发一句改写后的问题，而是发一段「请基于上述文档内容，回答以下问题：」再跟 **1. xxx  2. xxx  3. xxx** 的编号列表。这样模型看到的是明确的多问列表，更容易逐点回答；同时 KB 的文档是按子问题分块组织的（检索侧 formatKbContext 里有按子问题/意图的结构），模板里也约束了「按块回答、不跨块引用」，所以子问题和文档块是对应的。

**Q5：检索出来的 chunk 直接拼成一段文字吗？有没有结构？**

- **答**：不是随便拼。在**检索阶段**就用 **ContextFormatter**（我们实现是 DefaultContextFormatter）把 chunk 格式化成固定结构：KB 会带「回答规则」（意图节点上的 promptSnippet）和「知识库片段」（chunk 的 text，可能按意图分组）；MCP 会带「意图规则」和「动态数据片段」。单意图、多意图、无意图的格式略有不同（单意图可以按节点 id 取 chunk，多意图会合并规则和片段并去重）。所以**组装提示词时**拿到的已经是「带标题、带规则、带片段」的 **kbContext / mcpContext 字符串**，prompt 服务只负责选模板和按顺序拼成 system/user 消息，不再改 chunk 内部结构。

---

### 一句话总结

**「我们 RAG 里组装提示词是：检索结果在 RetrievalEngine 里就格式化成 kbContext、mcpContext 两段字符串；RAGPromptService 根据有没有 KB/MCP 选 system 模板（KB 场景还会看单意图是否用节点模板），然后按顺序拼成 [system 基模板, system 动态数据(若有), user 文档内容(若有), 历史, user 当前问题]，多子问题时最后一条 user 是编号问题列表，保证模型按点回答。」**

## 加入会话、反馈、链路追踪

下面按「原理 → 面试会怎么问 → 怎么答」把「写入会话、反馈与追踪」串起来。

---

### 一、项目中「写入会话、反馈与追踪」是怎么做的

#### 1.1 三块各自指什么（字符画）

```
  ┌─────────────────────────────────────────────────────────────────┐
  │ 写入会话：把「用户问题 + 助手回复」落库，供多轮记忆、列表展示、反馈关联   │
  └─────────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────────┐
  │ 反馈：  ① 流式过程中的 SSE 事件（meta/message/finish/done）       │
  │        ② 用户对某条助手消息的点赞/点踩（单独接口 + 表）             │
  └─────────────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────────────┐
  │ 追踪：  一次请求一条 Run，各环节打 @RagTraceNode，落库 Run + Node   │
  └─────────────────────────────────────────────────────────────────┘
```

---

#### 1.2 写入会话：谁在什么时候写、写到哪里

**用户消息：**

- **时机**：进入 RAG 对话时，在**做改写/意图/检索之前**就要把「当前这轮用户问题」写进会话，并拿到历史供后续用。
- **实现**：`RAGChatServiceImpl.streamChat` 里调 `memoryService.loadAndAppend(actualConversationId, userId, ChatMessage.user(question))`。
- **含义**：`ConversationMemoryService.loadAndAppend` = 先 `load(conversationId, userId)` 得到历史，再 `append(conversationId, userId, userMessage)` 把本条 user 落库，最后返回的是**未包含本条**的历史列表（用于改写、prompt）。
- **落库**：`ConversationMemoryStore`（如 MySQL）按会话 + 用户存；会话表可能同时 createOrUpdate（例如更新 lastTime、question 等）。

```
  请求进入 streamChat(question, conversationId, ...)
           |
           v
  memoryService.loadAndAppend(conversationId, userId, ChatMessage.user(question))
           |
           |  load() → 摘要 + 最近 N 轮
           |  append() → 写入 t_conversation_message（user 这条）
           v
  返回 history（不含本条 user），后续改写/检索/生成用
```

**助手消息：**

- **时机 1**：流式**正常结束**时，在 `StreamChatEventHandler.onComplete()` 里一次性写入**完整回复**。
- **时机 2**：用户**中途取消**时，在 `StreamTaskManager` 的取消逻辑里会调 `buildCompletionPayloadOnCancel()`，里面有 `memoryService.append(..., ChatMessage.assistant(content))`，把**已生成的那一段**落库。
- **实现**：都是 `memoryService.append(conversationId, userId, ChatMessage.assistant(content))`，底层仍是 `ConversationMemoryStore.append` → 插入 `t_conversation_message`，并可能触发会话更新、摘要压缩（`summaryService.compressIfNeeded`）。

```
  模型流式输出
       |
  onContent(chunk) → answer.append(chunk) ；只推 SSE，不落库
       |
  onComplete()
       |
       v
  memoryService.append(conversationId, userId, ChatMessage.assistant(answer.toString()))
       |
       v
  插入 t_conversation_message（assistant），返回 messageId
       |
       v
  sender.sendEvent(FINISH, CompletionPayload(messageId, title))
  sender.sendEvent(DONE, "[DONE]")
  summaryService.compressIfNeeded(...)  // 若开启摘要
```

所以：**写入会话 = 入口处 loadAndAppend 写 user，流式结束（或取消）时 append 写 assistant；存储是 MySQL 的会话/消息表，通过 ConversationMemoryStore 抽象。**

---

#### 1.3 反馈：SSE 事件 + 点赞/点踩

**（1）流式过程里的「反馈」：SSE 事件**

- 前端通过 SSE 收事件，用来展示进度、展示内容、知道何时结束、拿到 messageId 和标题。
- 事件类型（`SSEEventType`）：**meta**（会话 id、任务 id）、**message**（增量内容或思考片段）、**finish**（结束，带 messageId、标题）、**done**（收尾标记）、**cancel** / **reject**（取消/拒绝等）。
- 发送位置：
  - **StreamChatEventHandler.initialize()**：`sender.sendEvent(META, MetaPayload(conversationId, taskId))`。
  - **onContent / onThinking**：按块 `sendEvent(MESSAGE, MessageDelta(type, content))`。
  - **onComplete**：`sendEvent(FINISH, CompletionPayload(messageId, title))`，再 `sendEvent(DONE, "[DONE]")`，最后 `sender.complete()`。
- 前端用这些事件做：展示会话/任务 id、打字机效果、思考态、结束态、用 messageId 做反馈或跳转。

```
  前端 SSE 连接建立
       |
  initialize() → META(conversationId, taskId)
       |
  流式输出中 → MESSAGE(type, content) 多次
       |
  onComplete() → FINISH(messageId, title) → DONE("[DONE]") → complete()
```

**（2）用户对某条消息的点赞/点踩**

- **接口**：`POST /conversations/messages/{messageId}/feedback`，Body 里 vote（1=点赞，-1=点踩）、可选 reason/comment。
- **实现**：`MessageFeedbackController` → `MessageFeedbackService.submitFeedback(messageId, request)`。
- **逻辑**：校验当前用户、消息存在且为 **assistant** 消息；若该用户对该 messageId 尚无反馈则 insert，否则 update（vote、reason、comment）；存表 `t_message_feedback`（message_id、user_id、vote、reason、comment 等）。
- **用途**：效果评估、后续排序或模型优化；列表展示时可通过 `getUserVotes` 标出已点赞/点踩。

---

#### 1.4 追踪：RAG Trace 怎么记

- **目的**：一次对话请求一条「Run」，每个关键步骤一个「Node」，方便排查、看耗时、看错误。
- **入口**：在某个带 `conversationId`、`taskId` 的方法上打 **@RagTraceRoot**（若项目里有的话），由 **RagTraceAspect** 拦截。
- **Root**：切面里生成 `traceId`，从方法参数里按注解配置取 `conversationId`、`taskId`，调 **RagTraceRecordService.startRun** 写入一条 Run（trace_id、trace_name、conversation_id、task_id、user_id、status=RUNNING、start_time）；方法正常返回或抛异常后 **finishRun**（status=SUCCESS/ERROR、error_message、end_time、duration_ms）；最后 **RagTraceContext.clear()**。
- **Node**：各环节方法上打 **@RagTraceNode**（name、type），同一线程内通过 **RagTraceContext** 拿当前 traceId、父 node、深度；切面里 **startNode**（trace_id、node_id、parent_node_id、depth、node_type、node_name、class、method、status=RUNNING），执行完后 **finishNode**（status、error、duration）。这样形成一棵「Run → 多 Node」的树，落库到 Run 表 + Node 表。
- **透传**：TraceId（和可选 TaskId）放在 **RagTraceContext**（如 ThreadLocal / TTL），在异步或线程池里需要透传，避免断链。

```
  @RagTraceRoot 方法进入
       |
  startRun(traceId, conversationId, taskId, userId, RUNNING)
  RagTraceContext.setTraceId(traceId)
       |
  proceed() 过程中：
  @RagTraceNode 方法 → startNode(...) → RagTraceContext.pushNode(nodeId) → proceed() → finishNode(...) → popNode()
       |
  正常/异常结束 → finishRun(SUCCESS/ERROR, error, duration)
  RagTraceContext.clear()
```

---

### 二、面试会怎么问 & 怎么答

**Q1：用户问题和模型回复，你们是怎么存、什么时候存的？**

- **答**：我们统一走 **ConversationMemoryService**。**用户问题**在进入 RAG 主流程前就存：调 **loadAndAppend(conversationId, userId, user 消息)**，先 load 历史再 append 这条 user，返回的是「未含本条」的历史给改写和检索用，这样多轮能读到之前轮次。**助手回复**在流式结束时存：在 **StreamChatEventHandler.onComplete()** 里把整段回复 **append** 成一条 assistant 消息，拿到 messageId 后通过 **finish** 事件把 messageId 带给前端。用户中途点「停止」时，取消逻辑里也会把**已生成的那段** append 成一条 assistant，再发一次 finish，这样前端和列表里都能看到「截断后的那条回复」。存储实现是 **ConversationMemoryStore**（我们用的 MySQL），会话、消息表分开，append 时可能顺带更新会话的最近时间等。

**Q2：流式推给前端时，除了内容还推了哪些信息？**

- **答**：我们用 **SSE** 推好几类事件。连接建立后先发 **meta**，带 **conversationId** 和 **taskId**，前端可以拿来展示或后续停请求。流式过程中发 **message** 事件，带类型（普通回复或思考）和增量内容，前端做打字机效果。结束时发 **finish**，带 **messageId** 和会话标题，前端可以存 messageId、展示标题或用来做点赞/点踩；再发 **done** 表示流真正结束。所以「反馈」在这里有两层意思：一是**过程反馈**（meta、message、finish、done），二是用户对某条消息的**点赞/点踩**，那是另一个接口，按 messageId 存到反馈表。

**Q3：点赞/点踩是怎么做的？**

- **答**：我们有一个 **MessageFeedbackController**，接口是 **POST /conversations/messages/{messageId}/feedback**，body 里 **vote**（1 点赞，-1 点踩）和可选的 reason、comment。**MessageFeedbackService** 里会校验当前用户、校验消息存在且角色是 assistant，然后查是否已有该用户对该 messageId 的反馈：没有就 insert，有就 update vote/reason/comment，存 **t_message_feedback**。这样每条助手消息都可以有用户维度的好评/差评，用于效果评估或后续优化。

**Q4：RAG 链路追踪（Trace）是怎么实现的？**

- **答**：我们用 **AOP + 注解** 做 RAG 全链路追踪。一次请求对应一条 **Trace Run**，在入口方法上打 **@RagTraceRoot**，切面里生成 **traceId**，从方法参数里取 **conversationId**、**taskId**，调 **RagTraceRecordService.startRun** 落库；方法执行完或异常时 **finishRun** 更新 status、耗时、错误信息。各环节（改写、意图、检索、LLM 等）在方法上打 **@RagTraceNode**，切面里 **startNode**（node_id、parent、depth、type、name、class、method），执行完 **finishNode**，这样形成 Run 下挂多 Node 的树形结构，都存 DB。**TraceId** 放在 **RagTraceContext**（如 ThreadLocal），在异步或线程池里需要透传，保证同一次请求的节点都挂在同一条 Run 下。管理后台可以按 traceId/conversationId 查某次请求的完整链路和每步耗时，便于排查和优化。

**Q5：如果用户中途取消，会话里会有这条回复吗？**

- **答**：会。取消时我们会把**已经流式生成的那段内容**通过 **memoryService.append** 写成一条 assistant 消息，并构造 **CompletionPayload**（messageId、title）通过 **finish** 事件发给前端，所以会话列表里能看到「半截」的那条回复，前端也能拿到 messageId 做展示或反馈，不会丢。

---

### 三、一句话总结（背这句）

**「写入会话是：入口处 loadAndAppend 写用户问题并拿到历史，流式结束或取消时 append 写助手回复，都走 ConversationMemoryStore 落 MySQL；反馈有两部分，一是 SSE 的 meta/message/finish/done 做过程反馈和把 messageId 带给前端，二是单独接口对某条助手消息点赞/点踩存 t_message_feedback；追踪是 @RagTraceRoot 打 Run、@RagTraceNode 打各环节 Node，AOP 里 start/finish Run 和 Node 落库，TraceId 放 RagTraceContext 并在线程间透传，方便查一次请求的完整链路和耗时。」**

## MCP工具

### MCP 结果如何进 Prompt

- RetrievalContext 里同时有 kbContext 和 mcpContext（以及 intentChunks）。
- RAGPromptService.buildStructuredMessages 里：若 context.getMcpContext() 非空，会加一条 SYSTEM 消息，内容为「## 动态数据片段」+ mcpContext；KB 的文档内容以 USER「## 文档内容」+ kbContext 形式加入。
- DefaultContextFormatter.formatMcpContext：按 mcpIntents 顺序、按 toolId 分组 MCPResponse；对每个工具可带该意图节点的 promptSnippet（意图规则），再拼该工具所有成功响应的 textResult；失败的工具会单独拼一段错误说明。最终是一大段「意图规则 + 动态数据片段」的文本，和 KB 的「回答规则 + 知识库片段」结构类似，只是来源是「工具返回」而不是向量检索。

所以：MCP 工具 = 意图绑定 mcpToolId → 参数提取 → 执行器执行 → 结果格式化成 mcpContext → 和 kbContext 一起进 Prompt。

------

### 面试可怎么说

- MCP 在项目里做什么：当用户问题命中「MCP 类型」的意图节点时，不查知识库，而是根据节点上的 mcpToolId 找到对应工具，从用户问题里抽参数，调工具，把返回的动态数据和知识库检索结果一起塞进 Prompt，让模型结合「文档 + 动态数据」回答。
- 工具从哪来：MCPToolRegistry 存 toolId → MCPToolExecutor。本地工具：实现 MCPToolExecutor 并注册为 Bean，启动时被 DefaultMCPToolRegistry 自动扫进去。远程工具：配置 MCP Server 列表，MCPClientAutoConfiguration 连 Server、拉工具列表、为每个工具创建 RemoteMCPToolExecutor 并 register。
- 一次调用的链路：意图解析得到 MCP 意图 → RetrievalEngine 里对 MCP 意图 buildMcpRequest（用 MCPParameterExtractor 从问题里抽参数）→ executeMcpTools 并行 executor.execute(request) → formatMcpContext 得到 mcpContext 字符串 → 和 kbContext 一起进 RetrievalContext，组装 Prompt 时作为「动态数据片段」给模型。
- 和 KB 的区别：KB 走向量/多路检索，拿的是「文档块」；MCP 走「意图 → 工具 → 返回文本」，拿的是「接口/服务返回的动态数据」。两者在同一个 RAG 请求里可以并存（同一子问题既有 KB 意图也有 MCP 意图时，会同时检索和调工具，再一起拼进 Prompt）。

> “我们设计了一套 MCP（Model Context Protocol）工具注册与调用链路。
>
> 在架构上，我们利用 Spring 的扩展机制，通过 `ConcurrentHashMap` 维护了一个全局的工具注册表。无论是本地写死的 Bean，还是通过 HTTP 接入的远程工具，都会在启动时统一注册。
>
> 在运行时，当用户的意图被判定为需要调用动态数据时，我们会先用 LLM 作为一个参数提取器，从用户的自然语言中精准提取出 API 需要的结构化参数（比如日期、单号）。接着，利用 `CompletableFuture` 线程池并发调用这些内部接口。
>
> 最后，将接口返回的 JSON 数据格式化为纯文本的 `mcpContext`，作为系统的‘动态数据片段’注入到最终的 System Prompt 中。这样就完美实现了静态知识库与企业内部动态数据的混合 RAG 检索。”
