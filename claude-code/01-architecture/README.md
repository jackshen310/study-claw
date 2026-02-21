# 模块一：Claude Code 整体架构与启动流程

> 学习目标：理解 Claude Code 是什么、由哪些层组成、如何启动、如何与 API 通信
>
> 参考资料：官方文档、npm 包分析、社区逆向研究

---

## 1.1 Claude Code 的定位：CLI 工具 vs Agent 系统

### 表面：一个 CLI 工具

```bash
# 安装方式
npm install -g @anthropic-ai/claude-code

# 启动方式
claude                    # 交互模式
claude -p "fix the bug"  # 非交互（一次性任务）
claude --help
```

从用户视角看，Claude Code 就是个终端命令。但本质上它是一个 **Agentic System（智能体系统）**。

### 本质：一个 Agent 运行时

| 维度     | 普通 CLI 工具      | Claude Code（Agent 系统）            |
| -------- | ------------------ | ------------------------------------ |
| 执行方式 | 用户 → 命令 → 结果 | 用户目标 → Agent 自主规划 → 多步执行 |
| 状态管理 | 无状态             | 维护完整对话历史 + 上下文            |
| 决策者   | 人类               | LLM（Claude 模型）                   |
| 工具调用 | 硬编码             | 动态决策调用哪个工具                 |
| 错误处理 | 报错退出           | 自动重试 / 换策略                    |

### 运行时技术栈

Claude Code **不是** Node.js 的解释型脚本，而是：

```
技术栈：
  运行时：Bun（高性能 JS 运行时）
  打包方式：编译为单一可执行二进制文件
  安装位置：~/.local/share/claude/versions/<version>
  入口链接：~/.local/bin/claude -> ~/.local/share/claude/versions/2.1.x
```

> 💡 **为什么用 Bun 编译？**
>
> - 启动速度比 Node.js 快 ~3x
> - 打包为单一二进制，无需 node_modules
> - 源码保护（编译后无法直接读取 JS 源码）

---

## 1.2 进程启动流程

### 宏观启动链路

```
用户执行 $ claude
    │
    ▼
[1] Bun 二进制启动
    │
    ▼
[2] 初始化阶段（同步）
    ├── 读取配置文件（~/.claude/settings.json）
    ├── 读取项目配置（.claude/settings.json）
    ├── 读取 CLAUDE.md（项目上下文）
    ├── 初始化权限系统
    └── 注册内置工具（Read/Write/Bash/Glob 等）
    │
    ▼
[3] 连接阶段
    ├── 验证 ANTHROPIC_API_KEY
    ├── 初始化 MCP 服务器连接（如有配置）
    └── 建立 SSE 流式连接通道
    │
    ▼
[4] 进入交互模式 或 直接执行（-p 参数）
    │
    ▼
[5] 启动 Agent Loop（见模块二）
```

### 关键配置加载顺序

```
优先级（高 → 低）：
① 命令行参数（--model, --dangerously-skip-permissions 等）
② 环境变量（ANTHROPIC_API_KEY, CLAUDE_MODEL 等）
③ 项目级配置（.claude/settings.json）
④ 全局配置（~/.claude/settings.json）
⑤ 内置默认值
```

### 重要环境变量

```bash
ANTHROPIC_API_KEY          # 必须：API 密钥
ANTHROPIC_MODEL            # 可选：指定模型（默认 claude-sonnet-4）
ANTHROPIC_BASE_URL         # 可选：自定义 API 端点（代理场景）
ANTHROPIC_MAX_TOKENS       # 可选：最大输出 token
CLAUDE_BASH_TIMEOUT_MS     # Bash 工具超时（默认 120000ms）
CLAUDE_CODE_MAX_OUTPUT_TOKENS # 最大输出 token 数
```

---

## 1.3 核心组件拆解（三层架构）

### 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                    用户交互层 (UI Layer)                  │
│  Terminal UI │ IDE Extension │ Desktop App │ Browser     │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                  Agent 核心层 (Core Layer)                │
│                                                         │
│   ┌─────────────┐    ┌─────────────┐                   │
│   │  nO 主循环   │◄──►│  h2A 消息队列 │                   │
│   │ (Agent Loop)│    │(Async Queue) │                   │
│   └──────┬──────┘    └──────┬──────┘                   │
│          │                  │                           │
│   ┌──────▼──────────────────▼──────┐                   │
│   │         Context Manager         │                   │
│   │  (消息历史 + Token 预算管理)       │                   │
│   └─────────────────────────────────┘                   │
│                                                         │
│   ┌─────────────────────────────────┐                   │
│   │         Permission System        │                   │
│   │  (操作审批 + 沙箱边界)             │                   │
│   └─────────────────────────────────┘                   │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                   工具执行层 (Tool Layer)                  │
│                                                         │
│   Read  │ Write  │ Edit  │ Bash  │ Glob  │ Grep  │ ...  │
│                                                         │
│   MCP Servers（外部工具扩展）                              │
└─────────────────────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│               集成层 (Integration Layer)                  │
│   Anthropic API │ MCP Protocol │ Git │ LSP              │
└─────────────────────────────────────────────────────────┘
```

### 各层职责详解

#### 📺 用户交互层

- **Terminal**：默认交互界面，支持富文本渲染（颜色、进度条）
- **IDE Extension**：VS Code / JetBrains 插件，嵌入 IDE 侧边栏
- **Desktop App**：独立 GUI 应用（macOS/Windows）
- **Browser**：网页版入口

多种入口，但**核心 Agent 逻辑完全一致**，只是 UI 层不同。

#### 🧠 Agent 核心层（最重要）

| 组件                  | 内部代号 | 职责                                   |
| --------------------- | -------- | -------------------------------------- |
| **主循环**            | `nO`     | 单线程 while 循环，驱动整个 Agent 运转 |
| **消息队列**          | `h2A`    | 异步双缓冲队列，处理实时事件与用户打断 |
| **Context Manager**   | -        | 管理 messages 数组，控制 Token 预算    |
| **Permission System** | -        | 拦截危险操作，请求用户审批             |

#### 🔧 工具执行层

- 接收 LLM 的 `tool_use` 指令
- 在本地文件系统/Shell 中**真实执行**
- 将执行结果序列化为 `tool_result` 返回给 Agent Loop

#### 🔌 集成层

- **Anthropic API**：通过 SSE 与 Claude 模型通信
- **MCP（Model Context Protocol）**：标准化外部工具接入协议
- **Git**：代码版本管理集成
- **LSP**：语言服务器协议（代码理解增强）

---

## 1.4 与 Anthropic API 的通信协议

### 通信方式：Server-Sent Events（SSE）

Claude Code 与 API 的通信采用 **SSE 流式传输**，而非一次性请求/响应：

```
Claude Code                        Anthropic API
     │                                  │
     │   POST /v1/messages              │
     │   Content-Type: application/json │
     │   messages: [...]                │
     │   tools: [...]                   │
     │   stream: true                   │
     │ ────────────────────────────────►│
     │                                  │
     │◄─ data: {"type":"message_start"} │
     │◄─ data: {"type":"content_block"} │  流式返回
     │◄─ data: {"type":"content_delta"} │  token by token
     │◄─ data: {"type":"tool_use"}      │
     │◄─ data: {"type":"message_stop"}  │
     │                                  │
```

### 单次 API 请求的结构（关键）

```json
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 8192,
  "system": "<CLAUDE.md 内容> + <系统提示词>",
  "messages": [
    {"role": "user", "content": "帮我修复这个 bug"},
    {"role": "assistant", "content": [
      {"type": "text", "text": "我来看一下代码"},
      {"type": "tool_use", "id": "t1", "name": "Read", "input": {"path": "app.js"}}
    ]},
    {"role": "user", "content": [
      {"type": "tool_result", "tool_use_id": "t1", "content": "...文件内容..."}
    ]},
    ... // 历史对话完整保留
  ],
  "tools": [
    {"name": "Read", "description": "...", "input_schema": {...}},
    {"name": "Write", "description": "...", "input_schema": {...}},
    ... // 所有可用工具的 JSON Schema 定义
  ]
}
```

### 关键设计：LLM 是无状态的

```
⚠️ 重要认知：

Claude 模型本身完全无状态！
它不记得上一次对话说了什么。

Claude Code 的 CLI 负责：
  ✅ 维护完整的 messages 历史数组
  ✅ 每次 API 调用时把全部历史带上
  ✅ 管理 token 预算（历史太长时压缩）

这就是为什么 context window 管理如此关键！
```

### API 响应的 stop_reason 控制循环

```javascript
// 伪代码：Agent Loop 如何根据 stop_reason 决定行为
while (true) {
  const response = await callAPI(messages, tools);

  if (response.stop_reason === "end_turn") {
    // LLM 没有调用工具，任务完成，退出循环
    displayToUser(response.text);
    break;
  }

  if (response.stop_reason === "tool_use") {
    // LLM 请求调用工具，执行工具并继续循环
    const toolResults = await executeTools(response.tool_calls);
    messages.push(assistantMessage(response));
    messages.push(userMessage(toolResults));
    // 继续下一轮循环
  }

  if (response.stop_reason === "max_tokens") {
    // Token 超限，需要压缩上下文
    compressContext(messages);
  }
}
```

---

## 🔑 核心要点总结

| 知识点         | 关键结论                                      |
| -------------- | --------------------------------------------- |
| **运行时**     | Bun 编译为二进制，非 Node.js 解释执行         |
| **架构**       | 三层：交互层 → Agent 核心层 → 工具执行层      |
| **主循环**     | 单线程 while 循环（内部代号 `nO`），简单可靠  |
| **消息队列**   | 异步双缓冲 `h2A`，实现打断/实时控制           |
| **通信协议**   | SSE 流式，每次请求携带完整对话历史            |
| **LLM 无状态** | 状态由 CLI 的 messages 数组维护，不是模型维护 |
| **循环控制**   | API 返回 `stop_reason` 决定继续还是退出       |

---

## 📝 思考题

1. 为什么 Claude Code 选择**单线程** Agent Loop，而不是多线程并发？
2. `h2A` 异步消息队列解决了什么问题？如果没有它会怎样？
3. 每次 API 请求都携带完整历史，context 越来越长，性能如何保证？（→ 模块五）
4. 工具执行层与 MCP 是什么关系？（→ 模块三）

---

## 📚 延伸阅读

- [官方文档：Claude Code Overview](https://docs.anthropic.com/en/docs/claude-code/overview)
- [官方文档：Settings 配置](https://docs.anthropic.com/en/docs/claude-code/settings)
- [Anthropic：Tool Use API](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [ZenML 深度分析：Claude Code Architecture](https://zenml.io)

---

> ✅ 模块一完成。下一步：**模块二 - Agent Loop（核心循环机制）**
