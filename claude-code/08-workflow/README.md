# 模块八：Workflow 工作流系统

> 学习目标：掌握 Hooks 钩子系统与 Headless 模式，理解如何把 Claude Code 集成到自动化流程中

---

## 8.1 Hook 生命周期（全局事件）

```
SessionStart → UserPromptSubmit → PreToolUse → PermissionRequest → PostToolUse / PostToolUseFailure → Stop → SessionEnd
```

还有：SubagentStart/Stop、PreCompact、ConfigChange、Notification 等事件。

---

## 8.2 Hook 配置格式

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          { "type": "command", "command": "prettier --write \"$CLAUDE_FILE_PATH\"" }
        ]
      }
    ]
  }
}
```

字段说明：
- `matcher`：匹配工具名
- `type`：`command` / `prompt` / `agent`
- `async`：后台执行，不阻塞

---

## 8.3 stdin/stdout/exit code 协议

```
stdin：Claude Code 传入 JSON 上下文
stdout：注入到 Claude context（部分事件）
stderr：显示给用户
exit code 2：阻断当前操作
```

---

## 8.4 常用 Hook 场景

- **SessionStart**：注入项目上下文
- **PreToolUse**：拦截危险命令
- **PostToolUse**：自动格式化、自动测试
- **Stop**：任务完成后通知

---

## 8.5 异步 Hook

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "npm test", "async": true }
        ]
      }
    ]
  }
}
```

适合耗时任务：测试、构建、报告生成。

---

## 8.6 Headless 模式（CI/CD）

```bash
claude -p "运行测试并生成报告" > report.md
```

典型用法：
- CI 里自动代码审查
- 自动修复 lint
- 输出结构化报告

---

## 8.7 Hook 安全注意

- Hook 脚本以当前用户权限执行。
- 不信任项目需先审查 `.claude/settings.json`。
- 避免直接执行未经校验的输入。

---

## 8.8 核心要点总结

| 知识点 | 关键结论 |
| --- | --- |
| Hooks | 覆盖会话全生命周期 |
| 协议 | stdin JSON，exit 2 可拦截 |
| 异步 | 不阻塞主流程 |
| Headless | 适合 CI/CD 自动化 |

---

## 8.9 思考题

1. 你会如何用 Hook 实现“写文件后自动运行对应测试”？
2. SessionStart 注入内容过大，会带来什么问题？
3. 什么时候应当用 `prompt` 或 `agent` 类型 Hook？

---

## 8.10 延伸阅读

- [Anthropic：Hooks 参考](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [Claude Code：GitHub Actions 集成](https://docs.anthropic.com/en/docs/claude-code/github-actions)

---

模块八完成。下一步：模块九 - 实战综合项目
