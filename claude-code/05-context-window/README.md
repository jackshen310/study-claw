# 模块五：Context Window 上下文管理

> 学习目标：理解 Claude Code 如何在有限 context window 内持续完成长任务
>
> 核心问题：messages 线性增长，如何避免超限与成本失控？

---

## 5.1 Context 预算结构

```
System Prompt（CLAUDE.md + 工具定义）
Messages 历史（对话 + tool_result）
当前输入
输出预留（max_tokens）
```

**消耗最大的来源**：
- 大文件 Read
- 大量 Bash 输出
- 长任务对话历史

---

## 5.2 超限处理：Compaction / Summarization

### 自动触发

- Context 使用率接近上限（约 95%）时自动压缩。
- Claude 用摘要替换历史，保留关键决策与未完成事项。

### 手动触发

```bash
/compact
/compact 只保留 TODO 和未解决错误
/clear
```

**代价**：不可逆丢失细节，只保留摘要级记忆。

---

## 5.3 Truncation 策略变化

- 旧行为：静默截断最早消息（隐式失忆）。
- 新行为：超限报错 → 必须显式 compaction。

---

## 5.4 Prompt Caching：降低成本与延迟

### 关键概念

- **cache write**：首次建立缓存，略贵。
- **cache read**：命中缓存，成本显著下降。

### Claude Code 的自动缓存位置

1. System Prompt 末尾（稳定前缀）
2. 长对话历史的前段（重用率高）

### 典型收益

- 延迟显著下降
- 成本大幅降低

---

## 5.5 大文件与长任务的实践策略

**推荐流程（低 token）**：

```
Glob → Grep → Read(offset, limit)
```

**避免**：
- 一次性 Read 整个大文件
- Bash 输出巨大日志

---

## 5.6 核心要点总结

| 知识点 | 关键结论 |
| --- | --- |
| Context 上限 | 由 System + Messages + 输出预留共同占用 |
| Compaction | 95% 左右触发，可手动 `/compact` |
| Truncation | 新版不再静默截断，超限会报错 |
| Prompt Caching | 缓存 System Prompt 与历史前缀 |
| 大文件策略 | Glob → Grep → 分段 Read |

---

## 5.7 思考题

1. 你会如何在任务开始前减少 compaction 后的信息丢失？
2. 什么信息更适合写进 CLAUDE.md 而不是对话里反复说明？
3. 如果必须读取大文件，你会如何设计“渐进式读取”？

---

## 5.8 延伸阅读

- [Anthropic：Context Compaction](https://docs.anthropic.com/en/docs/claude-code/memory#conversations-and-context)
- [Anthropic：Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Claude Code：/compact 命令](https://docs.anthropic.com/en/docs/claude-code/cli-reference)

---

模块五完成。下一步：模块六 - CLAUDE.md 配置系统
