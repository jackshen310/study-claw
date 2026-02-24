# 模块六：CLAUDE.md 配置系统

> 学习目标：掌握 CLAUDE.md 的加载机制、层级优先级、模块化规则与最佳写法
>
> CLAUDE.md 是 Claude Code 的“项目宪法”。

---

## 6.1 加载时机与合并策略

```
会话启动 → 扫描所有 CLAUDE.md → 合并注入 System Prompt
```

**关键点**：
- 不是覆盖关系，而是**叠加合并**。
- 会话开始后保持不变，除非手动 /memory 修改。

---

## 6.2 加载路径与优先级（高 → 低）

```
1. /etc/claude-code/CLAUDE.md
2. /Library/Application Support/ClaudeCode/CLAUDE.md
3. ~/.claude/CLAUDE.md
4. ~/.claude/rules/*.md
5. {project}/CLAUDE.md
6. {project}/.claude/CLAUDE.md
7. {project}/.claude/rules/*.md
8. {subdir}/CLAUDE.md
9. CLAUDE.local.md
10. ~/.claude/projects/<project>/memory/MEMORY.md
```

---

## 6.3 三层配置：用户 / 项目 / 本地

- **用户级**（~/.claude/CLAUDE.md）：跨项目偏好。
- **项目级**（./CLAUDE.md 或 ./.claude/CLAUDE.md）：团队共享规范。
- **本地级**（./CLAUDE.local.md）：个人偏好，不进仓库。

---

## 6.4 `@import` 语法

```markdown
See @README for project overview
See @package.json for npm scripts
```

- 适合引入经常变化的文件（README、package.json）。
- 避免引入超大文件（消耗 context）。

---

## 6.5 .claude/rules/ 模块化规则

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API 规则
- 所有接口必须做输入校验
- 统一返回 { code, message, data }
```

**作用**：按路径加载规则，降低 context 成本，提高命中率。

---

## 6.6 Auto Memory（自动记忆）

- 存储位置：`~/.claude/projects/<project>/memory/MEMORY.md`
- 用于持久化关键项目事实（如测试命令、端口、约定）。
- 可用 `/memory` 查看与编辑。

---

## 6.7 最佳实践（高信噪比规则）

**推荐结构**：
- 项目概述
- 技术栈
- 常用命令
- 代码规范
- 架构约束
- 测试规范
- 禁止行为

**写法原则**：
- 具体 > 模糊
- 正向 > 负向
- 规则 + 示例

---

## 6.8 核心要点总结

| 知识点 | 关键结论 |
| --- | --- |
| 加载方式 | 会话开始一次性加载并合并 |
| 层级规则 | 多层级叠加，不是覆盖 |
| rules/ | 按路径加载，节省 token |
| Auto Memory | 持久化项目关键事实 |

---

## 6.9 思考题

1. 为什么 CLAUDE.md 的指令比对话内提示更有效？
2. 什么时候应该用 `.claude/rules/` 而不是把规则都写在 CLAUDE.md？
3. 如果要强制“不允许 var”，你会写在哪里？

---

## 6.10 延伸阅读

- [Anthropic：Manage Claude's Memory](https://docs.anthropic.com/en/docs/claude-code/memory)
- [Claude Code：/init 命令](https://docs.anthropic.com/en/docs/claude-code/cli-reference)
- [System Prompt 自定义](https://docs.anthropic.com/en/docs/claude-code/settings#system-prompt)

---

模块六完成。下一步：模块七 - Subagent 子智能体机制
