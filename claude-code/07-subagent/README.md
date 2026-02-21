# 模块七：Subagent 子智能体机制

> 学习目标：理解 Claude Code 如何通过 Subagent 实现任务拆分、并行执行和上下文隔离，掌握自定义 Subagent 的方法
>
> 核心问题：复杂任务如何避免单个 Agent 的 context 耗尽？如何让多个专家 Agent 协作？

---

## 7.1 Subagent 的核心原理：何时启动

### 什么是 Subagent？

```
主 Agent（你的对话）
    │
    │  遇到复杂/独立子任务时
    ▼
  Task 工具 → 启动 Subagent
    │
    ├─── 独立 context window（全新，不继承主 Agent 历史）
    ├─── 独立工具权限（可以受限）
    ├─── 独立 system prompt（专门为子任务设计）
    └─── 完成后将结果返回给主 Agent
```

### Subagent 解决的核心问题

```
问题一：Context 隔离
  大型任务的中间过程（如读取 100 个文件进行测试）
  不应该污染主 Agent 的 context window

问题二：专业化
  读写代码用全能 Agent
  运行测试用只读 + Bash Agent（避免意外修改）
  代码审查用只读 Agent（纯分析）

问题三：并行化
  同时探索 auth 模块 + database 模块 + API 模块
  比串行执行快 N 倍
```

---

## 7.2 内置 Subagent（Built-in Subagents）

Claude Code 默认内置三类 Subagent：

| Subagent            | 模型                | 工具                   | 用途                           |
| ------------------- | ------------------- | ---------------------- | ------------------------------ |
| **Explore**         | Haiku（快速低延迟） | 只读（Read/Grep/Glob） | 文件发现、代码搜索、代码库探索 |
| **Plan**            | 继承主对话          | 只读                   | 为计划阶段做代码库研究         |
| **General-purpose** | 继承主对话          | 全部工具               | 复杂研究、多步骤操作、代码修改 |

```
为什么 Explore 用 Haiku？
→ 文件探索任务不需要推理深度，Haiku 速度快、延迟低
→ 省钱！大量文件读取用 Haiku 比 Opus 便宜 60x
```

---

## 7.3 Task 工具：Subagent 的触发方式

### 主 Agent 如何启动 Subagent

```javascript
// 当主 Agent 的 API 响应中出现 Task tool_use：
{
  "type": "tool_use",
  "name": "Task",
  "input": {
    "description": "运行测试套件并报告失败用例",
    "prompt": "请运行 npm test，收集所有失败用例的错误信息并整理成报告",
    "agent_type": "general-purpose"  // 可选，指定 Subagent 类型
  }
}

// Claude Code 执行：
// 1. 创建新的 Subagent 实例（独立 context）
// 2. 传入 prompt 作为初始任务
// 3. Subagent 独立运行 Agent Loop
// 4. 完成后将摘要结果返回给主 Agent
```

### Task 工具的参数

```json
{
  "description": "简短的任务描述（给用户看，显示在 UI 上）",
  "prompt": "详细的任务指令（Subagent 的初始输入）",
  "agent_type": "可选，指定使用哪种 Subagent（默认由 Claude 自动选择）"
}
```

---

## 7.4 自定义 Subagent：文件定义格式

### Subagent 文件位置

```
作用域           路径                          说明
──────────────────────────────────────────────────────────
用户全局    ~/.claude/agents/*.md         所有项目可用
项目级      .claude/agents/*.md           仅当前项目可用（推荐提交 git）
```

### Subagent 文件格式（YAML frontmatter + Markdown）

```markdown
---
name: code-reviewer
description: >
  专业代码审查员。在代码修改后主动调用。
  分析代码质量、安全漏洞和最佳实践。
tools: Read, Glob, Grep
model: sonnet
permissionMode: acceptEdits
maxTurns: 20
memory: project
---

你是一位资深代码审查员。被调用时，请：

1. 分析最近的代码修改（可用 git diff）
2. 检查以下维度：
   - 代码可读性和命名规范
   - 潜在的安全漏洞
   - 性能问题
   - 测试覆盖率
3. 为每个问题提供：
   - 问题描述
   - 当前代码
   - 改进建议

以 Markdown 格式输出结构化报告。
```

### 所有可配置字段

| 字段              | 说明                                    | 示例                                            |
| ----------------- | --------------------------------------- | ----------------------------------------------- |
| `name`            | Subagent 唯一名称                       | `code-reviewer`                                 |
| `description`     | 触发时机描述（Claude 据此决定是否调用） | `在代码修改后使用`                              |
| `tools`           | 允许的工具列表                          | `Read, Grep, Glob, Bash`                        |
| `disallowedTools` | 明确禁止的工具                          | `Write, Edit`                                   |
| `model`           | 使用的模型                              | `haiku` / `sonnet` / `opus` / `inherit`         |
| `permissionMode`  | 权限模式                                | `default` / `acceptEdits` / `bypassPermissions` |
| `maxTurns`        | 最大迭代轮次                            | `20`                                            |
| `memory`          | 持久记忆作用域                          | `user` / `project` / `local`                    |
| `isolation`       | 隔离模式                                | `worktree`（git worktree 隔离）                 |
| `background`      | 是否后台运行                            | `true` / `false`                                |
| `skills`          | 预加载技能列表                          | `[api-conventions, error-handling]`             |
| `mcpServers`      | 使用的 MCP 服务器                       | `["slack", "github"]`                           |

---

## 7.5 主 Agent 与 Subagent 的通信

### 通信方式

```
主 Agent
    │
    │ Task(prompt: "分析 auth 模块")
    ▼
  Subagent（独立 context window 启动）
    │ 内部执行：Read → Grep → Bash → ...
    │ 完整的独立 Agent Loop 运行
    ▼
  返回结果摘要（文本形式）
    │
    ▼
主 Agent（tool_result 追加到自己的 messages）
```

**关键特性：**

- Subagent **不继承**主 Agent 的 messages 历史
- Subagent 完成后只返回**摘要结果**（不是全部历史）
- 主 Agent 只看到 `tool_result`，不看到 Subagent 的内部过程

### 前台 vs 后台运行

```
前台 Subagent（默认）：
  - 主对话等待，直到 Subagent 完成
  - 权限提示直接透传给用户
  - 适合：需要用户确认的任务

后台 Subagent（background: true）：
  - 主对话继续，Subagent 并发运行
  - 启动前预先申请所有权限
  - 运行中无法询问用户
  - MCP 工具不可用
  - 适合：耗时的独立任务（测试、扫描等）

触发后台运行：
  - 对 Claude 说："在后台运行这个"
  - 或在运行中按 Ctrl+B 切到后台
```

---

## 7.6 并行 Subagent 的调度与结果聚合

### 并行模式（最强大的用法）

```javascript
// 主 Agent 在一次响应中发起多个 Task（并行！）
assistant_message.content = [
  { type: "text", text: "我将并行研究三个模块" },
  {
    type: "tool_use",
    id: "t1",
    name: "Task",
    input: { description: "研究 auth 模块", prompt: "..." },
  },
  {
    type: "tool_use",
    id: "t2",
    name: "Task",
    input: { description: "研究 database 模块", prompt: "..." },
  },
  {
    type: "tool_use",
    id: "t3",
    name: "Task",
    input: { description: "研究 API 模块", prompt: "..." },
  },
];

// 三个 Subagent 并发运行！
// 全部完成后，主 Agent 收到三个 tool_result，汇总得出结论
```

**实际触发方式（用户 prompt）：**

```bash
# 在 Claude Code 中输入：
"请用独立的 Subagent 并行研究 auth、database 和 API 三个模块，
然后综合分析整体架构"
```

### 串行任务链（Chain）

```
使用 code-reviewer Subagent 找出性能问题
    ↓（得到性能问题列表）
使用 optimizer Subagent 修复这些问题
    ↓（得到修复后的代码）
使用 test-runner Subagent 验证修复是否有效
```

---

## 7.7 Task.md 任务追踪文件的生命周期

### task.md 在 Claude Code 自身的使用

Claude Code（本工具）本身也使用类似 `task.md` 的机制：

```
用途：
  - 记录当前任务的进度和子任务列表
  - Agent 在任务中途可以随时更新
  - 帮助 Agent 维持跨多轮循环的任务意识

典型结构：
  # 任务名称
  - [x] 已完成的子任务
  - [/] 进行中的子任务
  - [ ] 待完成的子任务

生命周期：
  创建（任务开始）→ 持续更新（每步完成）→ 最终归档（任务结束）
```

---

## 7.8 实验：触发一次多 Subagent 协作任务

### 实验：并行代码审查

```bash
# 在你的项目中运行 Claude Code，输入：
"请创建三个专门的 Subagent：
1. 一个只读 Subagent 分析项目的安全漏洞
2. 一个只读 Subagent 分析性能瓶颈
3. 一个只读 Subagent 检查代码规范问题
并行运行，最后综合输出一份完整的代码质量报告"
```

### 自定义 Subagent 实战示例

**创建一个测试运行器 Subagent：**

```markdown
# .claude/agents/test-runner.md

---

name: test-runner
description: >
运行测试套件并分析失败用例。
当需要运行测试或验证代码正确性时调用。
tools: Bash, Read
model: haiku
permissionMode: acceptEdits
maxTurns: 15

---

你是测试运行专家。被调用时：

1. 运行 `npm test`（或项目配置的测试命令）
2. 捕获所有测试输出
3. 分析失败用例：
   - 失败原因
   - 相关文件路径
   - 建议修复方向
4. 输出结构化摘要：
   - 总测试数 / 通过数 / 失败数
   - 每个失败用例的简洁描述
   - 最可能的根本原因

不要修改任何代码，只报告测试结果。
```

### 通过 `/agents` 管理 Subagent

```bash
# 在 Claude Code 对话中：
/agents                    # 查看所有可用 Subagent
/agents                    # 进入交互界面，可创建/编辑/删除

# 命令行快速注入：
claude --agents '{"my-agent": {"description": "...", "prompt": "...", "tools": ["Read"]}}'
```

---

## 🔑 核心要点总结

| 知识点               | 关键结论                                                   |
| -------------------- | ---------------------------------------------------------- |
| **核心价值**         | Context 隔离 + 专业化 + 并行化                             |
| **触发方式**         | 主 Agent 使用 Task 工具启动                                |
| **内置类型**         | Explore（Haiku只读）/ Plan（只读）/ General（全能）        |
| **定义文件**         | YAML frontmatter + Markdown，放在 `.claude/agents/`        |
| **通信方式**         | Subagent 完成后返回文字摘要，不暴露内部过程                |
| **前台/后台**        | 前台等待 + 可询问用户；后台并发 + 预申请权限               |
| **并行调度**         | 主 Agent 单次可发出多个 Task → 真正并发执行                |
| **description 字段** | Claude 根据 description 决定是否自动调用，要写清楚触发时机 |

---

## 📝 思考题

1. Subagent 独立 context 设计的优劣：它解决了什么，但又牺牲了什么？
2. 什么情况下应该用 Subagent 而不是直接让主 Agent 完成？
3. 如果 Subagent 失败了（如测试报错），主 Agent 如何感知并应对？
4. `permissionMode: bypassPermissions` 的 Subagent 存在什么安全风险？

---

## 📚 延伸阅读

- [Anthropic：Create Custom Subagents（官方）](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- [Anthropic：Agent Teams](https://docs.anthropic.com/en/docs/claude-code/agent-teams)
- [Claude Code：Git Worktrees 并行会话](https://docs.anthropic.com/en/docs/claude-code/common-workflows#run-parallel-claude-code-sessions-with-git-worktrees)

---

> ✅ 模块七完成。下一步：**模块八 - Workflow 工作流系统**
