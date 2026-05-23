请读取并严格执行 `/root/projects/github-secret-auditor-skill/SKILL.md`。

目标仓库：

```text
https://github.com/lemons101/agentic-ai.git
```

本地工作目录：

```text
/srv/openclaw-runner/repos/agentic-ai
```

执行目标：

```text
巡检 -> 调度 Claude Code 修复 -> OpenClaw 验收 -> commit -> push 到 GitHub -> 飞书汇报
```

硬性要求：

1. 必须使用 Skill 中定义的流程，不要用 OpenClaw 自己手工替代 Claude Code 的代码巡检和修复。
2. 自动化默认优先使用 Claude Code headless CLI，例如 `claude -p "..."`，启动 Claude Code 执行仓库巡检和本地修复。
3. 如果 CLI 不可用，可以使用当前通道支持的 ACP 会话能力；但不要把聊天 slash command 当成服务器 shell 命令。
4. 如果当前环境既无法调用 Claude Code CLI，也无法调用 ACP Claude Code 会话，必须返回 `failed`，并明确说明是 Claude Code CLI 不可用、ACP 不可用、权限不足，还是当前通道不支持；不要绕过 Claude Code 手工修复。
5. Claude Code 只负责本地仓库修复，不要在 Claude Code 会话内 push，不要创建 PR，不要输出完整密钥。
6. OpenClaw 在验收通过后执行 commit 和 push。
7. 不要 force push，不要清理 Git 历史，不要自动合并分支。
8. 不要在 GitHub 仓库中提交 `security-report.md`；巡检报告通过飞书发送，必要时归档到 `/srv/openclaw-runner/reports/agentic-ai-security-report.md`。
9. 如果发现疑似真实密钥已进入 Git 历史，最终状态必须是 `needs_review`，并提醒人工吊销和轮换密钥。

最终飞书回复格式：

```text
状态：
passed / needs_review / failed

目标仓库：
lemons101/agentic-ai

是否调用 Claude Code：
yes / no

调用方式：
claude-cli / acp / failed

Claude Code session：
<session-key 或失败原因>

是否已推送到 GitHub：
yes / no

commit：
<commit hash>

修改文件：
- ...

风险摘要：
- ...

已完成修复：
- ...

残余风险：
- ...

下一步建议：
- ...
```
