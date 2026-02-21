# 模块九：实战综合项目

> 本模块是前八个模块的融会贯通，通过四个真实场景，将 Claude Code 的各层机制串联成完整的工程实践
>
> 前提：请确保已阅读前八个模块的笔记

---

## 9.1 实战场景一：从零到 PR（功能开发全流程）

### 场景设定

```
需求：给一个 Express API 项目添加"用户登录限速"功能
要求：连续 5 次失败登录后，封禁 IP 15 分钟
```

### 最优工作流

```bash
# 步骤 1：配置 CLAUDE.md（模块六）
# 确保 Claude 了解项目架构、技术栈和规范

# 步骤 2：使用 Plan 模式分析（模块四：安全）
claude --plan "分析现有的 auth 模块，了解登录接口结构"

# Claude 只读分析，不做任何修改：
# → 阅读 src/routes/auth.ts
# → 阅读 src/middleware/ 目录
# → 输出架构报告

# 步骤 3：制定实现方案
claude "基于刚才的分析，请制定登录限速功能的实现方案，
        包括：存储方案（Redis/内存）、中间件设计、测试策略"

# Claude 输出结构化方案（不执行）

# 步骤 4：正式执行（配合权限设置）
# .claude/settings.json 已配置：
# deny: Bash(rm *), deny: Bash(git push *) 等安全规则

claude "实现登录限速功能，遵循我们的代码规范"

# Agent Loop 过程：
# THINK: 分析方案，规划步骤
# ACT:   创建 src/middleware/rate-limit.ts
# ACT:   修改 src/routes/auth.ts 引入中间件
# ACT:   创建 src/middleware/rate-limit.test.ts
# ACT:   Bash: npm test -- rate-limit
# OBSERVE: 测试全部通过 ✅
# ACT:   git add && git commit
# THINK: 任务完成，stop_reason = end_turn

# 步骤 5：PR 创建
claude "帮我创建一个 PR，描述这次限速功能的改动"
# Hook: PostToolUse → 自动运行 prettier 和 lint（模块八）
```

### 用到的模块知识点

| 步骤           | 涉及模块                                 |
| -------------- | ---------------------------------------- |
| Plan 只读分析  | 模块四（Permission）、模块七（Subagent） |
| CLAUDE.md 配置 | 模块六                                   |
| Agent Loop     | 模块二                                   |
| 工具调用       | 模块三（Read/Write/Edit/Bash）           |
| 权限保护       | 模块四（deny 规则）                      |
| 自动格式化     | 模块八（PostToolUse Hook）               |

---

## 9.2 实战场景二：复杂 Bug 调试（多文件跨模块）

### 场景设定

```
Bug：用户报告"有时候支付成功但订单状态没更新"
线索：偶发，日志不完整，怀疑是竞态条件
```

### 高效调试工作流

```bash
# 策略：充分利用 Grep/Glob 精准定位，避免盲目读大文件（模块五）

claude "帮我调查支付成功但订单未更新的 bug。
        从日志文件和支付相关代码入手"

# Agent 的高效搜索过程（模块三工具调用）：
# 1. Glob("**/payment*") → 定位支付相关文件
# 2. Grep("orderStatus|updateOrder", "src/") → 找状态更新逻辑
# 3. Grep("race|concurrent|Promise.all", "src/services/")
# 4. Read("src/services/payment.ts", offset=45, limit=80) → 精准读取

# Claude 分析后定位到：
# PaymentService.processPayment() 中使用了 Promise.all([
#   updatePaymentRecord(),
#   updateOrderStatus()  ← 这里有 race condition
# ])
# 两个操作没有事务保护

# 触发 Compaction（模块五）：
# 如果调查过程读取了大量文件，context 使用率上升
# Claude 自动压缩保留关键发现
# /compact "保留：bug 位置、根因分析、修复方案"

# 并行 Subagent 验证（模块七）：
claude "请用独立 Subagent 分别：
        1. 查找所有其他地方是否有相同的并发问题
        2. 搜索现有的事务处理工具函数
        3. 检查测试中是否有并发测试用例"
# 三个 Subagent 并行工作，加速定位

# 修复并验证：
claude "实现数据库事务修复这个竞态条件，并添加并发测试"
```

### 关键技术决策

```javascript
// 修复前（有竞态条件）：
await Promise.all([
  db.updatePaymentRecord(paymentId, { status: "paid" }),
  db.updateOrder(orderId, { status: "processing" }),
]);

// 修复后（数据库事务保护）：
await db.transaction(async (trx) => {
  await trx.updatePaymentRecord(paymentId, { status: "paid" });
  await trx.updateOrder(orderId, { status: "processing" });
});
```

---

## 9.3 实战场景三：大规模代码重构

### 场景设定

```
目标：将 1.5 万行的 monolith 按模块拆分，
      同时保证功能不变、测试全通过
```

### Subagent 并行重构策略（模块七）

```bash
# 第一阶段：并行分析（Explore Subagent，只读）
claude "请用多个 Explore Subagent 并行分析：
        - auth 模块的所有依赖关系
        - user 模块的所有依赖关系
        - payment 模块的所有依赖关系
        - 模块间的循环依赖
        汇总后输出依赖关系图"

# 第二阶段：制定拆分计划
claude --plan "基于依赖分析，制定模块拆分顺序（无依赖的先拆）"

# 第三阶段：逐模块重构（利用 Git Worktrees 并行）
# 使用 isolation: worktree 配置，避免修改冲突
```

### CLAUDE.md 的重要性（模块六）

```markdown
# CLAUDE.md - 重构专用配置

## 重构规则（严格遵守）

- 每次只重构一个模块
- 重构后立即运行 npm test
- 保持 public API 签名不变
- 新模块放在 packages/{name}/src/

## 禁止行为

- 不能同时修改多个核心模块
- 不能删除任何现有测试
- 不能修改数据库 schema（单独任务）
```

---

## 9.4 实战场景四：如何写高质量 Prompt

### Prompt 质量对 Agent 成功率的影响

```
低质量 Prompt（Agent 成功率 ~40%）：
  "修复这个 bug"
  "优化这段代码"
  "添加测试"

高质量 Prompt（Agent 成功率 ~90%）：
  详细描述上下文 + 明确目标 + 约束条件 + 验收标准
```

### 高质量 Prompt 模板

```
# 背景
[当前状态的简洁描述]

# 目标
[具体要实现什么]

# 约束
- [不能改变的东西]
- [必须遵守的规范]

# 验收标准
- [ ] [可检验的完成条件 1]
- [ ] [可检验的完成条件 2]

# 参考
[相关文件/示例/文档的路径或链接]
```

### 真实 Prompt 对比

```
❌ 低质量：
"给 API 添加缓存"

✅ 高质量：
"背景：GET /api/products 接口平均响应时间 800ms，数据库压力大。

目标：为这个接口添加 Redis 缓存，缓存有效期 5 分钟。

约束：
- 使用项目现有的 redis 客户端（src/lib/redis.ts）
- 缓存 key 格式：products:list:{query_params_hash}
- cache miss 时按原逻辑查询，不要改变查询逻辑

验收标准：
- [ ] 第一次请求可以正常返回数据
- [ ] 第二次请求 response header 中有 X-Cache: HIT
- [ ] 修改产品后缓存自动失效
- [ ] 添加了测试覆盖 cache hit/miss/invalidation 场景"
```

### Prompt 工程关键技巧

```
1. 指定输出格式
   "请输出 Markdown 格式的分析报告，包含：问题列表、根本原因、修复建议"

2. 设置检查点
   "完成每个文件修改后，先运行 npm run typecheck 确认类型正确"

3. 分阶段执行
   "先只分析，不做任何修改。分析完成后告诉我你的计划，等我确认后再执行"

4. 明确失败条件
   "如果遇到不确定的情况，停下来告诉我，不要猜测"

5. 利用 CLAUDE.md 代替重复说明
   "把项目规范写入 CLAUDE.md，而不是每次 prompt 都重复"
```

---

## 9.5 融会贯通：Claude Code 完整系统图

```
┌─────────────────────────────────────────────────────────────────┐
│                      用户的 Prompt                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SessionStart Hook（模块八）                    │
│              注入：git branch、项目状态、上下文信息                │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│               System Prompt 构建（模块一、六）                    │
│  内置指令 + 工具定义 + CLAUDE.md + .claude/rules/*.md             │
│  （通过 Prompt Caching 高效处理，模块五）                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Agent Loop（模块二）                            │
│                                                                 │
│   THINK → ACT → OBSERVE → THINK → ACT → OBSERVE → ... → DONE   │
│                                                                 │
│   每个 ACT 经过 Permission System（模块四）：                      │
│     allow? → 执行   deny? → 拒绝   ask? → 询问用户               │
│   每个 ACT 经过 PreToolUse Hook（模块八）：                       │
│     合法? → 继续   危险? → exit 2 拦截                           │
│                                                                 │
│   工具执行（模块三）：                                            │
│     Read/Write/Edit/Bash/Glob/Grep/Task/MCP...                  │
│                                                                 │
│   Context 超限时（模块五）：                                      │
│     自动 Compaction → 继续                                       │
│                                                                 │
│   复杂子任务时（模块七）：                                        │
│     Task → Subagent（独立 context）→ 返回摘要                    │
└────────────────────────────┬────────────────────────────────────┘
                             │ PostToolUse Hook
                             │ （模块八：自动格式化/测试/通知）
                             ▼
                         最终结果
```

---

## 9.6 从入门到专家：学习路径回顾

### 第一阶段：理解机制（模块 1-3）

```
模块一：Claude Code 是什么？（架构+启动）
  ↓
模块二：它如何工作？（Agent Loop 核心循环）
  ↓
模块三：它能做什么？（工具体系）
```

### 第二阶段：掌握配置（模块 4-6）

```
模块四：如何控制安全边界？（Permission + Sandbox）
  ↓
模块五：如何处理长任务？（Context 管理）
  ↓
模块六：如何定制行为？（CLAUDE.md 项目宪法）
```

### 第三阶段：高级进阶（模块 7-8）

```
模块七：如何协作扩展？（Subagent 多智能体）
  ↓
模块八：如何自动化集成？（Hooks + CI/CD）
```

### 第九阶段：融会贯通（本模块）

```
真实场景实战 → 高质量 Prompt → 系统性思维
```

---

## 🎯 Claude Code 专家检查清单

### 使用前

- [ ] 写好/更新了 `CLAUDE.md`（技术栈、规范、常用命令）
- [ ] 配置了合适的 `allow/deny` 规则（最小权限原则）
- [ ] 复杂探索先用 `/plan` 模式

### 使用中

- [ ] Prompt 包含：背景 + 目标 + 约束 + 验收标准
- [ ] 避免一次性读取大文件（Glob → Grep → Read(offset)）
- [ ] 需要并行任务时明确说明"用独立 Subagent 并行处理"
- [ ] Context 接近上限考虑 `/compact`

### 使用后

- [ ] 审查 Hook 日志确认自动操作
- [ ] Git 记录完整（每个有意义的步骤一个 commit）
- [ ] 将有价值的经验补充到 `CLAUDE.md`

---

## 📝 三个终极问题

1. **Agent Loop 和传统程序的最大区别是什么？**

   > 传统程序是确定性的：给定输入，输出固定。Agent Loop 是随机的：LLM 每次响应都有随机性，但通过约束（工具定义、权限、CLAUDE.md）降低不确定性。

2. **为什么 context window 管理是 Agent 系统最核心的工程问题？**

   > Agent 的"记忆"完全依赖 context window。超限则失忆、压缩则失精度。如何让 Agent 在有限 token 内维持高质量的任务意识，是所有长任务的根本挑战。

3. **什么是好的 CLAUDE.md？**
   > 好的 CLAUDE.md 让 Claude 从第一秒就像一个在这个项目工作了一年的工程师：知道用什么技术、遵守什么规范、在哪里找什么东西、哪些事情绝对不能做。

---

## 📚 总资料索引

| 模块 | 核心主题       | 配套文档                                                      |
| ---- | -------------- | ------------------------------------------------------------- |
| 一   | 架构与启动     | [01-architecture/README.md](../01-architecture/README.md)     |
| 二   | Agent Loop     | [02-agent-loop/README.md](../02-agent-loop/README.md)         |
| 三   | Tool Use       | [03-tool-use/README.md](../03-tool-use/README.md)             |
| 四   | Permission     | [04-permission/README.md](../04-permission/README.md)         |
| 五   | Context Window | [05-context-window/README.md](../05-context-window/README.md) |
| 六   | CLAUDE.md      | [06-claude-md/README.md](../06-claude-md/README.md)           |
| 七   | Subagent       | [07-subagent/README.md](../07-subagent/README.md)             |
| 八   | Workflow/Hooks | [08-workflow/README.md](../08-workflow/README.md)             |
| 九   | 实战综合       | 本文件                                                        |

---

> 🎉 **恭喜完成 Claude Code 系统学习！**
>
> 你现在已掌握了 Claude Code 从架构设计到日常使用的完整知识体系。
> 下一步建议：在真实项目中实践，并将心得补充回各模块笔记。
