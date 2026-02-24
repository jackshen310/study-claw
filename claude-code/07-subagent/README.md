# 模块七：Subagent 子智能体机制

> 学习目标：理解 Subagent 如何实现任务拆分、并行与上下文隔离，并掌握自定义方式

---

## 7.1 Subagent 的核心价值

```
主 Agent → Task 工具 → Subagent
```

Subagent 解决三类问题：
- **Context 隔离**：子任务不污染主对话。
- **专业化**：不同任务配不同模型/权限。
- **并行化**：多子任务并发执行。

---

## 7.2 内置 Subagent 类型

| 类型 | 默认模型 | 工具 | 用途 |
| --- | --- | --- | --- |
| Explore | Haiku | 只读 | 文件探索、定位线索 |
| Plan | 继承主对话 | 只读 | 规划与分析 |
| General-purpose | 继承主对话 | 全工具 | 复杂任务 |

---

## 7.3 Task 工具：启动 Subagent

```json
{
  "type": "tool_use",
  "name": "Task",
  "input": {
    "description": "运行测试并汇总失败",
    "prompt": "请运行 npm test 并整理失败用例",
    "agent_type": "general-purpose"
  }
}
```

关键字段：
- `description`：给用户看
- `prompt`：子任务初始指令
- `agent_type`：可选，指定子智能体类型

---

## 7.4 自定义 Subagent（文件定义）

### 位置

```
~/.claude/agents/*.md        # 用户全局
.claude/agents/*.md          # 项目级（推荐提交）
```

### 示例

```markdown
---
name: code-reviewer
description: 代码修改后进行审查
tools: Read, Glob, Grep
model: sonnet
permissionMode: acceptEdits
maxTurns: 20
memory: project
---

你是一名代码审查员，请输出问题清单与改进建议。
```

---

## 7.5 前台 vs 后台

- **前台**：主对话等待，权限提示可交互。
- **后台**：并行执行，需预先授予权限，适合耗时任务。

---

## 7.6 并行调度与结果汇总

```javascript
// 一次响应可发起多个 Task 并行
content: [ Task(...), Task(...), Task(...) ]
```

所有 Subagent 完成后，主 Agent 汇总多个 `tool_result`。

---

## 7.7 Task.md 生命周期（任务追踪）

- 记录任务拆解与进度
- 随任务推进更新
- 任务结束后归档

---

## 7.8 核心要点总结

| 知识点 | 关键结论 |
| --- | --- |
| 触发方式 | 主 Agent 用 Task 工具启动 |
| 价值 | 隔离 / 专业化 / 并行 |
| 自定义 | `.claude/agents/*.md` + frontmatter |
| 前台/后台 | 前台可交互，后台需预授权 |

---

## 7.9 思考题

1. 什么时候适合用 Subagent，而不是主 Agent？
2. Subagent 独立 context 的利弊是什么？
3. 后台 Subagent 为什么必须预先授予权限？

---

## 7.10 延伸阅读

- [Anthropic：Subagents](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- [Anthropic：Agent Teams](https://docs.anthropic.com/en/docs/claude-code/agent-teams)

---

模块七完成。下一步：模块八 - Workflow 工作流系统
