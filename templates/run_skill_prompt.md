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
巡检 -> 通过 ACP 调度 Claude Code 修复 -> OpenClaw 验收 -> commit -> push 到 GitHub -> 飞书汇报
```

硬性要求：

1. 必须使用 Skill 中定义的流程，不要用 OpenClaw 自己手工替代 Claude Code 的代码巡检和修复。
2. 默认必须优先使用 ACP 调度 Claude Code：创建 Claude Code ACP 会话，记录完整 `session-key`，再通过 ACP steer 投递巡检修复任务。
3. 如果当前聊天通道不能直接执行 `/acp spawn`、`/acp steer`，不要把它当成服务器 shell 命令，也不要直接放弃 ACP；请调用 OpenClaw 内部等价的 ACP 后端能力完成 spawn/steer。
4. 只有 ACP 后端能力确实不可用时，才允许降级使用 Claude Code headless CLI，例如 `claude -p "..."`；降级时必须在最终回复中说明 ACP 失败原因。
5. 如果当前环境既无法调用 ACP Claude Code 会话，也无法调用 Claude Code CLI，必须返回 `failed`，并明确说明是 ACP 后端不可用、Claude Code CLI 不可用、权限不足，还是当前通道不支持；不要绕过 Claude Code 手工修复。
6. Claude Code 只负责本地仓库修复，不要在 Claude Code 会话内 push，不要创建 PR，不要输出完整密钥。
7. OpenClaw 在验收通过后执行 commit 和 push。
8. 不要 force push，不要清理 Git 历史，不要自动合并分支。
9. 不要在 GitHub 仓库中提交 `security-report.md`；巡检报告通过飞书发送，必要时归档到 `/srv/openclaw-runner/reports/agentic-ai-security-report.md`。
10. 如果发现疑似真实密钥已进入 Git 历史，最终状态必须是 `needs_review`，并提醒人工吊销和轮换密钥。

最终飞书回复格式：

```text
状态：
passed / needs_review / failed

目标仓库：
lemons101/agentic-ai

是否调用 Claude Code：
yes / no

调用方式：
acp / claude-cli / failed

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
