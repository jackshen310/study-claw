# Claude Code 系统学习计划（核心机制拆解）

> 目标：深度理解 Claude Code 作为 AI Coding Agent 的架构设计与核心机制
>
> 学习方式：文档阅读 → 源码逆向 → 原理分析 → 实验验证

---

## 📚 学习大纲

### 模块一：整体架构与启动流程 ✅ [笔记](./01-architecture/README.md)

- [x] 1.1 Claude Code 的定位：CLI 工具 vs Agent 系统
- [x] 1.2 进程启动流程：入口文件 → 初始化链路
- [x] 1.3 核心组件拆解：前端交互层 / Agent 核心层 / 工具执行层
- [x] 1.4 与 Anthropic API 的通信协议（SSE 流式响应）

---

### 模块二：Agent Loop（智能体循环）⭐ 最核心 ✅ [笔记](./02-agent-loop/README.md)

- [x] 2.1 Think → Tool Call → Observe 三步循环原理
- [x] 2.2 何时终止循环：停止条件判断逻辑
- [x] 2.3 多轮对话的 `messages` 数组如何演化
- [x] 2.4 错误恢复机制：工具报错后如何重试
- [x] 2.5 实验：手动构造 API 请求模拟一次 Agent Loop

---

### 模块三：Tool Use 工具调用系统 ⭐ ✅ [笔记](./03-tool-use/README.md)

- [x] 3.1 Anthropic Tool Use API 协议格式解析
- [x] 3.2 内置工具清单：`Read/Write/Edit/Bash/Glob/Grep` 等
- [x] 3.3 工具的定义方式：JSON Schema 描述
- [x] 3.4 工具执行结果的序列化与回传
- [x] 3.5 并行工具调用（Parallel Tool Use）的处理逻辑
- [x] 3.6 实验：自定义一个工具并注入到 Agent

---

### 模块四：Permission 权限与安全模型 ✅ [笔记](./04-permission/README.md)

- [x] 4.1 危险操作识别：哪些操作需要用户确认
- [x] 4.2 `--dangerously-skip-permissions` 参数的风险
- [x] 4.3 沙箱机制：对文件系统操作的限制边界
- [x] 4.4 Bash 执行安全策略：命令白/黑名单
- [x] 4.5 网络请求的权限控制

---

### 模块五：Context Window 上下文管理 ✅ [笔记](./05-context-window/README.md)

- [x] 5.1 Token 计算机制与 context 预算管理
- [x] 5.2 上下文超限时的压缩策略（Summarization）
- [x] 5.3 对话历史的 Truncation 策略
- [x] 5.4 大文件处理：如何避免一次性塞入大量代码
- [x] 5.5 实验：验证 context 超限时的行为

---

### 模块六：CLAUDE.md 配置系统 ✅ [笔记](./06-claude-md/README.md)

- [x] 6.1 CLAUDE.md 的加载时机与优先级
- [x] 6.2 项目级 vs 用户级 vs 目录级配置
- [x] 6.3 如何通过 CLAUDE.md 注入自定义 System Prompt
- [x] 6.4 最佳实践：CLAUDE.md 应该写什么
- [x] 6.5 实验：编写一份有效的 CLAUDE.md

---

### 模块七：Subagent 子智能体机制 ✅ [笔记](./07-subagent/README.md)

- [x] 7.1 Task 拆分原理：何时启动 Subagent
- [x] 7.2 主 Agent 与 Subagent 之间的通信方式
- [x] 7.3 并行 Subagent 的调度与结果聚合
- [x] 7.4 Task.md 任务追踪文件的生命周期
- [x] 7.5 实验：触发一次多 Subagent 协作任务

---

### 模块八：Workflow 工作流系统 ✅ [笔记](./08-workflow/README.md)

- [x] 8.1 全部 14 种 Hook 事件的生命周期
- [x] 8.2 Hook 配置格式（matcher + command）
- [x] 8.3 stdin/stdout/exit code 通信协议（exit 2 = 拦截）
- [x] 8.4 Headless 模式与 GitHub Actions CI/CD 集成

---

### 模块九：实战项目 ✅ [笔记](./09-practice/README.md)

- [x] 9.1 用 Claude Code 完成一个完整功能需求（从零到 PR）
- [x] 9.2 调试一个复杂 Bug（多文件跨模块）
- [x] 9.3 代码重构实战：模块解耦
- [x] 9.4 编写有效的 Prompt 提升 Agent 成功率

---

### 模块十：Skills 技能系统 ✅ [笔记](./10-skills/README.md)

- [x] 10.1 Skills vs CLAUDE.md：定位与分工
- [x] 10.2 SKILL.md 文件格式与目录结构
- [x] 10.3 三种触发方式（Slash / 自动 / Subagent preload）
- [x] 10.4 $ARGUMENTS 传参 与 `!` 动态上下文注入
- [x] 10.5 context: fork + agent 联动 Subagent

---

## 🗂️ 学习资源

| 资源                                                                           | 说明                                    |
| ------------------------------------------------------------------------------ | --------------------------------------- |
| [官方文档](https://docs.anthropic.com/en/docs/claude-code)                     | 核心入口                                |
| [Tool Use 文档](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) | 工具调用协议                            |
| [Anthropic API Docs](https://docs.anthropic.com/en/api)                        | API 参考                                |
| Claude Code npm 包源码                                                         | `~/.nvm/versions/node/.../claude-code/` |
| [OpenHands 源码](https://github.com/All-Hands-AI/OpenHands)                    | 同类开源参考                            |

---

## 📝 学习笔记目录

```
claude-code/
├── README.md              # 本文件（学习大纲）
├── 01-architecture/       # 模块一笔记
├── 02-agent-loop/         # 模块二笔记
├── 03-tool-use/           # 模块三笔记
├── 04-permission/         # 模块四笔记
├── 05-context-window/     # 模块五笔记
├── 06-claude-md/          # 模块六笔记
├── 07-subagent/           # 模块七笔记
├── 08-workflow/           # 模块八笔记
├── 09-practice/           # 实战项目
└── 10-skills/             # 模块十：Skills 技能系统
```

---

> 💡 建议学习顺序：模块二（Agent Loop）→ 模块三（Tool Use）→ 模块五（Context）→ 其余模块
