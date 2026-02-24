# 模块一：Claude Code 整体架构与启动流程

> 学习目标：理解 Claude Code 是什么、由哪些层组成、如何启动、如何与 API 通信
>
> 参考资料：官方文档、npm 包分析、社区逆向研究

---

## 1.1 Claude Code 的定位：CLI 外观，Agent 内核

### 表面：一个 CLI 工具

```bash
# 安装方式
npm install -g @anthropic-ai/claude-code

# 启动方式
claude                    # 交互模式
claude -p "fix the bug"  # 非交互（一次性任务）
claude --help
```

从用户视角看，Claude Code 是一个终端命令；从系统本质看，它是一个 **Agent 运行时**：

| 维度     | 普通 CLI 工具      | Claude Code（Agent 系统）            |
| -------- | ------------------ | ------------------------------------ |
| 执行方式 | 用户 → 命令 → 结果 | 用户目标 → Agent 自主规划 → 多步执行 |
| 状态管理 | 无状态             | 维护完整对话历史 + 上下文            |
| 决策者   | 人类               | LLM（Claude 模型）                   |
| 工具调用 | 硬编码             | 动态决策调用哪个工具                 |
| 错误处理 | 报错退出           | 自动重试 / 换策略                    |

---

## 1.2 运行时与发布形态

Claude Code **不是** Node.js 的解释型脚本，而是：

```
技术栈：
  运行时：Bun（高性能 JS 运行时）
  打包方式：编译为单一可执行二进制文件
  安装位置：~/.local/share/claude/versions/<version>
  入口链接：~/.local/bin/claude -> ~/.local/share/claude/versions/2.1.x
```

选择 Bun 编译的目的（核心动机）：

- 启动速度快
- 单一二进制，避免 node_modules
- 源码保护（编译后无法直接读取 JS 源码）

---

## 1.3 进程启动流程

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
[5] 启动 Agent Loop（详见模块二）
```

### 配置加载优先级

```
优先级（高 → 低）：
① 命令行参数（--model, --dangerously-skip-permissions 等）
② 环境变量（ANTHROPIC_API_KEY, CLAUDE_MODEL 等）
③ 项目级配置（.claude/settings.json）
④ 全局配置（~/.claude/settings.json）
⑤ 内置默认值
```

### 常用环境变量

```bash
ANTHROPIC_API_KEY             # 必须：API 密钥
ANTHROPIC_MODEL               # 可选：指定模型
ANTHROPIC_BASE_URL            # 可选：自定义 API 端点（代理场景）
ANTHROPIC_MAX_TOKENS          # 可选：最大输出 token
CLAUDE_BASH_TIMEOUT_MS        # Bash 工具超时（默认 120000ms）
CLAUDE_CODE_MAX_OUTPUT_TOKENS # 最大输出 token 数
```

---

## 1.4 三层架构与组件职责

### 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                    用户交互层 (UI Layer)                  │
│  Terminal UI │ IDE Extension │ Desktop App │ Browser     │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                  Agent 核心层 (Core Layer)                │
│   主循环 (Agent Loop) │ 消息队列 │ Context Manager │ 权限系统 │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                   工具执行层 (Tool Layer)                 │
│  Read │ Write │ Edit │ Bash │ Glob │ Grep │ ... │ MCP     │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│               集成层 (Integration Layer)                  │
│   Anthropic API │ MCP Protocol │ Git │ LSP              │
└─────────────────────────────────────────────────────────┘
```

### 各层职责

**用户交互层**

- Terminal、IDE 插件、桌面端、浏览器只是入口不同
- 核心 Agent 逻辑一致，UI 只负责展示与输入

**Agent 核心层**

| 组件              | 内部代号 | 职责                                   |
| ----------------- | -------- | -------------------------------------- |
| 主循环            | `nO`     | 单线程循环，驱动整个 Agent 运转         |
| 消息队列          | `h2A`    | 异步双缓冲，处理事件与用户打断          |
| Context Manager   | -        | 管理 messages 历史，控制 Token 预算     |
| Permission System | -        | 工具调用审批与沙箱边界控制              |

**工具执行层**

- 接收 LLM 的 `tool_use` 指令
- 在本地文件系统或 Shell 中执行
- 将结果序列化为 `tool_result` 返回

**集成层**

- Anthropic API：SSE 流式通信
- MCP：外部工具接入协议
- Git / LSP：代码与语义能力增强

---

## 1.5 与 Anthropic API 的通信协议

### 通信方式：SSE（Server-Sent Events）

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
     │◄─ data: {"type":"content_block"} │
     │◄─ data: {"type":"content_delta"} │
     │◄─ data: {"type":"tool_use"}      │
     │◄─ data: {"type":"message_stop"}  │
     │                                  │
```

### 单次请求结构（关键字段）

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
    ]}
  ],
  "tools": [
    {"name": "Read", "description": "...", "input_schema": {}},
    {"name": "Write", "description": "...", "input_schema": {}}
  ]
}
```

> 说明：API 响应中的 `stop_reason` 会驱动 Agent Loop 的继续或结束，具体逻辑见模块二。

---

## 1.6 核心要点总结

| 知识点     | 关键结论                                      |
| ---------- | --------------------------------------------- |
| 运行时     | Bun 编译为二进制，非 Node.js 解释执行         |
| 架构       | 交互层 → 核心层 → 工具层 → 集成层             |
| 启动流程   | 初始化 → 连接 API → 进入 Agent Loop           |
| 通信协议   | SSE 流式，每次请求携带完整对话历史            |
| 配置优先级 | CLI 参数 > 环境变量 > 项目配置 > 全局配置     |

---

## 1.7 思考题

1. Claude Code 为什么坚持“单一 Agent Loop”，而不做多线程并发？
2. SSE 与一次性请求/响应相比，对用户体验有什么直接影响？
3. 如果 API 响应变慢，你会从哪些环节排查瓶颈？

---

## 1.8 延伸阅读

- [官方文档：Claude Code Overview](https://docs.anthropic.com/en/docs/claude-code/overview)
- [官方文档：Settings 配置](https://docs.anthropic.com/en/docs/claude-code/settings)
- [Anthropic：Tool Use API](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [ZenML 深度分析：Claude Code Architecture](https://zenml.io)

---

模块一完成。下一步：模块二 - Agent Loop（核心循环机制）
