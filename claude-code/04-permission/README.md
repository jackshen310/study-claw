# 模块四：Permission 权限与安全模型

> 学习目标：理解 Claude Code 如何通过权限规则与沙箱，控制工具调用的安全边界
>
> 核心问题：如何让 Agent "能做事" 且 "不乱来"？

---

## 4.1 风险分级与默认行为

```
风险等级        操作类型                是否需要确认
────────────────────────────────────────────────────
🟢 低风险   Read / Glob / Grep         默认自动
🟡 中风险   Write / Edit / MultiEdit   默认需确认（首次）
🟠 高风险   Bash 执行                  默认需确认
🔴 极高风险  rm -rf / sudo / 外网请求  强烈建议 deny
```

---

## 4.2 规则体系：allow / ask / deny

### 规则优先级

```
deny（最高） → ask → allow → 默认行为
```

### 规则语法

```
Tool
Tool(specifier)

*  匹配单级路径
** 匹配多级路径
```

### 示例配置

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Bash(git diff *)",
      "Bash(npm run *)",
      "Read(**/*.ts)"
    ],
    "ask": [
      "Write(**)",
      "Bash(git commit *)"
    ],
    "deny": [
      "Bash(rm *)",
      "Bash(sudo *)",
      "Read(**/.env)",
      "WebFetch"
    ]
  }
}
```

---

## 4.3 权限模式与危险参数

### `--dangerously-skip-permissions`

```
效果：绕过所有权限确认
适合：完全受控的 CI / 沙箱环境
风险：任何工具调用都可直接执行
```

### defaultMode（权限模式）

```json
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}
```

可选值：
- `default`：按规则询问
- `acceptEdits`：自动接受 Write/Edit（Bash 仍需确认）
- `plan`：只读模式（不执行写操作）
- `bypassPermissions`：完全绕过

---

## 4.4 沙箱机制：文件系统与网络隔离

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["docker", "git"],
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org"],
      "allowUnixSockets": ["/var/run/docker.sock"],
      "allowLocalBinding": true
    }
  }
}
```

**文件系统边界**：
- 默认只允许当前项目目录。
- 可用 `additionalDirectories` 放行额外路径。

---

## 4.5 Bash 权限策略（实用模板）

```json
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(npm run test)",
      "Bash(npm run lint)"
    ],
    "ask": ["Bash(git commit *)", "Bash(git push *)"],
    "deny": ["Bash(rm *)", "Bash(sudo *)", "Bash(curl *)"]
  }
}
```

---

## 4.6 安全风险与最佳实践

### Prompt Injection 风险

恶意文件内容可能诱导执行危险命令。防护原则：

- 最小权限原则（只允许必要工具）
- deny 优先
- 沙箱隔离
- 不在 root 权限下运行
- 审查 MCP 与 Hooks 来源

---

## 4.7 核心要点总结

| 知识点 | 关键结论 |
| --- | --- |
| 规则优先级 | deny > ask > allow |
| 权限模式 | 支持只读、自动编辑、完全绕过 |
| 沙箱 | 文件系统 + 网络双隔离 |
| 风险控制 | 最小权限 + deny 优先 + 沙箱 |

---

## 4.8 思考题

1. 为什么 deny 必须最高优先级？
2. `acceptEdits` 与 `bypassPermissions` 的本质区别是什么？
3. 你会如何为 CI 设计一个既安全又高效的权限配置？

---

## 4.9 延伸阅读

- [Claude Code Permissions](https://docs.anthropic.com/en/docs/claude-code/permissions)
- [Claude Code Sandboxing](https://docs.anthropic.com/en/docs/claude-code/sandboxing)

---

模块四完成。下一步：模块五 - Context Window 上下文管理
