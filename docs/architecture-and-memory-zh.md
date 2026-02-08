# Moltbot 架构与记忆管理

本文档旨在说明 Moltbot 的系统架构以及它是如何管理记忆（Memory）的。

## 1. 系统架构 (Architecture)

Moltbot 的核心是一个基于 WebSocket 的 **Gateway (网关)**，它作为所有消息渠道、代理（Agents）和设备节点（Nodes）的控制平面。

### 1.1 核心组件

*   **Gateway (网关)**
    *   **控制平面**: 它是整个系统的大脑，负责维持与各个客户端的 WebSocket 连接。
    *   **消息路由**: 接收来自 WhatsApp, Telegram, Discord, Slack 等渠道的消息，并将其路由到正确的 Agent 或 Session。
    *   **协议**: 使用基于 JSON 的自定义 WebSocket 协议进行通信（包括请求/响应、事件推送）。
    *   **状态管理**: 管理在线状态 (Presence)、健康检查 (Health) 和定时任务 (Cron)。

*   **Agents (代理)**
    *   **执行逻辑**: Agent 负责处理对话逻辑、调用 LLM（如 Claude, GPT-4）、执行工具（Tools）和管理上下文。
    *   **Session (会话)**: 每个对话（无论是私聊还是群聊）都有一个独立的 Session。Agent 在 Session 中维护上下文历史。
    *   **Capabilities (能力)**: Agent 可以通过 Gateway 调用连接的设备能力（如 macOS 上的 `system.run` 或 iOS 上的 `camera.snap`）。

*   **Channels (渠道)**
    *   **多渠道支持**: 支持 WhatsApp (Baileys), Telegram (grammY), Slack, Discord, Signal, iMessage 等。
    *   **接入方式**: Gateway 直接连接这些服务（如 WhatsApp 协议），或者通过 webhook/bot API 接入。

*   **Nodes (节点)**
    *   **分布式能力**: macOS, iOS, Android 设备可以作为 "Node" 连接到 Gateway。
    *   **本地执行**: 它们暴露本地能力（如屏幕截图、位置信息、通知），Agent 可以远程调用这些能力。

### 1.2 数据流 (Data Flow)

1.  **消息接收**: 用户在 Telegram 发送消息 -> Gateway 接收。
2.  **路由**: Gateway 根据配置找到对应的 Agent Session。
3.  **处理**: Agent 接收消息 -> 构建上下文 (Context) -> 调用 LLM。
4.  **工具调用 (可选)**: LLM 决定调用工具 (如 `memory_search`) -> Agent 执行工具 -> 结果返回 LLM。
5.  **回复**: LLM 生成回复 -> Agent 发送回 Gateway -> Gateway 推送到 Telegram。

---

## 2. 记忆管理 (Memory Management)

Moltbot 的记忆系统设计为 **“文件即记忆” (Files are Memory)**。它不依赖复杂的数据库来存储长期记忆，而是直接读写用户 Workspace 中的 Markdown 文件。这种设计使得记忆对用户可见、可编辑且易于迁移。

### 2.1 记忆层级

Moltbot 的记忆分为三个主要层级：

1.  **上下文窗口 (Context Window) - 短期记忆**
    *   这是 LLM 当前能看到的“工作记忆”。
    *   包含：系统提示词 (System Prompt)、最近的对话历史、工具调用结果。
    *   **限制**: 受限于模型的 Token 上限（如 128k, 200k）。

2.  **文件记忆 (File Memory) - 长期记忆**
    *   存储在 `~/clawd/memory/` (或配置的 workspace) 目录下的 Markdown 文件。
    *   **`MEMORY.md`**: 核心记忆文件。用于存储长期事实、用户偏好、重要决策。Agent 会在主 Session (Main Session) 启动时读取此文件。
    *   **`memory/YYYY-MM-DD.md`**: 每日日志文件。用于记录当天的流水账、临时笔记。Agent 会读取今天和昨天的日志。

3.  **向量索引 (Vector Index) - 检索增强 (RAG)**
    *   为了在海量记忆中找到相关信息，Moltbot 会对 `MEMORY.md` 和 `memory/*.md` 进行向量化索引。
    *   **混合搜索 (Hybrid Search)**: 结合了 **向量搜索 (Semantic Search)** 和 **关键词搜索 (BM25)**，以提高检索准确率。

### 2.2 记忆生命周期与机制

#### A. 写入记忆 (Writing Memory)
*   **显式指令**: 用户可以告诉 Agent “记住这个...”，Agent 会调用 `write_file` 或 `edit_file` 将内容写入 Markdown 文件。
*   **Memory Flush (记忆刷新)**:
    *   当 Session 的上下文快满时（接近 Compaction 阈值），Moltbot 会自动触发一个 **“静默思考” (Silent Thought)** 回合。
    *   系统会提示模型：“Session 即将压缩，请将重要信息写入 `memory/YYYY-MM-DD.md`，如果没有则回复 `NO_REPLY`。”
    *   这确保了在上下文被压缩丢失之前，关键信息被持久化到磁盘。

#### B. 读取记忆 (Reading Memory)
*   **启动加载**: Session 启动时，自动加载 `MEMORY.md` 和最近的日志到 Context。
*   **工具检索**: Agent 拥有 `memory_search` 工具。
    *   当用户问及过去的事实（如“我上次提到的项目代码是多少？”），Agent 会主动调用 `memory_search`。
    *   系统在后台检索向量数据库，返回相关的 Markdown 片段。

#### C. 上下文管理 (Context Management)
为了在有限的 Context Window 中保持高效，Moltbot 实现了以下机制：

1.  **Compaction (压缩)**
    *   **自动触发**: 当 Token 使用量达到模型限制的某个阈值（如 80%）。
    *   **机制**:
        1. 触发 Memory Flush（保存重要信息）。
        2. 将旧的对话历史发送给 LLM 进行摘要 (Summarization)。
        3. 用摘要替换掉旧的历史消息，仅保留最近的 N 条消息。
        4. Session 历史记录在磁盘上（`sessions/*.jsonl`）仍然完整，但在 LLM 的 Context 中被“压缩”了。

2.  **Session Pruning (修剪)**
    *   **动态修剪**: 自动移除旧的、不再需要的工具调用结果（如过长的 `cat` 文件内容或网页抓取结果），只保留工具调用的结论或摘要。
    *   这大大节省了 Token，防止无关的工具输出挤占对话空间。

### 2.3 向量数据库 (Vector Store)

*   **SQLite-vec**: 默认使用 SQLite 的向量扩展 (`sqlite-vec`) 来存储和检索 Embedding。
*   **Embeddings**: 支持本地模型 (通过 node-llama-cpp) 或远程 API (OpenAI, Gemini) 来生成文本向量。
*   **实时更新**: 文件系统监听器 (Watcher) 会监控 `memory/` 目录，一旦文件发生变化，自动更新索引。

## 总结

Moltbot 通过 **Gateway** 连接世界，通过 **Markdown 文件** 存储记忆，并利用 **向量搜索** 和 **自动压缩** 技术，在有限的上下文窗口中实现了持久且智能的记忆管理。
