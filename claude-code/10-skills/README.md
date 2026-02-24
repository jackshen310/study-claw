# 模块十：Skills 技能系统

> 学习目标：掌握 Skills 的结构、触发方式与上下文注入机制
>
> 核心价值：CLAUDE.md 给“知识”，Skills 给“可复用操作能力”。

---

## 10.1 Skills vs CLAUDE.md

| 维度 | CLAUDE.md | Skills |
| --- | --- | --- |
| 作用 | 项目知识与规范 | 可复用的流程能力 |
| 加载 | 会话自动加载 | 按需触发 |
| 参数 | 无 | 支持 `$ARGUMENTS` |
| 动态数据 | 不支持 | 支持 `!` 命令注入 |

---

## 10.2 目录结构与文件格式

```
.claude/skills/<skill>/
├── SKILL.md
├── reference.md (可选)
└── scripts/ (可选)
```

**SKILL.md 示例**：

```markdown
---
name: fix-issue
description: 根据 GitHub issue 修复问题
argument-hint: "[issue-number]"
allowed-tools: Read, Grep, Glob, Bash(gh *), Write, Edit
context: fork
agent: general-purpose
---

根据 GitHub issue $ARGUMENTS 修复代码并补充测试。
```

---

## 10.3 触发方式

1. **Slash 命令**：`/fix-issue 123`
2. **Claude 自动触发**：按 description 匹配
3. **Subagent 预加载**：在 agent 文件里配置 `skills`

---

## 10.4 参数与动态上下文

### `$ARGUMENTS`

```
/ migrate-component SearchBar React Vue
→ $ARGUMENTS[0]=SearchBar, [1]=React, [2]=Vue
```

### `!` 命令注入

```markdown
- PR diff: !`gh pr diff`
- 变更文件: !`gh pr diff --name-only`
```

在 Claude 处理前自动执行并注入结果。

---

## 10.5 context: fork（在 Subagent 中运行）

- 适合大输出、重任务、探索型技能
- 主对话只接收摘要结果

---

## 10.6 核心要点总结

| 知识点 | 关键结论 |
| --- | --- |
| Skills 定位 | 可复用 SOP |
| 触发方式 | Slash / 自动 / preload |
| 参数 | `$ARGUMENTS` 支持位置参数 |
| 动态注入 | `!` 命令在执行前插入上下文 |
| fork | 让技能在独立 Subagent 中运行 |

---

## 10.7 思考题

1. 哪些操作必须设置 `disable-model-invocation: true`？
2. Skills 与 Hooks 的边界在哪里？
3. 什么时候应该把流程写成 Skill，而不是放在 CLAUDE.md？

---

## 10.8 延伸阅读

- [Anthropic：Skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Subagents - Preload Skills](https://docs.anthropic.com/en/docs/claude-code/sub-agents#preload-skills-into-subagents)

---

模块十完成。建议为团队建立共享技能库，提升一致性与效率。
