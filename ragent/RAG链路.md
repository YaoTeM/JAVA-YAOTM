# RAG链路

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

