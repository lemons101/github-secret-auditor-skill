# 第 19 节前置部署参考

本文件用于课程环境初始化，不属于 Skill 的默认执行流程。只有在需要部署 OpenClaw、Claude Code、ACP 或排查写入权限时读取。

## 目标

让 OpenClaw 能通过 ACP 调度 Claude Code，并允许 Claude Code 在授权仓库目录内完成必要的文件修改。飞书交互和后台自动化都应复用同一套 ACP 调度能力。

## 推荐目录

```text
/root/projects/github-secret-auditor-skill
/srv/openclaw-runner/repos
/srv/openclaw-runner/tasks
/srv/openclaw-runner/reports
```

## ACP 健康检查

在 OpenClaw/飞书聊天对话框中执行。`/acp ...` 是聊天 slash command，不是 shell 命令，不要在 SSH/bash/PowerShell 里执行：

```text
/acp doctor
```

应确认：

```text
configuredBackend: acpx
registeredBackend: acpx
healthy: yes
```

继续验证可创建 Claude Code 会话：

```text
/acp spawn claude --mode persistent --thread off --cwd /srv/openclaw-runner/repos/agentic-ai
```

应返回完整 `session-key`，例如 `agent:claude:acp:...`。

后台任务不应把 `/acp ...` 当 shell 命令执行；应调用 OpenClaw 内部 ACP service，或把 ACP 指令作为聊天消息提交给同一套 slash command router。

## 写入权限说明

ACP 会话通常是非交互式运行。如果权限策略过于保守，Claude Code 可能可以搜索和分析仓库，但无法执行 `Edit`、`Write` 或修改授权仓库文件。

课程演示环境可以在明确授权的仓库范围内，临时使用更宽松的 ACPX 权限策略。该配置是持久写入 OpenClaw 配置文件的，不是一次性命令。任务完成后应恢复保守策略。

示例命令：

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

配置后必须重启 OpenClaw gateway，并重新执行：

```text
/acp doctor
```

然后重新创建 Claude ACP 会话。旧 session 可能沿用旧权限。

## 恢复保守策略

课程演示结束后，可恢复更保守的策略：

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-reads
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions deny
```

恢复后同样需要重启 OpenClaw gateway。

## 安全边界

- 不要把 ACP 会话 `--cwd` 指向 `/root`、用户 home、系统目录或包含生产密钥的目录。
- 只把 `--cwd` 指向明确授权的仓库，例如 `/srv/openclaw-runner/repos/agentic-ai`。
- 即使使用 `approve-all`，任务 prompt 仍必须禁止读取 `.env`、私钥、Cookie、生产配置和用户个人目录。
- 如果发现真实密钥进入 Git 历史，代码修复、commit、push 和飞书报告仍应自动完成；同时在飞书报告风险备注中记录风险类型、疑似文件、脱敏片段和建议动作。
