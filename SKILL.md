---
name: github-secret-auditor
description: 当用户希望让 OpenClaw 通过 ACP 调度 Claude Code，对 GitHub 仓库进行 API Key、Token、密码、私钥、Webhook URL 等敏感信息泄露巡检、自动修复、验收、推送修复 commit，并通过飞书发送巡检报告时使用。
---

# GitHub 密钥泄露巡检 Skill

## 目标

本 Skill 用于让 OpenClaw 通过 ACP 调度 Claude Code，对授权 GitHub 仓库执行密钥泄露巡检、安全修复、验收、推送修复 commit，并通过飞书发送巡检报告。

核心原则：

- OpenClaw 负责任务编排、ACP 调度、仓库准备、验收、commit、push 和飞书报告。
- Claude Code 负责在目标仓库内执行代码级搜索、修复和验证。
- 不允许 OpenClaw 跳过 Claude Code 手工替代巡检或修复。
- 本 Skill 的默认且唯一调度通道是 ACP。不要把 Claude Code CLI 作为本 Skill 的常规降级路径。

## 适用场景

当用户提出以下需求时使用本 Skill：

- 巡检某个 GitHub 仓库是否存在 API Key、Token、密码、Webhook URL、私钥等泄露风险。
- 自动把硬编码敏感值改为环境变量读取或安全占位符。
- 自动更新 `.env.example`、`.gitignore` 和 README 的环境变量配置说明。
- 验收通过后自动推送普通修复 commit 到授权 GitHub 仓库。
- 通过飞书发送巡检报告。
- 通过 ACP 让 OpenClaw 调度 Claude Code 完成代码级任务。

## 快速启动

给 OpenClaw/龙虾的一句话启动模板见 `templates/run_skill_prompt.md`。

OpenClaw 必须先读取本 `SKILL.md`，再执行默认任务流。当前已验证飞书交互通道可以正常执行：

```text
/acp doctor
/acp spawn claude --mode persistent --thread off --cwd /srv/openclaw-runner/repos/agentic-ai
/acp steer --session <session-key> <task prompt>
```

后台自动化任务应复用同一套 ACP 能力：要么调用 OpenClaw 内部 ACP service，要么把 ACP 指令交给同一个 slash command router 处理。不要把 `/acp ...` 当 shell 命令执行。

## ACP 调用 Claude Code 规范

OpenClaw 必须通过 ACP 调用 Claude Code。当前已验证的可用方式是飞书/OpenClaw 对话框中的 slash command。

重要：`/acp doctor`、`/acp spawn`、`/acp steer` 不是服务器 shell 命令，不能在 bash、zsh、PowerShell 或 SSH 终端里执行。它们必须作为聊天消息发送给 OpenClaw 的对话/消息路由，由 OpenClaw slash command router 解析执行。后台自动化也必须复用这条 ACP 执行链路，或调用与其等价的 OpenClaw 内部 ACP service。

### 对话通道调用

在飞书/OpenClaw 对话框中，按顺序发送以下聊天消息：

```text
/acp doctor
```

如果返回 `healthy: yes`，继续在同一个对话框中发送以下聊天消息，创建 Claude Code 会话：

```text
/acp spawn claude --mode persistent --thread off --cwd /srv/openclaw-runner/repos/agentic-ai
```

返回示例：

```text
Spawned ACP session agent:claude:acp:258c3125-77df-42ab-90e6-207af58ceef6
```

OpenClaw 必须记录完整 `session-key`：

```text
agent:claude:acp:258c3125-77df-42ab-90e6-207af58ceef6
```

然后继续在对话框中发送以下聊天消息，投递任务：

```text
/acp steer --session <session-key> 读取 /srv/openclaw-runner/tasks/agentic-ai-secret-audit.json，并严格按照 /root/projects/github-secret-auditor-skill/templates/acp_steer_prompt.md 执行。
```

如果验收发现缺失项，继续使用同一个 `session-key` 追加 steer：

```text
/acp steer --session <session-key> 请根据 OpenClaw 验收结果补齐缺失项：<具体缺失项>。完成后重新输出修改文件清单、风险摘要、测试/静态检查结果和 git diff 摘要。
```

### 后台自动化调用

后台任务、Heartbeat 或定时任务不能把 `/acp ...` 写入 shell、subprocess、SSH 命令或脚本里执行。它必须执行以下二选一策略：

- 调用 OpenClaw 内部 ACP service，效果等价于 `/acp doctor`、`/acp spawn`、`/acp steer`。
- 将 ACP 指令作为聊天消息提交给与飞书对话相同的 slash command router，让 router 执行命令并返回结果。

后台调用契约如下：

```text
acp.doctor()
acp.spawn(agent="claude", mode="persistent", thread="off", cwd="/srv/openclaw-runner/repos/agentic-ai")
acp.steer(session="<session-key>", prompt="<acp_steer_prompt>")
```

如果 OpenClaw 当前没有这些内部方法，就应复用 slash command router，而不是让模型自行判断 ACP 不可用。只有 `/acp doctor` 不健康、spawn 失败或 steer 失败时，才返回 `failed`。

### Session 保存

OpenClaw 应把 session 信息保存到当前任务状态：

```json
{
  "runner": "acp",
  "agent": "claude",
  "session_key": "agent:claude:acp:258c3125-77df-42ab-90e6-207af58ceef6",
  "repo_path": "/srv/openclaw-runner/repos/agentic-ai",
  "task_path": "/srv/openclaw-runner/tasks/agentic-ai-secret-audit.json"
}
```

如果任务重试且 session 仍可用，优先复用现有 `session-key`。如果 session 丢失或不可解析，重新 spawn。

## 前置条件

使用本 Skill 前，OpenClaw 应确认：

- OpenClaw 所在服务器已安装 Claude Code，并完成认证或 API 中转配置。
- ACP 后端健康，`/acp doctor` 显示 `healthy: yes`。
- `configuredBackend` 和 `registeredBackend` 均为 `acpx`。
- `/acp spawn claude ...` 能返回 `agent:claude:acp:...`。
- OpenClaw 运行用户具备目标 GitHub 仓库的 clone、commit 和 push 权限。
- 目标仓库已获得用户授权。
- 默认工作根目录为 `/srv/openclaw-runner`。

如果 ACP 不健康或无法 spawn Claude Code，会话状态为 `failed`，并返回具体阻塞原因。不要改用 OpenClaw 自己手工巡检。

## 输入契约

用户至少应提供：

- `repo_url`：GitHub 仓库地址，例如 `https://github.com/lemons101/agentic-ai.git`。
- `branch`：目标分支，默认 `main`。
- `mode`：默认 `audit_fix_push_report`。

如果用户未提供 `repo_path`，默认使用：

```text
/srv/openclaw-runner/repos/<repo-name>
```

默认策略：

```json
{
  "allow_auto_fix": true,
  "allow_push": true,
  "push_strategy": "commit_to_current_branch",
  "runner": "acp"
}
```

## 输出契约

任务完成后，OpenClaw 应返回：

```json
{
  "status": "passed | failed",
  "repo": "lemons101/agentic-ai",
  "runner": "acp",
  "session_key": "agent:claude:acp:...",
  "report_path": "/srv/openclaw-runner/reports/agentic-ai-security-report.md",
  "changed_files": [],
  "risk_summary": "",
  "completed_fixes": [],
  "residual_risks": [],
  "risk_notes": [],
  "pushed": false,
  "commit": "",
  "notified": false,
  "next_action": ""
}
```

状态含义：

- `passed`：Claude Code 已完成代码修复，OpenClaw 验收通过，并已按配置完成 commit、push 和飞书报告。
- `failed`：ACP 调度失败、仓库不可访问、Claude Code 未完成任务、验收不通过或 push 失败。

Git 历史泄露、疑似外部凭证风险或无法自动判定的凭证归属，不改变本 Skill 的自动化完成状态；这些内容写入飞书报告的 `risk_notes`。

## 默认任务流

1. 确认输入：`repo_url`、`repo_path`、`branch`、`mode`、`allow_auto_fix`、`allow_push`。
2. 准备工作目录：`/srv/openclaw-runner/repos`、`/srv/openclaw-runner/tasks`、`/srv/openclaw-runner/reports`。
3. 如果本地没有仓库，clone 到 `repo_path`。
4. 如果本地已有仓库，先执行 `git status --short`；如有未提交修改，停止并报告 `failed`，避免覆盖用户工作。
5. 工作区干净时执行 `git pull --ff-only`。
6. 基于 `templates/openclaw_task.secret_audit.json` 生成任务包到 `/srv/openclaw-runner/tasks/<repo>-secret-audit.json`。
7. 向 OpenClaw 对话/消息路由发送 `/acp doctor`，确认 `healthy: yes`。
8. 向同一对话/消息路由发送 `/acp spawn claude --mode persistent --thread off --cwd <repo_path>`。
9. 记录完整 `session-key`，格式为 `agent:claude:acp:...`。
10. 向同一对话/消息路由发送 `/acp steer --session <session-key> <templates/acp_steer_prompt.md 内容或等价任务>`。
11. 等待 Claude Code 输出修改文件清单、风险摘要、测试/静态检查结果和 Git Diff 摘要。
12. 如验收缺失项，使用同一 `session-key` 追加一轮 `/acp steer` 聊天消息定向修正。
13. OpenClaw 验收 Git Diff 和残余风险。
14. 验收通过后，OpenClaw 执行 `git add`、`git commit`、`git push`。
15. OpenClaw 生成飞书报告；报告不得提交到 GitHub 仓库。
16. 返回最终状态、commit、风险摘要、后续动作和飞书通知结果。

## OpenClaw 与 Claude Code 分工

OpenClaw 负责：

- 拉取或更新目标仓库。
- 生成任务包。
- 创建 Claude Code ACP 会话。
- 多轮 steer Claude Code 完成巡检、修复和补漏。
- 验收 `.gitignore`、README、`.env.example`、Git Diff 和残余风险。
- 验收通过后执行 commit 和 push。
- 发送飞书报告，必要时写入 `/srv/openclaw-runner/reports`。

Claude Code 负责：

- 只在 `repo_path` 内工作。
- 搜索疑似 API Key、Token、密码、Webhook URL、私钥片段、数据库连接串等风险。
- 将硬编码敏感值改成环境变量读取或安全占位符。
- 更新 `.env.example`，只写占位符。
- 更新 `.gitignore`，忽略 `.env`、`.env.*`、`*.pem`、`*.key`，以及经判断不应提交的本地敏感配置文件。
- 更新 README 的环境变量配置说明。
- 输出修改文件清单、风险摘要、测试/静态检查结果和 Git Diff 摘要。
- 不 push、不创建 PR、不输出完整密钥。

## 安全规则

- 只处理用户明确授权的仓库路径。
- 不读取 `.env`、`.env.*`、SSH Key、私钥、Cookie、生产配置和用户个人目录。
- 不在报告中输出完整密钥，只允许输出脱敏片段，例如 `sk-...abcd`。
- 不强推、不清理 Git 历史、不自动合并分支。
- 不把 `security-report.md` 作为仓库文件提交。
- 所有自动修复都必须留下 Git Diff 摘要。
- Claude Code 会话内不得执行 push；push 只由 OpenClaw 在验收后执行。

## 历史泄露风险记录

如果发现疑似真实密钥已进入 Git 历史：

- Claude Code 继续完成当前代码修复，不要因为历史风险停止自动化流程。
- OpenClaw 仍可在验收通过后提交并推送普通修复 commit。
- 验收、commit、push 和飞书报告完成后，最终状态仍可为 `passed`。
- 飞书报告中的 `risk_notes` 必须记录风险类型、疑似文件、脱敏片段和建议动作。
- 不要声称已经完成外部平台凭证处理，除非任务包明确提供了可调用的外部接口。

## 验收标准

OpenClaw 必须检查：

- 仓库 diff 中没有 `security-report.md`。
- `.gitignore` 包含 `.env`、`.env.*`、`*.pem`、`*.key`。
- `.env.example` 只包含占位符，不包含真实密钥。
- README 已说明环境变量配置。
- 当前代码中不再出现明显完整硬编码密钥。
- Claude Code 输出了修改文件清单、风险摘要、测试/静态检查结果和 Git Diff 摘要。
- 若有历史泄露或外部凭证风险，报告中有 `risk_notes`。

验收通过后，OpenClaw 可执行：

```bash
cd /srv/openclaw-runner/repos/agentic-ai
git status --short
git add .gitignore README.md openclaw-infra/configs/.env.example openclaw.json
git commit -m "fix: remediate leaked secret configuration"
git push origin HEAD
```

实际 `git add` 文件清单应以 Git Diff 中的授权修复文件为准，不要添加报告文件或禁止文件。

## 飞书报告

飞书报告必须包含：

- 状态：`passed` 或 `failed`。
- 目标仓库与分支。
- ACP session-key。
- 是否已 push。
- commit hash。
- 修改文件清单。
- 风险摘要。
- 已完成修复。
- 残余风险。
- 风险备注 `risk_notes`。
- 下一步动作。

报告可归档到：

```text
/srv/openclaw-runner/reports/agentic-ai-security-report.md
```

归档报告不得提交到 GitHub 仓库。

## 失败处理

如果 ACP 会话创建失败：

- 向 OpenClaw 对话/消息路由发送 `/acp doctor`。
- 如果 `healthy: no`，返回 `failed`，说明 ACP 后端不可用。
- 如果默认 agent 不是 `claude`，显式使用 `/acp spawn claude ...`。
- 不要改用 OpenClaw 手工巡检。

如果 `/acp steer` 无法解析 session：

- 检查是否使用完整 `agent:claude:acp:...`。
- 不要只使用 label。
- 重新 spawn 并使用新的完整 `session-key`。

如果 Claude Code 报权限不足：

- 检查 `--cwd` 是否指向目标仓库。
- 检查运行用户是否能读写 `repo_path`。
- 检查 ACPX 非交互写入权限。
- 修复权限后重新创建 Claude ACP 会话。
