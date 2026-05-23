---
name: github-secret-auditor
description: 当用户希望让 OpenClaw 通过 ACP 调度 Claude Code，对 GitHub 仓库进行 API Key、Token、密码、私钥、Webhook URL 等敏感信息泄露巡检、自动修复、生成安全报告与 Diff 摘要时使用。
---

# GitHub 密钥泄露巡检 Skill

## 目标

本 Skill 用于让 OpenClaw 调度 Claude Code，对授权 GitHub 仓库执行夜间密钥泄露巡检与安全修复。

OpenClaw 负责调度、任务包、边界与验收；Claude Code 负责进入仓库执行搜索、修改、验证和报告生成。

## 适用场景

当用户提出以下需求时使用本 Skill：

- 巡检某个 GitHub 仓库是否存在 API Key、Token、密码、Webhook URL、私钥等泄露风险。
- 让 Claude Code 自动把硬编码敏感值改成环境变量读取。
- 生成 `security-report.md`、修改文件清单、风险摘要和 Git Diff 摘要。
- 通过 ACP 插件让 OpenClaw 调度 Claude Code 完成代码级任务。

## 前置条件

使用本 Skill 前，OpenClaw 应确认：

- OpenClaw 所在服务器已安装 Claude Code。
- Claude Code 已在 OpenClaw 运行用户环境中完成认证或 API 中转配置。
- ACP 后端健康，`/acp doctor` 应显示 `healthy: yes`。
- `acpx` 后端已启用，`registeredBackend` 应为 `acpx`。
- ACPX 已在课程环境初始化阶段配置为允许 Claude Code 对授权仓库执行必要写入。
- 目标仓库已获得用户授权。
- 默认工作根目录为 `/srv/openclaw-runner`。

如果 `/acp doctor` 不健康，应先停止任务并提示用户修复 ACP/Claude Code 配置，不要进入代码巡检。

### ACPX 写入权限检查

ACP 会话是非交互式运行的，无法在写文件或执行 shell 时弹出人工确认。如果 ACPX 仍使用默认读权限，Claude Code 可能可以完成巡检分析，但在 `Edit`、`Write` 或生成报告时失败。

本 Skill 不负责修改 OpenClaw 的全局 ACPX 权限配置。运行本 Skill 前，应由课程前置部署流程完成 ACPX 写入权限配置，并在任务结束后按安全要求恢复更保守策略。

前置部署参考：需要配置 OpenClaw、Claude Code、ACP 或 ACPX 写入权限时，读取 `references/preflight_setup.md`。执行巡检任务时不要默认加载该文件。

Skill 只做检查和提示：

- 如果 Claude Code 只能巡检但不能落盘，应返回 `failed` 或 `needs_review`。
- 提醒用户回到“环境初始化流程”检查 ACPX 写入权限。
- 权限调整后必须重启 OpenClaw gateway，并重新创建 Claude ACP 会话。
- 不要在 Skill 执行过程中自行放宽全局权限。

## 输入契约

用户至少应提供：

- `repo_url`：GitHub 仓库地址，例如 `https://github.com/lemons101/agentic-ai.git`。
- `branch`：目标分支，默认 `main`。
- `mode`：默认 `audit_and_fix`。

如果用户未提供 `repo_path`，默认使用：

```text
/srv/openclaw-runner/repos/<repo-name>
```

默认禁止自动推送：

```json
{
  "allow_push": false,
  "allow_auto_fix": true
}
```

## 输出契约

任务完成后，OpenClaw 应给用户返回：

```json
{
  "status": "passed | needs_review | failed",
  "repo": "lemons101/agentic-ai",
  "session_key": "agent:claude:acp:...",
  "report_path": "/srv/openclaw-runner/repos/agentic-ai/security-report.md",
  "changed_files": [],
  "risk_summary": "",
  "residual_risks": [],
  "next_action": ""
}
```

状态含义：

- `passed`：未发现严重残余风险，或风险已在本地修复。
- `needs_review`：发现疑似真实密钥、Git 历史泄露、无法自动判断的凭证或需要人工轮换的风险。
- `failed`：ACP 调用失败、仓库不可访问、Claude Code 未完成任务或验收不通过。

## 分工

OpenClaw 负责：

- 确认目标仓库、分支、工作目录和授权范围。
- 拉取或更新目标仓库。
- 生成 `openclaw_task.json` 任务包。
- 通过 `/acp spawn claude` 创建 Claude Code ACP 会话。
- 通过 `/acp steer --session <session-key>` 投递巡检任务。
- 验收 `security-report.md`、Git Diff、修改文件清单和残余风险。

Claude Code 负责：

- 在指定仓库内搜索敏感信息。
- 定位硬编码 API Key、Token、密码、Webhook URL、私钥片段、数据库连接串等风险。
- 将硬编码敏感值替换为环境变量读取。
- 补充或更新 `.env.example`、`.gitignore` 和 README 配置说明。
- 生成 `security-report.md`。
- 返回风险摘要、修改文件清单、测试/静态检查结果和 Git Diff 摘要。

## 安全规则

- 只处理用户明确授权的仓库路径。
- 不允许读取 `.env`、`.env.*`、SSH Key、私钥、Cookie、生产配置和用户个人目录。
- 不允许在报告里输出完整密钥；只能输出脱敏片段，例如 `sk-...abcd`。
- 不允许自动 push、创建 PR 或合并代码，除非用户明确授权。
- 如果疑似真实密钥已经进入 Git 历史，必须提示用户人工吊销并轮换密钥。
- 所有自动修复都必须留下 Git Diff 和报告。

## 默认任务流

1. 确认输入：`repo_url`、`repo_path`、`branch`、`mode`、`allow_auto_fix`、`allow_push`。
2. 如果本地没有仓库，clone 到 `/srv/openclaw-runner/repos/<repo-name>`。
3. 如果本地已有仓库，执行 `git status` 和 `git pull` 前先确认无未保存修改。
4. 基于 `templates/openclaw_task.secret_audit.json` 生成具体任务包。
5. 使用 ACP 创建 Claude Code 会话。
6. 记录返回的 `session-key`。
7. 使用 ACP 将巡检任务投递给 Claude Code。
8. 等待 Claude Code 完成巡检、修复和报告。
9. 验收输出。
10. 返回 OpenClaw 总结：`status`、`changed_files`、`risk_summary`、`residual_risks`、`next_action`。

## OpenClaw 如何调度 Claude Code

OpenClaw 不直接调用 Claude Code 的内部 API，而是通过 ACP 插件把 Claude Code 启动成一个外部 Coding Agent 会话。

调度链路是：

```text
OpenClaw
  -> /acp spawn claude
  -> acpx backend
  -> Claude Code ACP session
  -> /acp steer --session <session-key>
  -> Claude Code 在指定 repo_path 内执行巡检和修复
```

### 1. 准备仓库

如果服务器还没有目标仓库，先准备本地工作目录：

```bash
mkdir -p /srv/openclaw-runner/repos /srv/openclaw-runner/tasks /srv/openclaw-runner/reports
cd /srv/openclaw-runner/repos
git clone https://github.com/lemons101/agentic-ai.git
```

如果仓库已经存在：

```bash
cd /srv/openclaw-runner/repos/agentic-ai
git status
```

只有在 `git status --short` 没有输出时，才允许执行：

```bash
git pull
```

如果仓库存在未保存修改，OpenClaw 应停止自动拉取，并提示用户先处理本地变更，避免覆盖人工工作。

### 2. 生成任务包

OpenClaw 应将任务包写入：

```text
/srv/openclaw-runner/tasks/agentic-ai-secret-audit.json
```

任务包可以从 `templates/openclaw_task.secret_audit.json` 复制，并根据真实仓库路径和分支更新字段。

### 3. 创建 Claude Code ACP 会话

在 OpenClaw 中执行：

```text
/acp spawn claude --mode persistent --thread off --cwd /srv/openclaw-runner/repos/agentic-ai
```

OpenClaw 会返回类似：

```text
agent:claude:acp:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

这就是 `session-key`。飞书渠道通常不支持 `--bind here`，所以后续必须显式使用这个 `session-key`。

### 3.1 如何获取和保存 session-key

`session-key` 不需要提前写入环境变量，也不是 API Key。它是 OpenClaw 执行 `/acp spawn claude` 后，由 ACP runtime 返回的会话标识。

获取方式：

1. OpenClaw 执行 `/acp spawn claude ...`。
2. 从返回文本中提取 `agent:claude:acp:...` 这一整段。
3. 将它作为后续 `/acp steer --session <session-key>` 的目标。

示例返回：

```text
Spawned ACP session agent:claude:acp:bdd56b3d-92e9-4762-b39c-c9d18c348db7
```

此时应记录：

```text
session-key = agent:claude:acp:bdd56b3d-92e9-4762-b39c-c9d18c348db7
```

在课程演示中，可以由学员手动复制这个 `session-key`。在自动化任务中，OpenClaw 应把它保存到当前任务上下文或任务状态中，例如：

```json
{
  "agent": "claude",
  "session_key": "agent:claude:acp:bdd56b3d-92e9-4762-b39c-c9d18c348db7",
  "repo_path": "/srv/openclaw-runner/repos/agentic-ai"
}
```

如果 `session-key` 丢失：

- 优先尝试 `/acp sessions` 查看现有会话。
- 如果当前渠道不支持会话列表，重新执行 `/acp spawn claude ...` 创建新会话。
- 不要把 `session-key` 当成长期凭证保存；它只是 ACP runtime 的临时会话目标。

### 4. 投递任务给 Claude Code

使用 `/acp steer` 将任务交给 Claude Code：

```text
/acp steer --session <session-key> 读取 /srv/openclaw-runner/tasks/agentic-ai-secret-audit.json，并在当前仓库执行 GitHub 密钥泄露巡检与安全修复。只在当前仓库内工作，不读取 .env、私钥、SSH Key、Cookie、生产配置和用户个人目录。发现硬编码敏感值后，改为环境变量读取，补充或更新 .env.example、README、.gitignore，生成 security-report.md。最后输出修改文件清单、风险摘要、测试/静态检查结果和 git diff 摘要。不要 push，不要创建 PR，不要输出完整密钥。如果无法写文件或无法生成报告，必须明确返回权限错误类型，不要只做分析。
```

### 5. Claude Code 的执行要求

Claude Code 收到任务后，应在 ACP 会话的 `cwd` 中工作，也就是：

```text
/srv/openclaw-runner/repos/agentic-ai
```

它需要完成：

- 搜索疑似泄露的 API Key、Token、密码、Webhook URL、私钥片段、数据库连接串。
- 修复硬编码敏感值。
- 更新 `.env.example`、`.gitignore`、README。
- 生成 `security-report.md`。
- 输出 Git Diff 摘要。

### 6. OpenClaw 验收结果

OpenClaw 收到 Claude Code 返回后，检查：

- `security-report.md` 是否存在。
- `.env.example` 是否只包含占位符。
- `.gitignore` 是否忽略 `.env`、`*.pem`、`*.key`。
- README 是否说明环境变量配置。
- `git diff` 是否只包含授权范围内的修改。

如果疑似真实密钥已经进入 Git 历史，OpenClaw 必须提醒用户人工吊销并轮换密钥。

### 7. 夜间自动化

夜间任务中，Heartbeat 只需要唤醒 OpenClaw 并给出目标仓库。OpenClaw 按本 Skill 自动执行：

```text
生成任务包 -> /acp spawn claude -> /acp steer 投递任务 -> 验收报告和 Diff
```

课堂 MVP 不自动 push；生产增强版可在人工审批后再创建分支和 PR。

## 失败处理

如果 ACP 会话创建失败：

- 先执行 `/acp doctor`。
- 如果 `healthy: no`，停止任务并报告 ACP 后端不可用。
- 如果默认 agent 不是 `claude`，显式使用 `/acp spawn claude ...`。

如果 `/acp steer` 无法解析 session：

- 检查是否使用了完整 `agent:claude:acp:...`。
- 不要只使用 `--label`，部分渠道不能用 label 定位会话。
- 重新 spawn 并使用新的完整 `session-key`。

如果 Claude Code 报权限不足：

- 检查 `--cwd` 是否指向目标仓库。
- 检查 OpenClaw/Claude Code 运行用户是否能读写 `/srv/openclaw-runner/repos/<repo-name>`。
- 提醒用户检查课程前置环境中的 ACPX 写入权限配置。
- 权限调整后，必须重启 gateway，并重新创建 Claude ACP 会话。
- 不要放宽到读取 `.env`、私钥或用户个人目录。

如果发现真实密钥：

- 可以修复代码引用方式，但不能宣称密钥已经安全。
- 必须将最终状态设为 `needs_review`。
- 明确提醒用户到对应平台吊销并轮换密钥。

## 任务包字段

任务包建议包含：

- `goal`：本次巡检目标。
- `repo_url`：GitHub 仓库地址。
- `repo_path`：服务器本地仓库路径。
- `branch`：目标分支。
- `scan_scope`：扫描文件范围。
- `forbidden_files`：禁止读取的文件。
- `risk_types`：风险类型。
- `fix_policy`：修复策略。
- `acceptance_criteria`：验收标准。
- `report_path`：报告输出路径。

## 推荐 ACP 投递 Prompt

```text
读取任务包：<task_path>，并在当前仓库执行 GitHub 密钥泄露巡检。

要求：
1. 只在任务包指定的 repo_path 内工作。
2. 不读取 .env、私钥、SSH Key、Cookie、生产配置和用户个人目录。
3. 搜索 API Key、Token、密码、Webhook URL、私钥片段和数据库连接串。
4. 发现硬编码敏感值后，改为环境变量读取。
5. 补充或更新 .env.example、README 和 .gitignore。
6. 生成 security-report.md。
7. 最后输出修改文件清单、风险摘要、测试/静态检查结果和 git diff 摘要。
8. 不要 push，不要创建 PR，不要输出完整密钥。
9. 如果无法写文件或无法生成 `security-report.md`，必须明确返回阻塞原因，不要只做分析。
```

## 验收口径

任务完成后，OpenClaw 应检查：

- 是否有 `security-report.md`。
- 是否存在未脱敏的 API Key、Token、密码、私钥等明显泄露。
- 是否新增或更新 `.env.example`。
- 是否更新 `.gitignore`。
- 是否说明真实密钥需要人工轮换。
- 是否有未授权文件修改。

如果存在真实密钥泄露风险，最终状态应为 `needs_review`，并提醒用户轮换密钥，而不是直接标记为完全修复。
