# 模块二：Agent Loop（智能体循环）⭐ 最核心

> 学习目标：深度理解 Claude Code 的 Agent Loop 运作机制，掌握“思考→行动→观察”的完整循环
>
> 这是整个 Claude Code 体系中**最核心**的模块，理解它等于理解了 Agent 的灵魂。

---

## 2.1 Think → Tool Call → Observe 三步循环原理

### 什么是 Agent Loop？

Agent Loop 是 Claude Code 的“心跳”。一句话概括：

> **LLM 不断思考 → 决定调用工具 → 观察结果 → 继续思考，直到任务完成。**

### 三步循环图解

```
用户输入：“帮我修复 app.js 中的 bug”
         │
         ▼
╔═════════════════════════════════════════════╗
║                AGENT LOOP                   ║
║                                             ║
║  ┌─────────────────────────────────────┐    ║
║  │  Step 1: THINK（思考）              │    ║
║  │  • 把 messages 历史 + 工具定义      │    ║
║  │    发送给 Claude API                │    ║
║  │  • Claude 分析情况，决定下一步      │    ║
║  └──────────────┬──────────────────────┘    ║
║                 │                           ║
║                 ▼                           ║
║  ┌─────────────────────────────────────┐    ║
║  │  Step 2: ACT（行动）                │    ║
║  │  • Claude 输出 tool_use 请求        │    ║
║  │  • CLI 解析并执行对应工具           │    ║
║  └──────────────┬──────────────────────┘    ║
║                 │                           ║
║                 ▼                           ║
║  ┌─────────────────────────────────────┐    ║
║  │  Step 3: OBSERVE（观察）            │    ║
║  │  • tool_result 追加到 messages      │    ║
║  │  • 返回顶部继续循环                 │    ║
║  └──────────────┬──────────────────────┘    ║
║                 │                           ║
║    ┌────────────▼────────────┐              ║
║    │  stop_reason == end_turn │              ║
║    │  （无工具调用，任务完成）│              ║
║    └─────────────────────────┘              ║
╚═════════════════════════════════════════════╝
         │
         ▼
    输出最终结果给用户
```

### 一次任务的完整执行流程（示例）

```
轮次 1:
  [用户]   “修复 app.js 的 null 指针 bug”
  [Claude] tool_use: Read("app.js")
  [CLI]    执行 Read，返回文件内容

轮次 2:
  [Claude] tool_use: Read("utils.js")
  [CLI]    返回 utils.js 内容

轮次 3:
  [Claude] tool_use: Write("app.js", 修复后的内容)
  [CLI]    执行写入

轮次 4:
  [Claude] tool_use: Bash("npm test")
  [CLI]    返回测试结果

轮次 5:
  [Claude] text: “已修复并通过测试”
  stop_reason: "end_turn" → 退出循环
```

---

## 2.2 何时终止循环：停止条件判断逻辑

每次 API 响应都包含 `stop_reason`，它决定循环是否继续：

| stop_reason     | 含义               | Agent 行为             |
| --------------- | ------------------ | ---------------------- |
| `tool_use`      | 模型要调用工具     | 执行工具 → 继续循环    |
| `end_turn`      | 模型认为任务完成   | **退出循环**，显示结果 |
| `max_tokens`    | 输出达到最大 token | 压缩上下文，处理截断   |
| `stop_sequence` | 遇到停止词         | 退出循环（特殊场景）   |

### 迭代安全阀：最大循环次数

```
默认最大迭代次数：100 次（可配置）
作用：防止陷入无进展的死循环或过高费用。
触发后：强制停止并提示用户。
```

### 用户主动打断（h2A 队列）

```
用户按 Ctrl+C：
  h2A 接收打断信号 → 中止流式响应 → 在安全检查点退出
（不会在工具执行中途强行终止，避免文件损坏）
```

---

## 2.3 多轮对话的 messages 数组如何演化

LLM 无状态，`messages` 数组就是它的“外部记忆”。每轮循环都会增长：

```javascript
// 初始状态
messages = [
  { role: "user", content: "帮我修复 app.js 的 bug" }
]

// 第1轮：Claude 请求 Read 工具
messages = [
  { role: "user", content: "帮我修复 app.js 的 bug" },
  { role: "assistant", content: [
    { type: "text", text: "我来检查一下代码" },
    { type: "tool_use", id: "tu_001", name: "Read", input: { path: "app.js" } }
  ]}
]

// CLI 执行后追加 tool_result（以 user 角色）
messages = [
  { role: "user", content: "帮我修复 app.js 的 bug" },
  { role: "assistant", content: [/* text + tool_use */] },
  { role: "user", content: [
    { type: "tool_result", tool_use_id: "tu_001", content: "// app.js ..." }
  ]}
]
```

> 说明：`tool_use / tool_result` 的格式与约束详见模块三（Tool Use 协议）。

---

## 2.4 错误恢复机制：工具报错后如何重试

### 错误处理流程

```
工具执行失败
    │
    ▼
CLI 将错误信息封装为 tool_result（is_error: true）
    │
    ▼
发送给 Claude，进入下一轮
    │
    ▼
Claude 根据错误选择：重试 / 换策略 / 询问用户
```

### 常见错误场景与 Claude 的应对

| 错误类型             | Claude 的典型应对策略              |
| -------------------- | ---------------------------------- |
| 文件不存在           | 用 Glob 搜索相似文件名，或询问用户 |
| Bash 命令失败        | 分析错误输出，尝试修复命令后重试   |
| 写入权限不足         | 报告权限问题，请用户确认           |
| 网络超时（MCP）      | 重试或告知用户                     |
| 语法错误（测试失败） | 读取错误日志，修复代码，再次测试   |

### 无限循环的识别与防护

```
危险模式：重复调用相同工具却无进展
防护机制：
  ✅ 最大迭代次数限制
  ✅ 用户可随时 Ctrl+C 打断
  ✅ 模型自行检测停滞并改变策略
  ✅ Hooks 可注入额外提示（见模块八）
```

---

## 2.5 实验：手动模拟一次 Agent Loop

用 Node.js 代码体验 Agent Loop 的完整流程（简化版）：

```javascript
import Anthropic from "@anthropic-ai/sdk";
import fs from "fs";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// 模拟工具：只实现 Read
const tools = [
  {
    name: "Read",
    description: "读取文件内容",
    input_schema: {
      type: "object",
      properties: { path: { type: "string", description: "文件路径" } },
      required: ["path"],
    },
  },
];

async function executeTool(name, input) {
  if (name === "Read") {
    try {
      return fs.readFileSync(input.path, "utf-8");
    } catch (e) {
      return `Error: ${e.message}`;
    }
  }
  return "Unknown tool";
}

async function agentLoop(userInput) {
  const messages = [{ role: "user", content: userInput }];
  let iteration = 0;
  const MAX_ITERATIONS = 10;

  while (iteration < MAX_ITERATIONS) {
    iteration++;
    console.log(`\n=== 第 ${iteration} 轮循环 ===`);

    // 1. THINK：调用 API
    const response = await client.messages.create({
      model: "claude-opus-4-5",
      max_tokens: 4096,
      tools,
      messages,
    });

    console.log("stop_reason:", response.stop_reason);
    messages.push({ role: "assistant", content: response.content });

    // 2. 结束条件
    if (response.stop_reason === "end_turn") {
      const text = response.content.find((b) => b.type === "text");
      console.log("\n✅ 任务完成:", text?.text);
      break;
    }

    // 3. ACT + OBSERVE
    if (response.stop_reason === "tool_use") {
      const toolResults = [];
      const toolUses = response.content.filter((b) => b.type === "tool_use");

      for (const toolUse of toolUses) {
        const result = await executeTool(toolUse.name, toolUse.input);
        toolResults.push({
          type: "tool_result",
          tool_use_id: toolUse.id,
          content: result,
        });
      }

      messages.push({ role: "user", content: toolResults });
    }
  }
}

agentLoop("读取 package.json 并告诉我项目的名称和版本号");
```

---

## 2.6 核心要点总结

| 知识点            | 关键结论                                           |
| ----------------- | -------------------------------------------------- |
| 循环本质          | 单线程 while 循环驱动 Agent                        |
| 终止信号          | `stop_reason === "end_turn"` 时退出               |
| 迭代上限          | 默认 100 次，防止死循环                            |
| messages 增长     | 每轮追加 assistant + user(tool_result)，线性增长   |
| 错误处理          | `is_error: true` 让 Claude 自主应对失败            |
| 用户打断          | h2A 队列实现安全中断，不会损坏文件                  |

---

## 2.7 思考题

1. 如果 Claude 连续多轮调用同一个工具但没有进展，你会如何检测并打破死循环？
2. stop_reason 除了 `end_turn` 与 `tool_use` 外，还有哪些常见取值？
3. 如果用户频繁 Ctrl+C 打断，会对上下文与任务推进造成哪些影响？

---

## 2.8 延伸阅读

- [Anthropic：Tool Use Overview](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview)
- [Anthropic：Agentic Loop Best Practices](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/implement-tool-use)
- [Building Effective Agents（Anthropic 官方博客）](https://www.anthropic.com/research/building-effective-agents)

---

模块二完成。下一步：模块三 - Tool Use 工具调用系统
