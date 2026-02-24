# 模块八：Workflow 工作流系统

> 学习目标：掌握 Claude Code 的 Hooks 钩子系统，理解如何通过自动化脚本扩展 Claude Code 的能力，以及如何在 CI/CD 中使用 Headless 模式
>
> 核心问题：如何让 Claude Code 自动触发自定义逻辑？如何在无人值守环境中运行？

---

## 8.1 Hooks 钩子系统：完整生命周期

### 所有 Hook 事件（按执行顺序）

```
会话级（Session）
━━━━━━━━━━━━━━━━━━
SessionStart          ← 会话开始或恢复时
    │
    │  （用户输入后）
UserPromptSubmit      ← 用户提交 prompt，Claude 处理前
    │
    │  （Claude 决定使用工具）
PreToolUse            ← 工具执行前（可以拦截！）
    │
PermissionRequest     ← 权限检查时（可以自动批准/拒绝）
    │
    │  （工具执行）
PostToolUse           ← 工具成功执行后
PostToolUseFailure    ← 工具执行失败后
    │
    │  （Subagent 相关）
SubagentStart         ← Subagent 启动时
SubagentStop          ← Subagent 结束时
    │
    │  （任务相关）
TaskCompleted         ← Agent 认为任务完成时
Stop                  ← Agent 停止响应时
TeammateIdle          ← Agent Team 中其他 Agent 空闲时
    │
    │  （其他事件）
Notification          ← 需要发送通知时
PreCompact            ← Context 压缩前
ConfigChange          ← 配置文件变化时
    │
SessionEnd            ← 会话结束时
━━━━━━━━━━━━━━━━━━
```

---

## 8.2 Hook 配置格式与语法

### 在 settings.json 中配置

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write \"$CLAUDE_FILE_PATH\""
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/scripts/validate-bash.sh"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Branch: $(git branch --show-current)\nLast commit: $(git log -1 --oneline)\""
          }
        ]
      }
    ]
  }
}
```

### Hook 配置字段

```json
{
  "matcher": "工具名正则表达式，如 Bash|Write|Edit",
  "hooks": [
    {
      "type": "command", // 执行 Shell 命令
      "command": "your-script.sh",
      "timeout": 30, // 超时秒数（默认无限）
      "async": false // true = 后台运行，不阻塞
    }
  ]
}
```

---

## 8.3 Hook 的 stdin/stdout/退出码协议

### Hook 脚本通信机制

```
Claude Code ──stdin(JSON)──→ Hook 脚本
Claude Code ←──stdout────── Hook 脚本（反馈给 Claude）
Claude Code ←──stderr────── Hook 脚本（显示给用户）
Claude Code ←──exit code─── Hook 脚本（控制是否继续）
```

### 输入：stdin JSON 格式

```json
// Claude Code 传给 Hook 的 JSON（通过 stdin）
{
  "session_id": "abc123",
  "transcript_path": "/home/user/.claude/projects/.../transcript.jsonl",
  "cwd": "/home/user/my-project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

### 退出码语义

| 退出码 | 含义                     | 典型场景                     |
| ------ | ------------------------ | ---------------------------- |
| `0`    | 成功，继续执行           | Hook 验证通过                |
| `1`    | 失败（非阻断），继续执行 | Hook 内部错误，不影响 Claude |
| `2`    | **阻断！** 终止当前操作  | 拦截危险命令                 |

### exit code 2 的效果（按事件类型）

| Hook 事件          | exit 2 效果                                        |
| ------------------ | -------------------------------------------------- |
| `PreToolUse`       | **拦截工具调用**，stderr 内容作为错误反馈给 Claude |
| `UserPromptSubmit` | 阻止 Claude 处理该 prompt                          |
| `Stop`             | 阻止 Claude 停止，继续执行                         |
| `PostToolUse`      | 仅记录，不影响已执行的工具                         |
| `SessionStart`     | 仅记录                                             |

---

## 8.4 核心 Hook 事件详解与实用示例

### SessionStart：注入上下文

```bash
#!/bin/bash
# ~/.claude/hooks/session-context.sh
# 会话开始时自动注入项目上下文

cat << EOF
=== 项目状态 ===
当前分支: $(git branch --show-current 2>/dev/null || echo "非 git 项目")
最近提交: $(git log -1 --oneline 2>/dev/null || echo "无")
待处理 Issues:
$(cat TODO.md 2>/dev/null | head -5 || echo "无 TODO.md")

Node 版本: $(node --version 2>/dev/null || echo "未安装")
包管理器: $(cat package.json 2>/dev/null | jq -r '.packageManager' || echo "npm")
=================
EOF
```

```json
// settings.json 配置
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/session-context.sh"
          }
        ]
      }
    ]
  }
}
```

> ⚡ SeesionStart 的 **stdout** 会直接注入到 Claude 的 context！

---

### PreToolUse：拦截危险操作

```bash
#!/bin/bash
# ~/.claude/hooks/validate-bash.sh
# 拦截危险的 Bash 命令

# 从 stdin 读取 Hook 上下文 JSON
input=$(cat)
command=$(echo "$input" | jq -r '.tool_input.command')

# 检查危险命令
dangerous_patterns=("rm -rf" "rm -f /" "dd if=" "mkfs" ":(){ :|:& };:")
for pattern in "${dangerous_patterns[@]}"; do
  if [[ "$command" == *"$pattern"* ]]; then
    echo "🚫 拦截危险命令: $pattern" >&2
    echo "命令内容: $command" >&2
    exit 2  # 阻止执行！
  fi
done

# 记录所有执行的命令（审计日志）
echo "[$(date)] CMD: $command" >> ~/.claude/command-audit.log

exit 0  # 允许继续
```

---

### PostToolUse：自动格式化

```bash
#!/bin/bash
# 文件写入后自动运行格式化工具

input=$(cat)
tool_name=$(echo "$input" | jq -r '.tool_name')
file_path=$(echo "$input" | jq -r '.tool_input.file_path // .tool_input.path // ""')

if [[ -z "$file_path" ]]; then
  exit 0
fi

# 根据文件类型选择格式化工具
case "$file_path" in
  *.ts|*.tsx|*.js|*.jsx)
    prettier --write "$file_path" 2>/dev/null
    ;;
  *.py)
    black "$file_path" 2>/dev/null
    ;;
  *.go)
    gofmt -w "$file_path" 2>/dev/null
    ;;
esac

exit 0
```

---

### Stop Hook：任务完成后通知

```bash
#!/bin/bash
# Claude 完成任务时发送系统通知

input=$(cat)
session_id=$(echo "$input" | jq -r '.session_id')

# macOS 系统通知
osascript -e 'display notification "Claude Code 任务完成！" with title "Claude Code"'

# 或者发送到 Slack
# curl -X POST "$SLACK_WEBHOOK" \
#   -H "Content-Type: application/json" \
#   -d "{\"text\": \"Claude Code 任务完成 (Session: $session_id)\"}"

exit 0
```

---

## 8.5 异步 Hook（后台运行）

### 配置异步 Hook

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npm test -- --passWithNoTests",
            "async": true // ← 后台运行，不阻塞 Claude
          }
        ]
      }
    ]
  }
}
```

**适用场景：**

```
同步 Hook：                   异步 Hook：
- 拦截验证（PreToolUse）      - 运行测试（耗时）
- 快速格式化（<1s）           - 生成文档
- 日志记录                   - 发送通知
                             - 构建检查
```

---

## 8.6 Headless 模式：CI/CD 集成

### 无人值守运行方式

```bash
# 基本 headless 模式（-p 或 --print）
claude -p "运行测试并修复所有 TypeScript 类型错误"

# 输出结果到文件
claude -p "分析安全漏洞" > security-report.md

# 管道集成
echo "请为这段代码添加单元测试" | claude -p --output-format json

# 使用 --dangerously-skip-permissions（CI/CD）
claude -p "自动修复 lint 错误" \
  --dangerously-skip-permissions \
  --allowedTools "Write,Edit,Bash(npm run lint -- --fix)"
```

### GitHub Actions 集成示例

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Run Claude Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # 获取变更文件
          changed_files=$(git diff --name-only origin/main)

          # 让 Claude 审查变更
          claude -p "
            请审查以下变更文件并提供代码质量反馈：
            $changed_files

            重点检查：
            1. 潜在的 bug
            2. 安全漏洞
            3. 代码规范问题

            输出 Markdown 格式的审查报告。
          " --dangerously-skip-permissions > review.md

          # 将报告作为 PR 评论
          gh pr comment ${{ github.event.number }} --body-file review.md
        env:
          GH_TOKEN: ${{ github.token }}
```

### Headless 模式的 settings.json

```json
// CI/CD 专用配置 (.claude/settings.json)
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git *)",
      "Read(**)",
      "Write(src/**)",
      "Edit(src/**)"
    ],
    "deny": ["Bash(rm *)", "Bash(curl *)", "WebFetch"]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint -- --fix",
            "async": true
          }
        ]
      }
    ]
  }
}
```

---

## 8.7 Prompt-Based Hooks 和 Agent-Based Hooks

### Prompt-Based Hook（LLM 判断）

```json
// 让 Claude 来决定 Stop Hook 是否允许停止
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "请检查对话历史。如果用户的所有需求都已满足，返回 {\"decision\": \"allow\"}。如果还有未完成的任务，返回 {\"decision\": \"block\", \"reason\": \"原因\"}",
            "model": "haiku" // 用便宜的模型做判断
          }
        ]
      }
    ]
  }
}
```

### Agent-Based Hook（Subagent 执行）

```json
// 使用专门的 Subagent 处理 Hook 逻辑
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "agent",
            "agent": "code-reviewer", // 调用自定义 Subagent
            "async": true
          }
        ]
      }
    ]
  }
}
```

---

## 8.8 Hook 安全警告

```
⚠️  Hooks 的安全风险：
  - Hook 脚本以当前用户权限执行
  - 恶意项目的 .claude/settings.json 中的 Hook 可执行任意命令
  - 项目 Hook 需审查后再信任

✅ 安全最佳实践：
  1. 对不信任项目：打开前先阅读 .claude/settings.json
  2. Hook 脚本中不要使用未验证的输入
  3. 永远用引号包裹 shell 变量（防注入）
  4. 避免直接将 tool_input 传递给 eval 或 bash -c
```

---

## 🔑 核心要点总结

| 知识点                | 关键结论                                               |
| --------------------- | ------------------------------------------------------ |
| **Hook 类型**         | 14 种生命周期事件，覆盖会话全程                        |
| **通信协议**          | stdin 接收 JSON，stdout 反馈给 Claude，exit 2 阻断操作 |
| **SessionStart**      | stdout 直接注入 Claude context，用于动态注入上下文     |
| **PreToolUse**        | exit 2 = 拦截工具调用，是最强大的安全控制点            |
| **异步 Hook**         | `async: true` 后台运行，不阻塞 Claude 继续工作         |
| **Headless 模式**     | `-p` 参数无交互运行，适合 CI/CD                        |
| **Prompt/Agent Hook** | 可以用 LLM 或 Subagent 来做 Hook 逻辑判断              |
| **安全**              | 不信任来源的 Hook 有安全风险，审查后再用               |

---

## 📝 思考题


2. SessionStart 的 stdout 注入到 Claude context 有什么限制？会增加多少 token 消耗？
3. `async: true` 的 Hook 如果执行失败，Claude 还能感知到吗？
4. 如何利用 Hook 系统实现"每次文件修改后自动运行对应的测试"？

---

## 📚 延伸阅读

- [Anthropic：Hooks 完整参考文档](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [Claude Code：CI/CD 集成指南](https://docs.anthropic.com/en/docs/claude-code/github-actions)
- [Claude Code：GitHub Actions](https://github.com/anthropics/claude-code-action)

---

> ✅ 模块八完成。最后一步：**模块九 - 实战综合项目**
