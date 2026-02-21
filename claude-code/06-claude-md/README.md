# 模块六：CLAUDE.md 配置系统

> 学习目标：掌握 CLAUDE.md 的完整机制——加载路径、优先级、import 语法、.claude/rules/ 模块化规则，以及如何写一份高效的 CLAUDE.md
>
> CLAUDE.md 是 Claude Code 的"项目宪法"，是影响 Agent 行为最直接的方式

---

## 6.1 CLAUDE.md 的加载时机与优先级

### 加载时机

```
$ claude 启动
    │
    ▼
扫描并加载所有 CLAUDE.md 文件（会话开始时一次性读取）
    │
    ▼
合并内容 → 注入 System Prompt（在工具定义之前）
    │
    ▼
整个会话期间保持不变（除非 /memory 命令手动修改）
```

### 所有 CLAUDE.md 加载路径（优先级从高到低）

```
优先级  路径                                    作用域
────────────────────────────────────────────────────────
  1    /etc/claude-code/CLAUDE.md              系统级（企业/组织统一管理）
  2    /Library/Application Support/           系统级（macOS 企业部署）
       ClaudeCode/CLAUDE.md
  3    ~/.claude/CLAUDE.md                     用户全局（跨项目个人偏好）
  4    ~/.claude/rules/*.md                    用户全局规则（模块化）
  5    {project}/CLAUDE.md                     项目级（团队共享）
  6    {project}/.claude/CLAUDE.md             项目级（推荐位置）
  7    {project}/.claude/rules/*.md            项目规则（模块化）
  8    {subdir}/CLAUDE.md                      子目录级（特定模块规则）
  9    CLAUDE.local.md                         个人本地（不提交到 git）
 10    ~/.claude/projects/<project>/memory/    Auto Memory（自动生成）
       MEMORY.md
```

> ⚡ **关键**：所有层级的 CLAUDE.md 都会被加载并合并，**不是覆盖关系**，而是叠加。

---

## 6.2 项目级 vs 用户级 vs 目录级配置

### 三个最重要的层级

```
层级一：用户全局（~/.claude/CLAUDE.md）
────────────────────────────────────────────
适合：个人编码偏好、跨项目通用规则
示例：
  - 我偏好函数式编程风格
  - 总是用 TypeScript 而非 JavaScript
  - 注释用中文

注意：⚠️ 这些规则对你的所有项目生效！

────────────────────────────────────────────
层级二：项目级（./CLAUDE.md 或 ./.claude/CLAUDE.md）
────────────────────────────────────────────
适合：项目架构说明、团队规范、常用命令
应提交到 git，让团队成员共享
示例：
  - 这是一个 Next.js 15 + PostgreSQL 项目
  - 测试用 Vitest，不用 Jest
  - 构建命令：npm run build

────────────────────────────────────────────
层级三：个人本地（./CLAUDE.local.md）
────────────────────────────────────────────
适合：个人对该项目的特殊偏好，不共享给团队
加入 .gitignore，不提交到 git
示例：
  - 我本地的数据库连接是 localhost:5433
  - 调试时不要清除 console.log
```

### 子目录级 CLAUDE.md（精准作用域）

```
项目结构示例：
project/
├── CLAUDE.md                  ← 全项目通用规则
├── frontend/
│   └── CLAUDE.md              ← 前端专用规则（Claude 在 frontend/ 工作时自动加载）
├── backend/
│   └── CLAUDE.md              ← 后端专用规则
└── docs/
    └── CLAUDE.md              ← 文档规则

当 Claude 编辑 frontend/src/Button.tsx 时：
  加载：项目 CLAUDE.md + frontend/CLAUDE.md
  不加载：backend/CLAUDE.md（不相关）
```

---

## 6.3 如何通过 CLAUDE.md 注入自定义 System Prompt

### CLAUDE.md 的内容就是 System Prompt 的一部分

```
实际 System Prompt 构成：
─────────────────────────────────────────────
[Anthropic 内置指令（不可见）]
[工具定义（所有内置工具的 JSON Schema）]
[CLAUDE.md 内容] ← 这就是你能控制的部分
[用户当前输入]
─────────────────────────────────────────────
```

### 实验验证（91.67% 遵守率）

研究表明，CLAUDE.md 中的指令遵守率约为 **91.67%**，远高于对话内联 prompt（约 5%）。

这说明：**在 CLAUDE.md 中写的规则，比在聊天中说的规则有效得多！**

---

## 6.4 CLAUDE.md 的 import 语法

### @文件引用（动态内容导入）

```markdown
# CLAUDE.md 中使用 @ 引用其他文件

See @README for project overview
See @package.json for available npm commands

# Additional Instructions

- git workflow: @docs/git-instructions.md
- API docs: @docs/api-spec.md
```

**引用规则：**

- `@文件路径` → Claude 会读取该文件内容并内联到上下文
- 适合引用经常变化的内容（package.json、README 等）
- 避免引用过大的文件（消耗大量 context token）
- `` `@anthropic-ai/claude-code` ``（反引号包裹）不会被解析为导入

### CLAUDE.local.md 引用个人配置

```markdown
# 在项目 CLAUDE.md 结尾添加：

# Individual Preferences（下方的 local 文件不会被提交）

- @~/.claude/my-project-instructions.md
```

---

## 6.5 最佳实践：CLAUDE.md 应该写什么

### 黄金结构模板

```markdown
# [项目名] - CLAUDE.md

## 项目概述

这是一个 [技术栈] 的 [项目类型]。
主要功能：[简洁描述]

## 技术栈

- 前端：React 18 + TypeScript + Vite
- 后端：Node.js + Express + PostgreSQL
- 测试：Vitest + Playwright
- 包管理：pnpm

## 常用命令

- 开发：`pnpm dev`
- 测试：`pnpm test`
- 构建：`pnpm build`
- 代码检查：`pnpm lint`
- 类型检查：`pnpm typecheck`

## 代码规范

- 缩进：2 空格
- 组件命名：PascalCase
- 工具函数命名：camelCase
- 文件命名：kebab-case.ts
- 注释：英文（公共 API 必须有 JSDoc）

## 架构规范

- API 路由放在 `src/routes/`
- 业务逻辑放在 `src/services/`
- 数据模型放在 `src/models/`
- 禁止在 routes 层直接写 SQL

## 测试规范

- 每个 service 必须有单元测试
- 测试文件和源文件同目录：`foo.ts` → `foo.test.ts`
- 不要用 `any` 类型

## 重要约束

- 不要修改 `src/core/` 下的文件（核心框架，只读）
- 环境变量通过 `src/config/env.ts` 统一访问
- 不要直接使用 `process.env`
```

### 长度指导

```
✅ 推荐：80~120 行
⚠️  可接受：120~150 行
❌ 避免：超过 150 行（研究显示 Claude 可能忽略部分规则）

解决方案：超出时用 .claude/rules/ 模块化拆分！
```

### 写法原则

```
✅ 具体明确："使用 2 空格缩进"
❌ 模糊笼统："正确格式化代码"

✅ 正向表述："FAVOR 函数式组件"
❌ 负向表述："不要用类组件"（负向指令有时反而提示 Claude 做错误的事）

✅ 包含示例："错误处理格式：throw new AppError(code, message)"
❌ 只有规则，没有示例
```

---

## 6.6 .claude/rules/ 模块化规则系统

### 目录结构

```
your-project/
├── CLAUDE.md                   ← 核心摘要（保持简洁）
└── .claude/
    ├── CLAUDE.md               ← 项目详细说明
    └── rules/
        ├── code-style.md       ← 代码风格（所有文件适用）
        ├── testing.md          ← 测试规范（所有文件适用）
        ├── security.md         ← 安全规范（所有文件适用）
        ├── frontend/
        │   ├── react.md        ← React 规范（TSX 文件适用）
        │   └── styles.md       ← 样式规范（CSS/SCSS 适用）
        └── backend/
            ├── api.md          ← API 规范（routes 适用）
            └── database.md     ← 数据库规范（services 适用）
```

### 路径作用域（Path-Specific Rules）

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API 开发规范

- 所有 API 端点必须包含输入验证
- 使用标准错误响应格式：{ code, message, data }
- 必须包含 OpenAPI 注释
- 响应时间超过 100ms 的接口必须添加缓存
```

**工作原理：**

```
Claude 要修改 src/api/users/get.ts
    ↓
检查所有 rules/*.md 的 paths frontmatter
    ↓
paths: ["src/api/**/*.ts"] → 匹配！
    ↓
自动将该规则文件加载进 System Prompt
    ↓
Claude 修改时自动遵循 API 规范

Claude 要修改 src/components/Button.tsx
    ↓
paths: ["src/api/**/*.ts"] → 不匹配
    ↓
该规则不加载（节省 context token）
```

### 全局用户规则

```
~/.claude/rules/
├── preferences.md    ← 个人编码偏好（跨项目）
└── workflows.md      ← 个人工作流偏好
```

---

## 6.7 Auto Memory（自动记忆系统）

### 什么是 Auto Memory？

Claude Code 会在对话中**自动提取重要信息**，存储到项目记忆文件中，下次会话自动加载。

```
存储位置：~/.claude/projects/<project-hash>/memory/MEMORY.md

触发条件：
  - Claude 执行了重要的架构决策
  - 发现了项目特有的模式或规范
  - 用户纠正了 Claude 的误解

内容示例：
  - "该项目使用 Yarn 而非 npm（npm install 会失败）"
  - "数据库表名使用 snake_case，但 TypeScript 类型用 camelCase"
  - "测试环境端口是 3001，不是 3000"
```

### 管理 Auto Memory

```bash
# 在对话中查看/编辑记忆
/memory

# 禁用 Auto Memory
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1
```

---

## 6.8 实验：编写一份有效的 CLAUDE.md

### 快速初始化

```bash
# 在项目目录中，让 Claude 自动生成 CLAUDE.md
claude
> /init

# Claude 会：
# 1. 扫描项目结构（package.json、框架等）
# 2. 自动生成符合项目的 CLAUDE.md
# 3. 包含常用命令、技术栈说明等
```

### 验证 CLAUDE.md 是否生效

```bash
# 启动新会话，询问 Claude：
> "我们项目用什么测试框架？"
# 如果 CLAUDE.md 写了测试框架，Claude 应该正确回答

> "请用我们项目的代码风格写一个工具函数"
# Claude 应该遵循 CLAUDE.md 中的规范
```

---

## 🔑 核心要点总结

| 知识点          | 关键结论                                       |
| --------------- | ---------------------------------------------- |
| **加载时机**    | 会话开始时一次性全部加载，注入 System Prompt   |
| **加载策略**    | 多层级叠加（不是覆盖），所有匹配文件都生效     |
| **遵守率**      | CLAUDE.md 指令约 91.67% 遵守，远高于对话指令   |
| **推荐长度**    | 80~120 行，超过 150 行效果下降                 |
| **import 语法** | `@文件路径` 动态引入其他文件内容               |
| **rules/ 系统** | 路径作用域规则，按文件类型按需加载，节省 token |
| **Auto Memory** | 自动提取并记住项目关键信息，跨会话持久         |
| **写法原则**    | 具体 > 模糊，正向 > 负向，带示例 > 纯规则      |

---

## 📝 思考题

1. 为什么 CLAUDE.md 的遵守率（91.67%）远高于对话内联 prompt（5%）？从 System Prompt 的视角解释。
2. `.claude/rules/` 的路径作用域（paths frontmatter）解决了什么核心问题？和把所有规则放在 CLAUDE.md 相比有什么优劣？
3. CLAUDE.local.md 的设计体现了什么工程理念？（提示：想想 `.env` vs `.env.local`）
4. 如果你要让 Claude 永远记住"这个项目禁止使用 `var`"，最好写在哪里？写法是什么？

---

## 📚 延伸阅读

- [Anthropic：Manage Claude's Memory（官方）](https://docs.anthropic.com/en/docs/claude-code/memory)
- [Claude Code：/init 命令](https://docs.anthropic.com/en/docs/claude-code/cli-reference)
- [System Prompt 自定义](https://docs.anthropic.com/en/docs/claude-code/settings#system-prompt)

---

> ✅ 模块六完成。下一步：**模块七 - Subagent 子智能体机制**
