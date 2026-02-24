# 模块五：Context Window 上下文管理

> 学习目标：理解 Claude Code 如何在有限的 context window 内高效管理长对话，掌握压缩、截断和缓存策略
>
> 核心问题：messages 线性增长，200K token 总会超限，Claude Code 如何应对？

---

## 5.1 Token 计算机制与 Context 预算管理

### Context Window 的结构

```
╔══════════════════════════════════════════════════════╗
║           Claude 的 200K Token Context Window        ║
╠══════════════════════════════════════════════════════╣
║  System Prompt                           ~5K tokens  ║
║  (CLAUDE.md + 工具定义 + 指令)                       ║
╠══════════════════════════════════════════════════════╣
║  Messages 历史                      0 ~ 150K tokens  ║
║  (用户输入 + assistant 响应 + tool_result)            ║
╠══════════════════════════════════════════════════════╣
║  当前轮次输入                         ~5K tokens  ║
╠══════════════════════════════════════════════════════╣
║  输出预留 (max_tokens)               ~8K tokens  ║
╚══════════════════════════════════════════════════════╝

总计：200,000 tokens（约 150,000 个英文单词 / 50万汉字）
```

### Token 消耗的主要来源

```
大型代码库任务的 token 分布（典型场景）：

Read 大文件         → 每次 1K~20K tokens
Bash 命令输出       → 每次 100~5K tokens
多轮对话历史        → 累计 50K~150K tokens
工具定义（所有工具）→ 约 3K tokens（固定开销）
CLAUDE.md 内容     → 约 1K~10K tokens（固定开销）

消耗最快的操作：
  ① 读取多个大文件
  ② 运行输出大量日志的命令
  ③ 长时间多步骤任务（messages 不断增长）
```

### Token 使用量监测

```bash
# Claude Code 界面右上角显示 context 使用百分比
# 例如：Context: 45%  → 消耗了 90K/200K tokens

# API 响应中的 usage 字段：
{
  "usage": {
    "input_tokens": 85432,    # 本次请求输入的 token
    "output_tokens": 1024,    # 本次响应输出的 token
    "cache_read_input_tokens": 70000,  # 从缓存读取的 token
    "cache_creation_input_tokens": 0   # 写入缓存的 token
  }
}
```

---

## 5.2 上下文超限时的压缩策略（Compaction/Summarization）

### 自动触发 Compaction

```
Context 使用率达到 ~95% 时：
         │
         ▼
Claude Code 自动触发 Compaction
         │
         ▼
启动一个新的 Claude API 调用，任务是：
  "请总结以下对话历史，保留：
   - 重要的架构决策
   - 未解决的 bug 和问题
   - 已完成的工作
   - 关键的代码片段和文件路径
   - 下一步计划"
         │
         ▼
生成的摘要替换原有的 messages 历史
（大幅缩减 token 使用量，但保留关键信息）
         │
         ▼
继续当前任务，从摘要重新开始
```

### 手动触发 Compaction

```bash
# 在 Claude Code 对话中输入：
/compact

# 带自定义指令的压缩：
/compact 只保留 TODO 列表和未解决的错误

# 清空历史重新开始：
/clear
```

### Compaction 的信息保留策略

```
✅ 优先保留（核心信息）：
  - 用户的原始需求和目标
  - 已做出的重要技术决策
  - 当前未完成的任务列表
  - 修改过的关键文件及其变化摘要
  - 遇到的错误和已尝试的解决方案
  - 重要的文件路径和代码结构

❌ 可以丢弃（冗余信息）：
  - 中间步骤的探索过程
  - 已放弃的方案
  - 重复的工具调用结果
  - 详细的文件内容（路径保留，内容丢弃）
```

### Compaction 后的行为变化

```
压缩前：
  messages = [500条记录，历史150K token]

压缩后：
  messages = [1条摘要，约5K token]
  + 当前用户输入

影响：
  ✅ Context 恢复充裕，可继续长时间工作
  ⚠️  Claude 对早期细节的记忆变为"摘要级别"
  ⚠️  某些细节可能在摘要中丢失
  ❌  无法"回到"压缩前的状态
```

---

## 5.3 对话历史的 Truncation 策略

### 新版行为（Sonnet 3.7+）：报错而非截断

```
旧版行为（隐式截断）：
  超出 200K → 静默丢弃最早的消息 → Claude 失忆，可能行为异常

新版行为（显式报错）：
  超出 200K → API 返回 validation error
  → Claude Code 必须主动处理（触发 Compaction）

这个改变让问题更可见，但需要开发者主动管理 token 预算。
```

### 大文件的分段读取策略

```
❌ 错误做法（token 爆炸）：
  Read("large-file.js")  // 10000 行文件一次性读入
  → 消耗 20K+ tokens，很快撑爆 context

✅ 正确做法（精准读取）：
  Step 1: Grep("functionName", ".")  // 定位到第 3421 行
  Step 2: Read("file.js", offset=3400, limit=100)  // 只读相关片段
  → 只消耗 200 tokens

Claude Code 的内置工具已针对此优化：
  - Glob: 只返回文件路径（不读取内容）
  - Grep: 只返回匹配行（不返回全文）
  - Read: 支持 offset + limit 分页
```

---

## 5.4 Prompt Caching：大幅降低成本与延迟

### Prompt Caching 的工作原理

```
没有缓存时，每次 API 调用：
─────────────────────────────────────────────
请求  → [System Prompt 5K] + [Messages 历史 80K] + [当前输入 1K]
         ↑                    ↑
         每次都要重新处理！耗费大量计算
─────────────────────────────────────────────

有缓存时：
─────────────────────────────────────────────
第一次请求 → 全部处理，标记 cache breakpoint
后续请求   → [从缓存加载 85K] + [新增内容 1K]
              ↑ 几乎瞬间加载，成本极低
─────────────────────────────────────────────
```

### 缓存的收益

| 指标     | 无缓存 | 有缓存                |
| -------- | ------ | --------------------- |
| 延迟     | 基准   | 降低 **85%**          |
| 成本     | 基准   | 降低 **90%**          |
| 写入缓存 | -      | 比普通 input 贵 25%   |
| 读取缓存 | -      | 比普通 input 便宜 90% |

### 缓存的生命周期

```
缓存写入后：
  默认有效期：5 分钟（每次命中后刷新）
  延长选项：1 小时缓存（额外费用）

适合缓存的内容（稳定、重用率高）：
  ✅ System Prompt（很少变化）
  ✅ CLAUDE.md 内容（固定的项目说明）
  ✅ 工具定义列表（固定不变）
  ✅ 大型文档/代码文件的初次读入

不适合缓存的内容（频繁变化）：
  ❌ 当前用户输入
  ❌ 实时工具执行结果
```

### 最小可缓存长度

| 模型            | 最小缓存长度 |
| --------------- | ------------ |
| Claude Opus 4   | 4,096 tokens |
| Claude Sonnet 4 | 1,024 tokens |
| Claude Haiku    | 2,048 tokens |

### Claude Code 如何使用 Prompt Caching

```
Claude Code 自动在以下位置插入 cache breakpoint：

① System Prompt 末尾（最稳定，缓存命中率最高）
   System: "[CLAUDE.md] + [工具定义] + [通用指令]"
                                                  ↑ cache_control: {"type": "ephemeral"}

② 长对话历史的中间（重用较早的历史消息）
   messages: [早期历史 80K] | [新消息 5K]
                              ↑ breakpoint

实际 API 请求示例：
{
  "system": [
    {
      "type": "text",
      "text": "你是 Claude Code...\n[所有工具定义...]",
      "cache_control": {"type": "ephemeral"}  ← 启用缓存
    }
  ],
  "messages": [...]
}
```

---

## 5.5 深入 Prompt Caching：Claude Code 的实际工作方式

### 底层原理：为什么缓存有效（一分钟版本）

```
LLM 推理 = Prefill（处理所有 input）+ Decode（逐 token 生成）

没有缓存：每次 API 调用都重新 Prefill 全部上下文
有了缓存：相同前缀只 Prefill 一次，后续命中直接复用

类比：Prefill ≈ 编译，KV Cache ≈ 编译产物，context_id ≈ 产物的句柄
```

> 关键结论：100K token 的 Prefill 比生成 2K token 还慢——所以省掉重复 Prefill 极其重要

---

### Claude Code 中的自动缓存机制

Claude Code **不需要开发者手动设置** `cache_control`，系统自动管理：

```
每次请求时，Claude Code 自动在以下位置插入 cache breakpoint：

① System Prompt 末尾（最稳定）
   [CLAUDE.md 内容] + [全部工具定义] + [核心指令]
                                                ↑ 自动 cache breakpoint

② 长对话历史的末尾（随对话推进自动移动）
   [早期历史 80K] | [新消息 5K]
                    ↑ 自动 cache breakpoint（每轮更新位置）
```

**自动移动特性**：在多轮对话中，cache breakpoint 会随着新消息的加入**自动前移**，保证最新鲜的"固定前缀"始终被缓存。系统还会向前检测最多 ~20 个 content block，寻找隐式命中机会。

---

### 缓存命中的严格要求

```
✅ 命中条件：前缀内容精确匹配（byte 级别）+ token 数超过最小阈值

最小缓存长度（低于此长度不会建立缓存）：
  Claude Opus 4     → 4,096 tokens
  Claude Sonnet 4   → 1,024 tokens  ← Claude Code 默认模型
  Claude Haiku      → 2,048 tokens

❌ 导致 cache miss 的常见原因：
  - System prompt 中有任何一个字符变化
  - 工具定义列表发生变化（增减工具）
  - CLAUDE.md 内容被修改
  - 使用了不同的模型版本
```

---

### 成本与延迟对比

| 操作                            | 价格                 | 延迟         |
| ------------------------------- | -------------------- | ------------ |
| 普通 input token                | 基准 1x              | 基准         |
| **cache write**（首次建立缓存） | **1.25x**（贵 25%）  | 基准         |
| **cache read**（命中缓存）      | **0.1x**（便宜 90%） | **降低 85%** |

**为什么 90% 成本节省是现实的？**

```
典型 Claude Code 会话（每轮）：
  System prompt + 工具定义  ≈ 5K tokens（稳定，每轮完全命中）
  对话历史                  ≈ 60K tokens（前段命中，尾段不命中）
  新增用户输入              ≈ 1K tokens（不命中）

无缓存成本 = 66K × price
有缓存成本 ≈ 62K × 0.1 × price + 4K × price = 10.2K × price

节省比例 ≈ 85%（实际因场景而异）
```

---

### Prompt Caching 在 Claude Code 实践中的影响

**① CLAUDE.md 越长缓存收益越大**

```
CLAUDE.md 200 行（约 3K tokens）→ 缓存节省有限
CLAUDE.md 含大型代码示例（约 10K tokens）→ 每轮节省显著

启示：把稳定的背景信息放入 CLAUDE.md（不要每次 prompt 里写）
```

**② 工具定义是高价值缓存对象**

```
Claude Code 所有工具定义 ≈ 3K tokens（固定不变）
每次请求都命中缓存 → 免费
若用 MCP 扩展了很多工具（15K tokens）→ 缓存价值更高
```

**③ 长对话比短对话更省钱**

```
第 1 轮：建立缓存（写入费用）
第 2 轮：命中 System Prompt 缓存
第 N 轮：命中 System Prompt + 大部分历史缓存

对话越长，后期每轮的边际成本越低
```

**④ 缓存有效期决定"连续工作"的价值**

```
默认有效期：5 分钟（每次命中自动续期）
扩展选项：1 小时（额外付费）

实践：
  会话中途暂停 > 5 分钟 → 缓存失效，下次请求重新 Prefill
  连续工作中途不超 5 分钟 → 始终命中

结论：Claude Code 长时间连续工作成本最低
```

---

### 验证缓存是否命中

```bash
# 通过 API 响应的 usage 字段确认（Claude Code debug 模式）：
{
  "usage": {
    "input_tokens": 1200,                  # 本轮新增内容
    "cache_read_input_tokens": 65000,      # ← 这个 > 0 就是命中了！
    "cache_creation_input_tokens": 0,      # 0 = 没有新建缓存
    "output_tokens": 800
  }
}

# 在 Claude Code 中间接判断：
# 第 2 次及之后的请求响应速度明显快于第 1 次 → 缓存已生效
```

---

## 5.6 大文件处理：避免一次性读入大量代码

### 智能搜索策略（Claude 的最佳实践）

```javascript
// 处理大型代码库的推荐流程：

// ① 先用 Glob 获取文件列表（无 token 消耗）
Glob("**/*.ts");
// 返回：["src/api.ts", "src/auth.ts", "src/db.ts", ...]

// ② 用 Grep 定位关键代码（只返回匹配行）
Grep("getUserById", "src/");
// 返回：src/auth.ts:Line 142: export async function getUserById...

// ③ 精确 Read 相关片段
Read("src/auth.ts", (offset = 135), (limit = 30));
// 只读取第 135-165 行，精确高效

// ❌ 避免：
Read("src/auth.ts"); // 整个文件可能有 2000 行
```

### 工具 token 消耗对比

| 操作                      | token 消耗     | 适用场景     |
| ------------------------- | -------------- | ------------ |
| `Glob("**/*.ts")`         | ~100 tokens    | 普查文件结构 |
| `Grep("pattern", ".")`    | ~200 tokens    | 定位代码位置 |
| `Read(file, limit=50)`    | ~500 tokens    | 精确读取片段 |
| `Read("big-file.js")`     | ~20,000 tokens | 🚫 避免！    |
| `Bash("cat big-file.js")` | ~20,000 tokens | 🚫 避免！    |

---

## 5.7 实验：验证 Context 超限时的行为

### 实验一：观察自动 Compaction

```bash
# 在 Claude Code 中执行以下操作，触发高 context 使用：
1. 让 Claude 读取多个大文件
2. 执行多轮复杂任务
3. 观察右上角的 Context 使用率
4. 等待达到 ~95% 时，观察自动 Compaction 过程

预期现象：
- Claude 会主动说"我将压缩对话历史以继续工作..."
- 之后 context 使用率大幅下降（如从 95% 降到 5%）
- 任务继续，但基于摘要而非完整历史
```

### 实验二：手动 Compaction 内容对比

```bash
# 执行一个复杂的多步骤任务（如：分析项目架构）
# 记录当前状态，然后执行：
/compact

# 之后询问 Claude：
"你还记得我们刚才讨论的具体细节吗？"

# 观察：
# - 哪些信息被保留了？
# - 哪些细节丢失了？
# - Claude 的行为有什么变化？
```

### 实验三：验证 Prompt Caching 效果

```bash
# 在一个大型项目中启动 Claude Code
# 第一次请求（无缓存）：
claude -p "列出所有 TypeScript 文件的导出函数"
# 记录响应时间（如：8秒）

# 第二次请求（有缓存，相同 system prompt）：
claude -p "现在列出所有测试文件"
# 观察响应时间（可能：1-2秒，大幅加速）

# 通过 API 响应的 usage 字段验证：
# cache_read_input_tokens > 0  → 缓存命中！
```

---

## 5.8 控制流阶段对照（上下文预算入口）

在控制流视角下，上下文管理是每轮循环的“入口阶段”，具有先决性：

1. **触发点**：每次请求构建前先评估 messages 长度与 token 预算。
2. **决策路径**：达到阈值时触发 compaction；否则进入 system prompt 组装。
3. **影响范围**：压缩结果直接决定后续 tool_use 与推理的可见信息范围。
4. **与缓存协同**：Prompt Caching 降低前缀构建成本，使长对话具备工程可持续性。

因此，Context Window 管理不是“后台优化”，而是控制流的第一道门槛。

---

## 🔑 核心要点总结

| 知识点              | 关键结论                                                     |
| ------------------- | ------------------------------------------------------------ |
| **Context 上限**    | 200K tokens，被 System Prompt + Messages + 输出预留共享      |
| **自动 Compaction** | 达到 ~95% 时触发，用摘要替换历史，不可逆                     |
| **手动操作**        | `/compact` 手动压缩，`/clear` 清空历史                       |
| **新版截断行为**    | Sonnet 3.7+ 超限报错而非静默截断                             |
| **Prompt Caching**  | 自动缓存 System Prompt，成本降低 90%，延迟降低 85%           |
| **缓存有效期**      | 默认 5 分钟，每次命中刷新                                    |
| **KV Cache 本质**   | Transformer Attention 中间态张量，存于 GPU 显存              |
| **KV Cache 大小**   | ≈ 2.6MB/token（70B模型），1K token ≈ 2.6GB                   |
| **Prefill 原理**    | 所有 input token 并行处理产生 KV Cache；缓存省掉重复 Prefill |
| **大文件策略**      | Glob → Grep → Read(offset, limit)，禁止直接读大文件          |

---

## 📝 思考题

1. Compaction 之后 Claude "失忆"了部分细节，如何在任务开始前就减少这种信息丢失？
2. 如果需要 Claude 在整个任务中记住某个关键决策，最好的做法是什么？
3. Prompt Caching 对 CLAUDE.md 的启示是什么？CLAUDE.md 应该放在哪里、写什么格式？
4. 为什么 Anthropic 改变了截断策略（从静默截断到报错）？这个设计哲学体现了什么？
5. **（新）** 为什么 100K token 的 Prefill 比生成 2K token 还慢？（提示：并行 vs 串行）
6. **（新）** 如果 GPU 显存只有 40GB，最多能缓存多少个 token 的 KV Cache（70B 模型）？答案是约 15K token，这对 Agent 设计有何启示？

---

## 📚 延伸阅读

- [Anthropic：Context Compaction（官方文档）](https://docs.anthropic.com/en/docs/claude-code/memory#conversations-and-context)
- [Anthropic：Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Claude Code：/compact 命令](https://docs.anthropic.com/en/docs/claude-code/cli-reference)
- [Claude API：Token 计数](https://docs.anthropic.com/en/docs/build-with-claude/token-counting)
- [vLLM：PagedAttention 论文](https://arxiv.org/abs/2309.06180)
- [ChatGPT 对话：Claude Code 上下文缓存底层机制](https://chatgpt.com/share/69990675-b6bc-800f-babc-3ee76b6cc495)

---

> ✅ 模块五完成。下一步：**模块六 - CLAUDE.md 配置系统**
