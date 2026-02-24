# 模块十：Skills 技能系统

> 学习目标：掌握 Claude Code 的 Skills 系统，了解如何将可复用的工作流、知识和工具封装为技能，并在 Subagent 中按需加载
>
> 核心价值：**CLAUDE.md 给 Claude 项目知识，Skills 给 Claude 可复用的操作能力**

---

## 10.1 Skills 是什么？与 CLAUDE.md 的区别

```
CLAUDE.md（模块六）            vs.     Skills（本模块）
────────────────────────────────────────────────────────
项目的"背景知识 + 规范"              可复用的"能力包"
每次会话自动全量加载                  按需触发（命令或自动匹配）
主要是描述性内容                     可以包含可执行脚本
无法传参                            可以接收 $ARGUMENTS
静态内容                            可注入动态上下文 (!`cmd`)
```

> 类比：CLAUDE.md = 员工手册，Skills = 标准操作程序（SOP）

---

## 10.2 Skills 的目录结构与文件格式

### 存放位置

```
作用域       路径                         说明
───────────────────────────────────────────────────────
用户全局   ~/.claude/skills/<name>/       所有项目可用
项目级     .claude/skills/<name>/         当前项目可用（推荐提交 git）
```

### 典型目录结构

```
.claude/skills/
└── my-skill/
    ├── SKILL.md          ← 必须，技能入口和主要说明
    ├── reference.md      ← 可选，详细 API 文档（按需加载）
    ├── examples.md       ← 可选，使用示例（按需加载）
    └── scripts/
        └── helper.py     ← 可选，辅助脚本（执行，不读入 context）
```

> `SKILL.md` 是唯一必须的文件，其余文件在 SKILL.md 中引用，Claude 按需读取。

### SKILL.md 文件格式（YAML frontmatter + Markdown）

```markdown
---
name: fix-issue
description: 根据 GitHub issue 号修复代码问题
argument-hint: "[issue-number]"
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash(gh *), Write, Edit
context: fork
agent: general-purpose
---

根据 GitHub issue $ARGUMENTS 修复代码：

1. 用 `gh issue view $ARGUMENTS` 获取问题描述
2. 分析相关代码文件
3. 实现修复方案
4. 编写对应测试
5. 创建 commit（使用 fix: 前缀）
```

---

## 10.3 所有 frontmatter 字段详解

| 字段                       | 类型   | 默认    | 说明                                                                           |
| -------------------------- | ------ | ------- | ------------------------------------------------------------------------------ |
| `name`                     | string | 必填    | 技能唯一名称，同时是 slash 命令名 `/name`                                      |
| `description`              | string | 必填    | 技能描述，Claude 用此决定何时自动调用                                          |
| `argument-hint`            | string | 可选    | 参数提示，如 `[issue-number]` 或 `[filename] [format]`                         |
| `disable-model-invocation` | bool   | `false` | `true` = 只有用户能调用（禁止 Claude 自动触发）                                |
| `user-invocable`           | bool   | `true`  | `false` = 只有 Claude 能调用（不出现在 slash 命令列表）                        |
| `allowed-tools`            | list   | 继承    | 技能可使用的工具白名单                                                         |
| `context`                  | enum   | `main`  | `fork` = 在 Subagent 中运行；`main` = 在主对话中运行                           |
| `agent`                    | string | 无      | 搭配 `context: fork` 指定 Subagent 类型（Explore/Plan/general-purpose/自定义） |
| `hooks`                    | object | 无      | 技能专属的生命周期 Hook                                                        |

---

## 10.4 触发方式：三种调用模式

### 方式一：用户 Slash 命令（最常用）

```bash
# 直接调用（无参数）：
/deploy

# 带参数调用：
/fix-issue 123
/migrate-component SearchBar React Vue
```

### 方式二：Claude 自动触发

```
Claude 读取所有技能的 description 字段
当用户的请求匹配某技能描述时，Claude 自动调用

例：
  用户说："帮我修复 #456 这个 issue"
  Claude 识别到有 fix-issue 技能 → 自动调用 /fix-issue 456

控制开关：
  disable-model-invocation: true → 禁止 Claude 自动调用（仅用户触发）
  user-invocable: false → 禁止用户触发（仅 Claude 自动）
```

### 方式三：preload 到 Subagent（模块七联动）

```yaml
# .claude/agents/api-developer.md
---
name: api-developer
description: 开发 API 端点
skills:
  - api-conventions      ← 预加载技能知识库
  - error-handling-patterns
---
按照预加载技能中的规范开发 API 端点。
```

---

## 10.5 参数传递：`$ARGUMENTS`

```markdown
# 技能定义

---

name: migrate-component
description: 将组件从一个框架迁移到另一个框架

---

将 $ARGUMENTS[0] 组件从 $ARGUMENTS[1] 迁移到 $ARGUMENTS[2]。

# 调用方式

/migrate-component SearchBar React Vue

# Claude 看到的展开结果：

将 SearchBar 组件从 React 迁移到 Vue。
```

**参数替换语法：**

```
$ARGUMENTS       → 全部参数（拼接为字符串）
$ARGUMENTS[0]    → 第一个参数（等价于 $0）
$ARGUMENTS[1]    → 第二个参数（等价于 $1）
${CLAUDE_SESSION_ID}  → 当前会话 ID（内置变量）
```

---

## 10.6 动态上下文注入：`!` 命令

### 语法

```markdown
---
name: pr-summary
description: 总结 PR 的变更内容
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## PR 上下文（执行时自动获取）

- PR 差异: !`gh pr diff`
- PR 评论: !`gh pr view --comments`
- 变更文件: !`gh pr diff --name-only`

## 你的任务

基于以上信息，总结这个 PR 的核心改动、潜在风险和建议。
```

### 工作原理

```
用户调用 /pr-summary
    │
    ▼
Claude Code 执行所有 !`...` 命令（Claude 还没参与！）
    │ 执行：gh pr diff
    │ 执行：gh pr view --comments
    │ 执行：gh pr diff --name-only
    ▼
用实际输出替换占位符 → 生成完整 prompt
    │
    ▼
Claude 收到已注入实际数据的完整 prompt → 开始分析
```

> 类比：技能是模板，`!` 命令填充模板数据，Claude 处理最终结果

---

## 10.7 Skills 在 Subagent 中运行：`context: fork`

### 作用

```
context: fork 的效果：
  - 技能在独立 Subagent 中执行（全新 context window）
  - 主对话不受技能产生的大量中间输出影响
  - 技能完成后只返回摘要

适合场景：
  ✅ 耗时的研究和分析任务
  ✅ 会读取大量文件的探索任务
  ✅ 有副作用的部署 / 测试任务

不适合场景：
  ❌ 需要和用户频繁交互的任务
  ❌ 结果需要立即保留在主对话中的任务
```

### 指定 agent 类型

```yaml
# 配合 context: fork 使用
agent: Explore         # Haiku，只读，适合代码探索
agent: Plan            # 继承，只读，适合制定计划
agent: general-purpose # 继承，全工具，适合普通任务
agent: my-agent        # 使用 .claude/agents/ 中的自定义 Subagent
```

---

## 10.8 实战：三个实用技能模板

### 技能一：提交规范化（用户触发）

```markdown
# .claude/skills/commit/SKILL.md

---

name: commit
description: 创建符合规范的 git commit
disable-model-invocation: true ← 必须用户手动触发，不让 Claude 自动 commit
argument-hint: "[optional-scope]"

---

生成并执行一个符合 Conventional Commits 规范的 commit：

1. 运行 `git diff --staged` 查看暂存的改动
2. 分析改动类型（feat/fix/docs/refactor/test/chore）
3. 生成简洁的 commit message（50 字符以内标题 + 详细描述）
4. 执行 `git commit -m "..."` 创建提交
5. 展示 commit 结果

如果没有暂存改动，提示用户先 `git add`。
```

### 技能二：PR 代码审查（自动触发）

```markdown
# .claude/skills/code-review/SKILL.md

---

name: code-review
description: 审查代码变更，检查质量、安全和规范问题。在代码修改后自动执行。
context: fork
agent: Explore
allowed-tools: Read, Grep, Glob, Bash(git \*)

---

## 待审查的代码

- 变更文件: !`git diff --name-only HEAD`
- 完整差异: !`git diff HEAD`

## 审查清单

逐一检查：

1. **代码质量**：可读性、命名规范、代码重复
2. **安全性**：输入验证、SQL 注入、XSS 风险
3. **性能**：不必要的循环、N+1 查询
4. **测试**：关键路径是否有测试覆盖

以 Markdown 表格输出问题列表，按严重程度排序（高/中/低）。
```

### 技能三：组件迁移（带参数）

```markdown
# .claude/skills/migrate-component/SKILL.md

---

name: migrate-component
description: 将 UI 组件从一个框架迁移到另一个框架，保留所有功能和测试
argument-hint: "[component-name] [from-framework] [to-framework]"

---

将 $ARGUMENTS[0] 组件从 $ARGUMENTS[1] 迁移到 $ARGUMENTS[2]：

1. 找到 $ARGUMENTS[0] 相关的所有文件（组件、测试、Story）
2. 分析当前的实现方式和 API
3. 了解 $ARGUMENTS[2] 的等价写法
4. 迁移实现，保持对外 API 不变
5. 更新测试，确保测试全部通过
6. 删除旧的 $ARGUMENTS[1] 相关依赖（如果不再需要）
```

---

## 10.9 Skills 与 CLAUDE.md 的联合使用策略

```
放在 CLAUDE.md：
  ✅ 项目技术栈说明
  ✅ 代码规范和命名约定
  ✅ 常用命令参考（npm scripts 等）
  ✅ 禁止行为列表

放在 Skills：
  ✅ 可重复执行的工作流（deploy/commit/review）
  ✅ 需要传参的操作（fix-issue/migrate）
  ✅ 需要动态注入上下文的任务（pr-summary）
  ✅ 应该在独立 Subagent 中运行的重任务

同时使用的最佳模式：
  CLAUDE.md → 说明"用 /commit 命令提交代码"
  commit Skill → 实现具体的提交逻辑
```

---

## 🔑 核心要点总结

| 知识点                       | 关键结论                                             |
| ---------------------------- | ---------------------------------------------------- |
| **技能 vs CLAUDE.md**        | 技能是可执行的 SOP；CLAUDE.md 是项目知识库           |
| **触发方式**                 | Slash 命令 / Claude 自动 / Subagent preload          |
| **disable-model-invocation** | `true` = 只有人能触发（适合有副作用的操作如 deploy） |
| **user-invocable**           | `false` = 只有 Claude 能触发（背景知识技能）         |
| **$ARGUMENTS**               | 传参语法，支持位置参数 $0/$1/$2                      |
| **!`命令`**                  | 技能渲染时注入动态数据（Claude 执行前）              |
| **context: fork**            | 在独立 Subagent 中运行，隔离大量中间输出             |
| **preload 到 Subagent**      | 给专用 Agent 预装领域知识，无需重复说明              |

---

## 📝 思考题

1. `disable-model-invocation: true` 和 `user-invocable: false` 的区别是什么？分别用于什么场景？
2. `/deploy` 技能为什么一定要设置 `disable-model-invocation: true`？
3. `!` 命令在 Claude 处理 prompt 之前执行——这意味着它的输出不计入 Tool Use，有何利弊？
4. Skills 和 CLAUDE.md 的 `@import` 语法有什么本质区别？

---

## 📚 延伸阅读

- [Anthropic：Extend Claude with skills（官方文档）](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Claude Code：Subagents - Preload Skills](https://docs.anthropic.com/en/docs/claude-code/sub-agents#preload-skills-into-subagents)
- [Claude Code：Skill 在 Hooks 中使用](https://docs.anthropic.com/en/docs/claude-code/hooks#hooks-in-skills-and-agents)

---

> ✅ 模块十完成。Skills 是 Claude Code 最强大的可复用扩展机制，建议在业务场景中尽早建立团队公用技能库。
