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
2. 后台自动化必须通过 OpenClaw Sessions API 调度 Claude Code，不要使用聊天 slash command 作为后台默认路径。
3. 第一轮必须调用 `sessions_spawn(runtime="acp", agentId="claude", mode="run", thread=false, cwd="/srv/openclaw-runner/repos/agentic-ai", prompt=<完整任务 prompt>)`。
4. `<完整任务 prompt>` 必须包含任务包路径、`templates/acp_steer_prompt.md` 的任务要求、目标仓库路径、禁止读取范围和输出格式要求；不要把修复限定成固定文件模板。
5. 必须记录返回的完整 `childSessionKey`，格式为 `agent:claude:acp:...`，并在最终报告中作为 Claude Code session 输出。
6. 如果需要第二轮补漏，使用 `sessions_send(sessionKey=<childSessionKey>, prompt=<显式上下文 follow-up prompt>)`；follow-up prompt 必须带上上一轮输出、当前 git diff、验收缺失项和本轮目标，不要假设 Claude 自动记得上一轮。
7. `/acp doctor`、`/acp spawn`、`/acp steer` 仅用于飞书交互演示；它们不是 shell 命令，不要写入 bash/SSH/subprocess。
8. 如果 `sessions_spawn` 无法创建 Claude Code child session，必须返回 `failed` 并说明 ACP 阻塞原因；不要改用 OpenClaw 手工修复。
9. 不要把 Claude Code CLI 作为本 Skill 的常规降级路径。
10. Claude Code 只负责本地仓库修复，不要在 Claude Code 会话内 push，不要创建 PR，不要输出完整密钥。
11. OpenClaw 在验收通过后执行 commit 和 push。
12. 不要 force push，不要清理 Git 历史，不要自动合并分支。
13. 不要在 GitHub 仓库中提交 `security-report.md`；巡检报告通过飞书发送，必要时归档到 `/srv/openclaw-runner/reports/agentic-ai-security-report.md`。
14. 如果发现疑似真实密钥已进入 Git 历史，继续完成自动修复、commit、push 和飞书报告；不要因此把任务停在人工审核状态，只需在飞书报告的风险备注中记录风险类型、疑似文件、脱敏片段和建议动作。

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
