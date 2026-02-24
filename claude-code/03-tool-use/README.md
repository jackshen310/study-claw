# 模块三：Tool Use 工具调用系统 ⭐

> 学习目标：掌握 Claude Code 工具调用的完整机制——从 JSON Schema 定义、内置工具实现，到工具返回值处理和 MCP 扩展

---

## 3.1 Anthropic Tool Use API 协议格式解析

### 工具定义结构（JSON Schema）

每个工具由**三部分**组成，在每次 API 请求中以数组形式传递：

```json
{
  "tools": [
    {
      "name": "Read",
      "description": "读取文件内容。读取前务必确认文件路径正确。默认最多读取 2000 行，大文件可指定 offset 和 limit。",
      "input_schema": {
        "type": "object",
        "properties": {
          "path": {
            "type": "string",
            "description": "要读取的文件绝对路径或相对路径"
          },
          "offset": {
            "type": "integer",
            "description": "起始行号（从 0 开始），用于分页读取"
          },
          "limit": {
            "type": "integer",
            "description": "最大读取行数，默认 2000"
          }
        },
        "required": ["path"]
      }
    }
  ]
}
```

### 为什么 description 至关重要？

```
Claude 决定"用不用这个工具"以及"怎么用"，完全依赖 description！

糟糕的 description：
  "name": "execute"
  "description": "执行代码"
  → Claude 不知道何时用、用什么参数，容易误用

优秀的 description：
  "name": "Bash"
  "description": "在持久化 bash session 中执行 shell 命令。
    工作目录在 cd 后会保持。环境变量不会跨命令持久化。
    危险命令（rm -rf、sudo 等）需要用户确认。
    超时：120 秒。适合：运行测试、安装依赖、查看进程等。"
  → Claude 知道能力边界、限制和适用场景
```

### 完整的工具调用交互流程

```
① 请求阶段（Claude → CLI）
─────────────────────────────
API 响应中包含 tool_use block：
{
  "type": "tool_use",
  "id": "toolu_01XyyZ",       ← 唯一 ID，用于匹配结果
  "name": "Read",
  "input": {
    "path": "src/app.js",
    "offset": 0,
    "limit": 100
  }
}

② 执行阶段（CLI 本地执行）
─────────────────────────────
CLI 读取文件，收集执行结果

③ 返回阶段（CLI → Claude）
─────────────────────────────
以 user 消息形式返回 tool_result：
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01XyyZ",   ← 与请求 ID 对应
      "content": "// src/app.js\nconst express = require('express');\n...",
      "is_error": false                 ← 成功时为 false
    }
  ]
}
```

---

## 3.2 内置工具清单与详解

### 工具全景图

```
Claude Code 内置工具
│
├── 📁 文件操作
│   ├── Read          读取文件内容
│   ├── Write         创建/覆盖文件
│   ├── Edit          单处精确替换
│   └── MultiEdit     同文件多处替换
│
├── 🔍 搜索工具
│   ├── Glob          文件名 pattern 匹配
│   └── Grep          文件内容正则搜索
│
├── 💻 执行工具
│   └── Bash          Shell 命令执行
│
├── 🌐 网络工具（服务端）
│   ├── WebSearch     网络搜索
│   └── WebFetch      抓取网页内容
│
└── 🔌 扩展工具
    └── MCP Tools     外部 MCP 服务器提供
```

---

### 📄 Read — 读取文件

```json
{
  "name": "Read",
  "input_schema": {
    "properties": {
      "path": { "type": "string" },
      "offset": { "type": "integer" },
      "limit": { "type": "integer" }
    },
    "required": ["path"]
  }
}
```

**关键规则：**

- 默认读取上限：**2000 行**
- 大文件需用 `offset + limit` 分页读取
- Write 工具在写入已有文件时，**必须先 Read**（否则报错）
- 支持读取图片（返回 base64）

**典型使用场景：**

```
Claude: "让我先读一下文件再修改"
→ Read("src/app.js")
→ 看完后再决定用 Edit 还是 Write
```

---

### ✏️ Write — 写入文件

```json
{
  "name": "Write",
  "input_schema": {
    "properties": {
      "path": { "type": "string", "description": "文件路径" },
      "content": { "type": "string", "description": "完整文件内容" }
    },
    "required": ["path", "content"]
  }
}
```

**关键规则：**

- 写入**全量内容**（覆盖整个文件），不是追加
- 修改已有文件 → 必须先 Read，再把修改后的完整内容 Write
- 创建新文件 → 直接 Write 即可
- 写入前会触发 **Permission 审批**（见模块四）

---

### 🔧 Edit / MultiEdit — 精确编辑

```json
{
  "name": "Edit",
  "input_schema": {
    "properties": {
      "path": { "type": "string", "description": "文件路径" },
      "old_string": { "type": "string", "description": "要替换的精确文本" },
      "new_string": { "type": "string", "description": "替换后的新文本" }
    },
    "required": ["path", "old_string", "new_string"]
  }
}
```

**Edit vs Write 的选择逻辑：**

```
改动较小（几行）  → 用 Edit（只传改动部分，token 更省）
改动较大（大重构）→ 用 Write（传完整文件更安全）
同文件多处改动   → 用 MultiEdit（批量原子性操作）
```

**MultiEdit 结构：**

```json
{
  "name": "MultiEdit",
  "input": {
    "path": "src/app.js",
    "edits": [
      { "old_string": "const foo = 1", "new_string": "const foo = 2" },
      { "old_string": "let bar = 'old'", "new_string": "let bar = 'new'" }
    ]
  }
}
```

> ⚠️ `old_string` 必须在文件中**唯一精确匹配**，否则报错

---

### 🔭 Glob — 文件搜索

```json
{
  "name": "Glob",
  "input_schema": {
    "properties": {
      "pattern": { "type": "string", "description": "Glob 模式，如 **/*.ts" },
      "path": { "type": "string", "description": "搜索根目录（可选）" }
    },
    "required": ["pattern"]
  }
}
```

**常用 Glob 模式：**

```bash
**/*.ts          # 所有 TypeScript 文件
src/**/*.test.js # src 下所有测试文件
**/node_modules  # 所有 node_modules（通常排除）
*.{js,ts,jsx}    # 多扩展名匹配
```

**Result 格式：**

```
返回文件路径列表，按修改时间排序（最新的在前）
适合 Claude 快速定位大型项目中的目标文件
```

---

### 🔍 Grep — 内容搜索

```json
{
  "name": "Grep",
  "input_schema": {
    "properties": {
      "pattern": { "type": "string", "description": "正则表达式或字符串" },
      "path": { "type": "string", "description": "搜索路径" },
      "include": { "type": "string", "description": "文件类型过滤，如 *.ts" }
    },
    "required": ["pattern"]
  }
}
```

**底层实现：**

- 默认使用内置 ripgrep（`USE_BUILTIN_RIPGREP=0` 可改用系统 rg）
- 支持正则，大型代码库中极快
- 可结合 `include` 过滤文件类型

---

### 💻 Bash — Shell 执行

```json
{
  "name": "Bash",
  "input_schema": {
    "properties": {
      "command": { "type": "string", "description": "Shell 命令" },
      "timeout": { "type": "integer", "description": "超时毫秒数" },
      "description": {
        "type": "string",
        "description": "命令意图说明（给用户看）"
      }
    },
    "required": ["command"]
  }
}
```

**Bash 工具的特殊行为（重要！）：**

```
✅ 工作目录持久化：cd /path 后续命令都在该目录执行
❌ 环境变量不持久：export FOO=bar 在下一个 Bash 调用中失效

原因：
  工作目录 (CWD) 是进程状态，在同一 shell session 中持久
  环境变量每次 Bash 调用都是新的 shell 子进程，不继承

解决环境变量问题：
  方法1：在同一个命令中设置并使用
    → "export FOO=bar && npm test"
  方法2：用 CLAUDE_ENV_FILE 配置持久化环境
    → export CLAUDE_ENV_FILE=/path/to/env.sh
```

**超时配置：**

```bash
CLAUDE_BASH_TIMEOUT_MS=120000  # 默认 120 秒
# 长时间任务（如构建）需要增大这个值
```

---

## 3.3 工具的 JSON Schema 定义深度解析

### Schema 的 description 对 Claude 行为的影响

```javascript
// Claude 读取工具 description 来决策：
// 1. 什么时候调用这个工具？
// 2. 参数怎么填？
// 3. 什么情况下不应该调用？

// 糟糕示例（太简单）
{
  "name": "DeleteFile",
  "description": "删除文件",
  "input_schema": { ... }
}
// → Claude 可能轻易删除重要文件

// 优秀示例（有约束和上下文）
{
  "name": "DeleteFile",
  "description": "永久删除指定文件。此操作不可逆！调用前必须：\n1. 确认用户明确要求删除\n2. 告知用户文件路径和后果\n3. 获得明确确认\n永远不要删除 .git 目录下的文件或系统关键文件。",
  "input_schema": { ... }
}
// → Claude 会非常谨慎地使用此工具
```

### input_schema 类型系统

```json
{
  "input_schema": {
    "type": "object",
    "properties": {
      // 字符串
      "path": { "type": "string", "description": "文件路径" },

      // 数字
      "limit": { "type": "integer", "minimum": 1, "maximum": 10000 },

      // 布尔
      "recursive": { "type": "boolean", "default": false },

      // 枚举
      "mode": { "type": "string", "enum": ["read", "write", "append"] },

      // 数组
      "paths": { "type": "array", "items": { "type": "string" } },

      // 嵌套对象
      "options": {
        "type": "object",
        "properties": {
          "encoding": { "type": "string" }
        }
      }
    },
    "required": ["path"] // 必填参数
  }
}
```

---

## 3.4 工具执行结果的序列化与回传

### 成功返回

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01XyyZ",
  "content": "文件内容或命令输出（纯文本或结构化数据）",
  "is_error": false
}
```

### 失败返回

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01XyyZ",
  "content": "Error: ENOENT: no such file or directory, open 'foo.js'",
  "is_error": true
}
```

### 返回内容的 token 成本

```
工具返回内容 → 追加到 messages → 消耗 context token

大文件的应对策略：
  ① 先 Glob → 找到目标文件
  ② 再 Grep → 定位关键行
  ③ 最后 Read(offset, limit) → 只读相关片段

避免：
  ❌ 一次性 Read 整个 10000 行的文件
  ✅ 先搜索锁定范围，再分段读取
```

---

## 3.5 并行工具调用的处理逻辑

（详见模块二 2.5，核心原则总结）

```javascript
// Claude 在一次响应中发出多个 tool_use（并行）
assistant_message.content = [
  { type: "text", text: "我同时检查这三个文件" },
  { type: "tool_use", id: "t1", name: "Read", input: { path: "a.ts" } },
  { type: "tool_use", id: "t2", name: "Read", input: { path: "b.ts" } },
  { type: "tool_use", id: "t3", name: "Grep", input: { pattern: "TODO" } },
];

// CLI 并发执行，统一回传（顺序可以不同但 id 对应）
user_message.content = [
  { type: "tool_result", tool_use_id: "t1", content: "..." },
  { type: "tool_result", tool_use_id: "t2", content: "..." },
  { type: "tool_result", tool_use_id: "t3", content: "..." },
];
```

**写操作的顺序保证：**

- 涉及 Write/Edit/Bash 等有副作用的工具，Claude 通常**不并行**发起
- 需要按序执行的操作会逐步进行（安全优先）

---

## 3.6 实验：自定义工具并注入 Agent

通过 **MCP（Model Context Protocol）** 可以添加自定义工具：

### 方式一：MCP 服务器（推荐）

```javascript
// my-tool-server.js（MCP Server）
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server(
  { name: "my-tools", version: "1.0.0" },
  {
    capabilities: { tools: {} },
  },
);

// 注册自定义工具
server.setRequestHandler("tools/list", async () => ({
  tools: [
    {
      name: "GetWeather",
      description: "获取指定城市的天气信息",
      inputSchema: {
        type: "object",
        properties: {
          city: { type: "string", description: "城市名称" },
        },
        required: ["city"],
      },
    },
  ],
}));

// 实现工具逻辑
server.setRequestHandler("tools/call", async (req) => {
  if (req.params.name === "GetWeather") {
    const { city } = req.params.arguments;
    // 实际调用天气 API
    return { content: [{ type: "text", text: `${city}: 晴，25°C` }] };
  }
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

### 在 Claude Code 中注册 MCP Server

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

### 方式二：通过 Hooks 扩展（无需 MCP）

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo '命令执行完毕，记录日志' >> /tmp/claude-audit.log"
          }
        ]
      }
    ]
  }
}
```

---

## 3.7 控制流阶段对照（工具管线）

在控制流分析的框架中，Tool Use 属于“模型输出 → 工具执行 → 结果回流”的中间管线，关键点如下：

1. **流式解析入口**：工具输入通常通过 SSE 事件逐步输出，CLI 需要对 tool_input JSON 进行增量解析与校验。
2. **执行策略**：只读工具可并行、写入工具顺序执行，避免竞态与副作用冲突。
3. **结果回流**：每个 tool_use 对应 tool_result，按照 id 回填到 messages，成为下一轮的 user 输入。
4. **边界控制**：工具管线与权限系统强耦合，执行前必须通过 allow/ask/deny 判定。

这解释了为什么 Tool Use 不只是“调用 API”，而是控制流中的关键中间层。

---

## 3.8 深入：文件编辑管道的内部实现

> 来源：[Southbridge.AI - File Editing Analysis](https://www.southbridge.ai/blog/claude-code-an-analysis-file-editing)（对 Claude Code 编译产物的逆向分析）

### 四阶段编辑管道（所有编辑工具共用）

```
用户触发 Edit / MultiEdit / Write
           │
    Phase 1: Validation（验证）
           │  ① 路径安全检查（绝对路径 + 越界检测 + 敏感路径拦截）
           │  ② Permission 审批（allow/ask/deny 见模块四）
           │  ③ 文件状态检查（mtime 对比，检测外部修改）
           │  ④ 内容正确性（old_string 存在 + 数量对）
           │  ⑤ 安全扫描
           ↓
    Phase 2: Preparation（准备）
           │  读取缓存、检测文件编码（UTF-8/UTF-16/ASCII）
           │  检测行尾符（\n / \r\n / \r）
           ↓
    Phase 3: Application（执行）
           │  字符串精确替换 / 覆盖写入（保留原编码 + 行尾符）
           ↓
    Phase 4: Verification（验证）
           │  生成 unified diff（返回给 Claude 作为观察结果）
           │  更新文件状态缓存（mtime + content hash）
           │  生成编辑位置上下文片段（±5 行）
           ↓
       返回 EditResult 给 Claude（作为 tool_result）
```

---

### ⚠️ 行号陷阱：最常见的 Edit 失败原因

Read 工具的输出**带行号前缀**（`1\t代码内容`），但 Edit 的 `old_string` **必须不含行号**。

```
Read 返回给 Claude 的内容：
  1    function hello() {
  2      console.log('Hello, world!');
  3    }

❌ 错误（含行号）：
  old_string = "2  console.log('Hello, world!');"

✅ 正确（去掉行号和Tab）：
  old_string = "  console.log('Hello, world!');"
```

**系统的自动检测与提示**：

如果 `old_string` 包含行号模式（`^\d+\t`），系统会：

1. 报错提示"old_string contains line number prefix"
2. 自动给出去掉行号后的 suggestion
3. 如果完全找不到匹配，还会尝试推测是哪一行并提示

> 这是 Claude Code 错误恢复设计的典型样本：不只报错，还给出下一步行动建议。

---

### EditTool：精确字符串替换的关键设计

**强制 "先 Read 再 Edit" 不是约定，是技术强制：**

```
EditTool 执行步骤：
  Step 1: 检查 readFileState 缓存（必须有之前的 Read 记录）
           → 缓存不存在 → 立即报错："File must be read with ReadFileTool before editing"
  Step 2: 对比当前文件 mtime 与缓存的 timestamp
           → 不一致 → 报错："File has been modified externally. Please read again."
  Step 3: 验证 old_string
           → 找不到 → 报错 + findSimilarStrings 提示
           → 数量 ≠ expected_replacements → 报错（防意外多替换）
  Step 4: 执行替换（字节精确，保留编码和行尾符）
  Step 5: 生成 diff + 更新缓存
```

**`expected_replacements` 的价值：**

```javascript
// 如果 old_string 在文件中有 3 处匹配：
// 不设 expected_replacements → 默认期望 1 次 → 报错，要求澄清
// 设 expected_replacements: 3 → 允许替换全部 3 处

// 设计原因：避免 Claude 意外替换不该动的位置
```

---

### MultiEditTool：内存模拟 + 一次写入的原子性

```
MultiEdit 的执行策略（不是"执行一个写一次"）：

Step 1: 加载文件（从缓存）
Step 2: 在内存副本上模拟所有 edits（Dry Run）
         → 检测三类冲突：
           ① 依赖冲突：edit[j].old_string 依赖 edit[i] 的结果（但顺序错了）
           ② 位置重叠：两个 edit 修改的文本区域有交叉
           ③ 逻辑矛盾：edit[i] 和 edit[j] 替换同一文本为不同内容
         → 任何 edit 找不到匹配 → 全部中止
Step 3: 全部验证通过 → 一次性写入磁盘（原子操作）
Step 4: 更新缓存

关键：
  中途失败 → 文件完全不变（no partial writes）
  类比：数据库事务（all or nothing）
```

---

### 五层验证防护（Path Safety 细节）

```javascript
// Layer 1 - 路径安全：
validatePath(filePath) {
  // 必须是绝对路径
  if (!path.isAbsolute(filePath)) → 报错 + suggest path.resolve()

  // 防止 path traversal 攻击（../../../etc/passwd）
  if (resolved !== normalized) → 报错

  // 禁止越界（只允许 projectRoot + additionalWorkingDirectories）
  if (!allowed.some(dir => resolved.startsWith(dir))) → 报错

  // 禁止修改敏感路径（正则匹配）：
  forbidden = [/.git\//, /node_modules\//, /\.env$/, /\.ssh\//, /\.gnupg\//]
}
```

---

### 错误恢复：三路合并（外部修改冲突）

这是文章中最精彩的部分——当文件在 Claude 编辑过程中**被外部（如另一程序）修改**时：

```
情况：Claude 读取了文件（cached），之后文件被外部修改了

恢复策略（三路合并）：
  Base  = 上次 Read 时的缓存内容
  Ours  = Claude 要应用的修改
  Theirs = 外部修改后的当前文件内容

  ① 无冲突 → 自动合并 + 警告用户
  ② 有冲突 → 插入 git 风格的冲突标记（<<<<<<< / ======= / >>>>>>>）
  ③ 无法合并 → 返回三个选项：
     a. 覆盖（丢弃外部改动）
     b. 中止（保留外部改动）
     c. 重新读取（让 Claude 看到新内容后重新决策）
```

---

### 边界情况的精细处理

| 情况       | 处理方式                                                              |
| ---------- | --------------------------------------------------------------------- |
| 空文件     | Edit 拒绝（报错建议用 Write）；Read 返回 `<system-reminder>` 标签提示 |
| 二进制文件 | 前 8192 字节检测 null byte + magic number（PNG/JPG/PDF/ZIP）          |
| 符号链接   | 提示"将修改目标文件"，允许继续                                        |
| 编码检测   | BOM 检测 → UTF-8/UTF-16 → ASCII fallback → 不确定时警告               |
| 磁盘满     | 报告需要多少字节 + 分析哪些目录可清理                                 |
| 写入不完整 | 检查 `.backup` / `.tmp` 临时文件，尝试自动恢复                        |

---

## 🔑 核心要点总结

| 知识点          | 关键结论                                                        |
| --------------- | --------------------------------------------------------------- |
| **工具定义**    | name + description + input_schema，description 影响 Claude 决策 |
| **Read 限制**   | 默认 2000 行，大文件需 offset+limit 分页                        |
| **Write 规则**  | 修改已有文件必须先 Read，Write 写全量内容                       |
| **Edit 优势**   | 小改动用 Edit/MultiEdit，省 token，安全                         |
| **Bash 特性**   | CWD 持久，环境变量不持久，每次 Bash 是新 shell                  |
| **并行工具**    | 读操作可并行，写操作按序执行                                    |
| **tool_result** | is_error=true 让 Claude 自动应对失败                            |
| **MCP 扩展**    | 通过 MCP Server 注册自定义工具，无限扩展能力                    |

---

## 📝 思考题

1. 为什么 Edit 工具要求 `old_string` 唯一匹配？如果不唯一会发生什么？
   - **解答**：如果 `old_string` 不唯一，Claude 无法确定应该替换哪一个匹配项，可能导致意外修改其他位置的代码。这会降低工具的精确性，并可能引入难以察觉的错误。

2. Bash 工具环境变量不持久这个设计是 bug 还是 feature？为什么？
   - **解答**：这是一个 feature。为了安全和隔离，每个 `Bash` 工具调用都运行在一个新的 shell 进程中，确保一个命令的副作用不会影响后续命令。这强制开发者在每次调用时明确传递所需的全部环境变量。

3. 如果你要实现一个"数据库查询"工具，description 应该写什么来防止 Claude 误用？
   - **解答**：应在 description 中明确该工具为**只读**属性（仅允许 `SELECT` 语句），要求 Claude 在执行查询前必须先调用特定工具了解 Schema（表结构），并明确禁止任何数据修改操作（如 `UPDATE` 或 `DELETE`）。

4. MCP 和内置工具的权限模型有什么区别？（→ 模块四）

---

## 📚 延伸阅读

- [Anthropic：Tool Use API](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview)
- [Anthropic：Tool Use Best Practices](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/implement-tool-use)
- [MCP 协议规范](https://modelcontextprotocol.io/docs)
- [Claude Code Settings - Tools](https://docs.anthropic.com/en/docs/claude-code/settings#tools-available-to-claude)

---

> ✅ 模块三完成。下一步：**模块四 - Permission 权限与安全模型**
