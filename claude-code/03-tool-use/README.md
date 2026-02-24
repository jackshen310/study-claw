# 模块三：Tool Use 工具调用系统 ⭐

> 学习目标：掌握 Claude Code 工具调用的完整机制——从协议结构、内置工具，到并行调用与扩展能力

---

## 3.1 Tool Use 协议：定义、调用、回传

### 工具定义（JSON Schema）

每个工具由三部分组成：`name`、`description`、`input_schema`。

```json
{
  "tools": [
    {
      "name": "Read",
      "description": "读取文件内容。默认最多读取 2000 行，支持 offset + limit 分页。",
      "input_schema": {
        "type": "object",
        "properties": {
          "path": { "type": "string" },
          "offset": { "type": "integer" },
          "limit": { "type": "integer" }
        },
        "required": ["path"]
      }
    }
  ]
}
```

**description 很关键**：Claude 是否调用工具、如何填写参数，强烈依赖 description 的约束与示例。

### 调用与回传

```
Claude 输出 tool_use → CLI 执行 → 返回 tool_result
```

```json
// tool_use（assistant 消息）
{ "type": "tool_use", "id": "t1", "name": "Read", "input": { "path": "src/app.js" } }

// tool_result（user 消息）
{ "type": "tool_result", "tool_use_id": "t1", "content": "...", "is_error": false }
```

---

## 3.2 内置工具概览（能力地图）

| 类别 | 工具 | 说明 |
| --- | --- | --- |
| 文件 | Read / Write / Edit / MultiEdit | 读取、覆盖写、精确替换、批量替换 |
| 搜索 | Glob / Grep | 文件名匹配 / 内容搜索 |
| 执行 | Bash | 运行 shell 命令 |
| 网络 | WebSearch / WebFetch | 由服务端提供（视配置） |
| 扩展 | MCP Tools | 外部 MCP Server 提供 |

---

## 3.3 关键执行规则（必须记住）

### Read / Write / Edit

- **Read 默认 2000 行**，超出需 `offset + limit` 分页。
- **修改已有文件**：必须先 Read，再 Write（全量覆盖）或 Edit（精确替换）。
- **Edit 的 old_string 必须唯一匹配**，否则报错。
- **MultiEdit**：同文件多处替换，要求全部替换都成功，否则整体失败（原子性）。

### Bash

- **CWD 持久化**：`cd` 后续命令仍在该目录执行。
- **环境变量不持久**：每次 Bash 调用是新进程，需要在同一命令里 export。

### 错误回传

- 失败时 `is_error: true`，Claude 会根据错误选择重试或改变策略。

---

## 3.4 并行工具调用（Parallel Tool Use）

Claude 可以在单次响应中发出多个 `tool_use`：

```javascript
// assistant
content: [
  { type: "tool_use", id: "t1", name: "Read", input: { path: "a.ts" } },
  { type: "tool_use", id: "t2", name: "Read", input: { path: "b.ts" } }
]

// user（tool_result 同一个消息体内返回）
content: [
  { type: "tool_result", tool_use_id: "t1", content: "..." },
  { type: "tool_result", tool_use_id: "t2", content: "..." }
]
```

**原则**：
- 读操作可并行。
- 写操作（Write/Edit/Bash）通常串行，避免副作用冲突。

---

## 3.5 编辑管道要点（高频踩坑）

- **Read 输出带行号**，Edit 的 `old_string` 不能带行号前缀。
- **外部修改冲突**：若文件在 Read 之后被改动，Edit 会要求重新 Read。
- **expected_replacements** 可防止多处误改（MultiEdit 的安全阀）。

---

## 3.6 扩展工具：MCP（推荐方式）

MCP 允许你把外部能力注册为工具：

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "my-tools": {
      "command": "node",
      "args": ["/path/to/my-tool-server.js"]
    }
  }
}
```

> 具体服务器实现方式见 MCP 文档。Hooks 属于工作流扩展，详见模块八。

---

## 3.7 核心要点总结

| 知识点 | 关键结论 |
| --- | --- |
| Tool Use | tool_use → tool_result 构成闭环 |
| Read/Write | 修改文件必须先 Read，Write 为全量覆盖 |
| Edit | old_string 必须唯一匹配，否则失败 |
| Bash | CWD 持久化，环境变量不持久 |
| 并行 | 读操作可并行，写操作按序 |
| MCP | 外部工具统一接入入口 |

---

## 3.8 思考题

1. 为什么 Edit 必须要求 `old_string` 唯一匹配？
2. 如果 Read 输出带行号，你如何避免 Edit 失败？
3. 并行工具调用有哪些收益和风险？

---

## 3.9 延伸阅读

- [Anthropic：Tool Use API](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview)
- [Anthropic：Tool Use Best Practices](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/implement-tool-use)
- [MCP 协议规范](https://modelcontextprotocol.io/docs)

---

模块三完成。下一步：模块四 - Permission 权限与安全模型
