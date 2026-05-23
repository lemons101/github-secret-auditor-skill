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
2. 必须通过 ACP 调度 Claude Code。`/acp doctor`、`/acp spawn`、`/acp steer` 是 OpenClaw 对话框 slash command，不是 shell 命令；必须作为聊天消息发送给 OpenClaw 对话/消息路由执行。
3. 在对话框或同一 slash command router 中按顺序执行：`/acp doctor` -> `/acp spawn claude --mode persistent --thread off --cwd /srv/openclaw-runner/repos/agentic-ai` -> `/acp steer --session <session-key> ...`。
4. 必须记录完整 `session-key`，格式为 `agent:claude:acp:...`。如果已有可用 session，可以复用该 session 直接 steer。
5. 如果当前是后台任务，必须复用 OpenClaw 的 ACP 后端能力，或把 `/acp ...` 作为聊天消息提交给同一套 slash command router；不要写入 shell、subprocess、SSH 命令或脚本里执行。
6. 如果 ACP 无法创建 Claude Code session，必须返回 `failed` 并说明 ACP 阻塞原因；不要改用 OpenClaw 手工修复。
7. 不要把 Claude Code CLI 作为本 Skill 的常规降级路径。
8. Claude Code 只负责本地仓库修复，不要在 Claude Code 会话内 push，不要创建 PR，不要输出完整密钥。
9. OpenClaw 在验收通过后执行 commit 和 push。
10. 不要 force push，不要清理 Git 历史，不要自动合并分支。
11. 不要在 GitHub 仓库中提交 `security-report.md`；巡检报告通过飞书发送，必要时归档到 `/srv/openclaw-runner/reports/agentic-ai-security-report.md`。
12. 如果发现疑似真实密钥已进入 Git 历史，继续完成自动修复、commit、push 和飞书报告；不要因此把任务停在人工审核状态，只需在飞书报告的风险备注中记录风险类型、疑似文件、脱敏片段和建议动作。

最终飞书回复格式：

```text
状态：
passed / failed

目标仓库：
lemons101/agentic-ai

是否调用 Claude Code：
yes / no

调用方式：
acp / failed

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

风险备注：
- ...

下一步建议：
- ...
```
