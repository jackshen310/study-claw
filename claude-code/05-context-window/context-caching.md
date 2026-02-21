# Context Caching 技术文档（Claude Code 视角）

> 目标：给出可工程落地的 context caching 机制说明，覆盖分层模型、缓存策略、成本拆解、Prefill/Decode、KV Cache 存储，以及 vLLM 的 PagedAttention 关联。

---

## 1. 问题背景与工程动机

Claude Code 面对三个现实约束：

1. 上下文极大：repo、AGENTS.md / SPEC.md、skills、工具 schema、系统提示词。
2. 多轮频繁交互：plan / analyze / act / reflect，以及 agent 之间的反复通信。
3. 成本与延迟敏感：每轮都全量发送上下文，token 成本和 latency 会爆炸。

结论：Context caching 不是“可选优化”，而是保证系统可用性的必要能力。

---

## 2. 上下文分层模型

Claude Code 的上下文可分成四层，核心目的是区分“稳定信息”和“易变信息”，从而决定缓存策略。

### 2.1 四层划分

- L0：模型固有能力（不传输）
- L1：稳定系统上下文（强缓存）
- L2：半稳定项目上下文（弱缓存）
- L3：运行时上下文（不缓存）

### 2.2 典型内容

L1 常见内容：
- Claude Code system prompt
- agent role 定义
- 工具 schema

L2 常见内容：
- repo tree
- AGENTS.md / SPEC.md
- skills 注入内容

L3 常见内容：
- 当前用户指令
- 最近 N 轮对话
- tool 执行结果

---

## 3. Context Caching 的核心机制

Context caching 不是“缓存整段 prompt 字符串”，而是基于结构化 Context Blocks 的缓存与复用。

### 3.1 Context Block 结构

```ts
type ContextBlock = {
  id: string
  hash: string
  content: string
}
```

### 3.2 Cache ID 与复用

关键点：

1. 首次请求发送完整内容，生成 context_id。
2. 后续请求仅发送 context_id，不重复传输完整内容。
3. 当缓存命中时，系统直接复用已构建前缀。

### 3.3 强缓存与弱缓存

强缓存（Strong Cache）的特征：
- 内容 hash 稳定
- 长时间有效
- miss 通常意味着系统升级或定义变更

弱缓存（Soft Cache）的特征：
- 允许失效
- 支持 chunk 化
- 仅重算变更部分

---

## 4. 三类 token 成本拆解

很多讨论混淆了不同层面的 token 成本，导致对 caching 的误解。正确拆分如下：

### 4.1 传输 token

减少：只发送 context_id，避免反复传输大段上下文。

### 4.2 计费 token

减少或不重复计费：首次构建缓存计费，复用缓存通常不重复计费。

### 4.3 推理 token

不减少：模型推理时仍需“看到”所有 token，Attention 复杂度不降低。

结论：Context caching 主要降低传输成本与计费成本，不降低推理层面的 token 数量。

---

## 5. Prefill / Decode 推理流程

理解 Prefill 与 Decode 是理解 caching 的关键。

### 5.1 Prefill（前缀构建）

Prefill 阶段：
- 一次性处理全部已有上下文 token
- 构建 KV Cache
- 还未生成任何新 token

### 5.2 Decode（生成阶段）

Decode 阶段：
- 逐 token 生成
- 复用已有 KV Cache
- 每步只新增一个 token 的计算

### 5.3 为什么 Prefill 是主要成本

Prefill 的计算量约为 O(n) token × O(layers)，在超长上下文下显著高于 Decode。

结论：Context caching 优化的是 Prefill 构建成本，而不是 Attention 本身。

---

## 6. KV Cache 的本质与存储

### 6.1 KV Cache 是什么

KV Cache 是 Transformer 注意力层的中间态张量，而非文本或 embedding。

张量形状（简化）：

```
K: [num_tokens, num_heads, head_dim]
V: [num_tokens, num_heads, head_dim]
```

### 6.2 KV Cache 默认存储位置

- 默认在 GPU HBM（高带宽显存）
- CPU 内存用于溢出或冷存
- 磁盘几乎不使用（延迟不可接受）

### 6.3 Prefix KV 的复用

Context caching 的实质是：
- 已构建的 KV Cache 作为“前缀编译产物”
- context_id 指向该前缀
- 新请求仅 attach 已有 KV

---

## 7. vLLM 与 PagedAttention

vLLM 的核心创新是 PagedAttention，将 KV Cache 分页管理。

### 7.1 核心思想

- KV Cache 被拆分为固定大小 block
- GPU 放热 block，CPU 放冷 block
- 通过 page table 管理 token 到 block 的映射

### 7.2 工程收益

1. 降低显存连续块需求
2. 支持多请求共享 KV block
3. Prefix caching 变得可行

---

## 8. 工程实践与常见坑

### 8.1 缓存污染规避

必须避免缓存以下内容：

1. tool execution result
2. 用户输入
3. 不确定性输出

### 8.2 Agent Memory 与 Context Cache 的职责区分

- Context Cache：token 级复用
- Agent Memory：语义级总结

### 8.3 常见误解澄清

误解：context_id 让模型“不再加载内容”。

事实：模型仍加载内容，但复用已构建前缀，不重算 Prefill。

### 8.4 关键风险

1. KV Cache OOM
2. 并发导致显存耗尽
3. 过度共享引发隔离问题

---

## 9. 规模化系统中的工程结论

1. Context caching 是长上下文 Agent 系统的基础设施。
2. 节省成本的关键在于 Prefill 复用，而非 Attention 的复杂度下降。
3. KV Cache 管理是性能与稳定性的核心瓶颈。
4. PagedAttention 是在大规模并发下可行的 KV 管理方式之一。

---

## 10. 延伸问题与下一步探索

1. KV Cache 的 eviction 策略与 TTL 如何设计
2. 多 agent 并发如何避免显存打爆
3. Prefix caching 与上下文压缩协同策略
4. 跨 GPU 的 KV 共享与调度可行性

---

## 附：量级估算示例（理解显存压力）

示例假设：
- 70B 模型
- 80 layers
- 64 heads
- head_dim = 128
- fp16

估算结果：
- 单 token KV 约 2.6 MB
- 1K token 约 2.6 GB

结论：长上下文下 KV Cache 是显存瓶颈的决定因素。
