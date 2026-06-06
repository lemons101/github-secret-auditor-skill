# 第19节 实验手册：OpenClaw 调度 Claude Code 完成 GitHub 安全巡检

> 配套课程：AI 业务流架构师 · 第19节 多 Agent 协作：夜间代码自愈实验室  
> 操作方式：把每个实验中的 Prompt 复制给龙虾执行  
> 预计耗时：40-60 分钟

---

## 实验目标

1. 安装 Claude Code，并确认 OpenClaw 能调用它。
2. 安装 `github-secret-auditor` Skill。
3. 验证 OpenClaw 通过 ACP / ACPX 调度 Claude Code。
4. 完成一次只读试跑，确认 Claude Code 进入授权仓库。
5. 执行一次 GitHub 密钥泄露巡检，并由 OpenClaw 验收、提交、推送和飞书汇报。

---

## 核心链路

```text
用户给出仓库
-> OpenClaw 读取 Skill
-> 准备仓库和任务包
-> sessions_spawn 调度 Claude Code
-> Claude Code 巡检 / 修复 / 输出摘要
-> OpenClaw 验收 Git Diff
-> 必要时 sessions_send 补漏
-> OpenClaw commit / push / 飞书报告
```

默认测试仓库：

```text
https://github.com/lemons101/agentic-ai.git
```

默认路径：

```text
/root/projects/github-secret-auditor-skill
/srv/openclaw-runner/repos/agentic-ai
/srv/openclaw-runner/tasks/agentic-ai-secret-audit.json
/srv/openclaw-runner/reports/agentic-ai-security-report.md
```

安全边界：

- 只操作授权仓库。
- 不读取真实 `.env`、私钥、Cookie、生产配置和用户个人目录。
- Claude Code 不 push；push 只由 OpenClaw 验收后执行。
- 报告只发飞书或写入 `/srv/openclaw-runner/reports`，不要提交进仓库。

---

## 实验零：安装 Claude Code

官方文档：

```text
https://docs.anthropic.com/en/docs/claude-code/getting-started
https://code.claude.com/docs/en/installation
```

官方 npm 安装命令：

```bash
npm install -g @anthropic-ai/claude-code@latest
```

发给龙虾：

```text
请帮我安装并验证 Claude Code。

要求：
1. 检查 node、npm、git：
   node -v
   npm -v
   git --version

2. 检查命令是否可用：
   command -v claude
   claude --version

3. 完成 Claude Code 认证，然后执行：
   claude -p "只回复 OK"

4. 如果 OpenClaw gateway 不是当前 shell 用户启动的，请确认 gateway 运行用户也能访问 claude 命令。

回报：
- node / npm / git 版本
- claude 路径
- claude --version
- claude -p "只回复 OK" 的结果
- OpenClaw gateway 是否需要重启
- 如失败，贴核心报错
```

通过标准：

- `command -v claude` 有明确路径。
- `claude -p "只回复 OK"` 返回 `OK`。
- OpenClaw 运行用户能访问同一个 `claude` 命令。

---

## 实验一：安装 github-secret-auditor Skill

发给龙虾：

```text
请帮我安装并更新第19节实验用的 github-secret-auditor Skill。

目录：
- Skill：/root/projects/github-secret-auditor-skill
- 仓库：/srv/openclaw-runner/repos
- 任务包：/srv/openclaw-runner/tasks
- 报告：/srv/openclaw-runner/reports

请执行：
1. 创建目录：
   mkdir -p /root/projects /srv/openclaw-runner/repos /srv/openclaw-runner/tasks /srv/openclaw-runner/reports

2. 如果 Skill 已存在：
   cd /root/projects/github-secret-auditor-skill && git pull --ff-only

3. 如果 Skill 不存在：
   cd /root/projects && git clone git@github.com:lemons101/github-secret-auditor-skill.git

4. 记录版本：
   cd /root/projects/github-secret-auditor-skill && git rev-parse --short HEAD

5. 检查文件：
   - SKILL.md
   - templates/run_skill_prompt.md
   - templates/acp_steer_prompt.md
   - templates/openclaw_task.secret_audit.json
   - references/preflight_setup.md

暂时不要执行巡检、不要修改目标仓库、不要 push。

回报：
- Skill 安装状态：passed / failed
- Skill 路径
- 当前 commit
- 关键文件是否齐全
- 如失败，贴核心报错
```

通过标准：

- Skill 在 `/root/projects/github-secret-auditor-skill`。
- 关键文件齐全。
- 能记录当前 git commit，方便排查版本差异。

---

## 实验二：验证 ACP 握手

注意：`/acp ...` 是飞书 / OpenClaw 对话框命令，不是 shell 命令。

在飞书 / OpenClaw 对话框发送：

```text
/acp doctor
```

期望：

```text
configuredBackend: acpx
registeredBackend: acpx
healthy: yes
```

继续发送：

```text
/acp spawn claude --mode persistent --thread off --cwd /srv/openclaw-runner/repos/agentic-ai
```

期望返回：

```text
Spawned ACP session agent:claude:acp:...
```

记录完整 session-key：

```text
agent:claude:acp:<uuid>
```

---

## 关键配置

### 允许课堂演示环境写入授权仓库

如果 Claude Code 只能分析、不能修改授权仓库，可临时设置：

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

设置后必须：

1. 重启 OpenClaw gateway。
2. 重新执行 `/acp doctor`。
3. 重新创建 Claude ACP 会话。

旧 session 可能沿用旧权限。

实验结束后建议恢复：

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-reads
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions deny
```

### 开启后台补漏投递

如果要使用 `sessions_send` 补漏，确认：

```bash
openclaw config set tools.sessions.visibility all
openclaw config set tools.agentToAgent.enabled true
```

典型错误：

```text
Session send visibility is restricted
Agent-to-agent messaging is disabled
```

看到这些错误：设置配置、重启 gateway、重新创建 session。

---

## 实验三：准备仓库和任务包

发给龙虾：

```text
请准备第19节安全巡检实验的测试仓库和任务包。

目标仓库：
https://github.com/lemons101/agentic-ai.git

本地路径：
/srv/openclaw-runner/repos/agentic-ai

任务包路径：
/srv/openclaw-runner/tasks/agentic-ai-secret-audit.json

报告路径：
/srv/openclaw-runner/reports/agentic-ai-security-report.md

请执行：
1. 如果本地仓库不存在，clone 到 /srv/openclaw-runner/repos/agentic-ai。
2. 如果本地仓库已存在，先执行 git status --short。
3. 如果工作区不干净，停止并回报 failed，不要 reset，不要覆盖。
4. 如果工作区干净，执行 git pull --ff-only。
5. 基于 Skill 模板生成任务包到 /srv/openclaw-runner/tasks/agentic-ai-secret-audit.json。

任务包至少包含：
- repo_url: https://github.com/lemons101/agentic-ai.git
- repo_path: /srv/openclaw-runner/repos/agentic-ai
- branch: main
- allow_auto_fix: true
- allow_push: true
- report_path: /srv/openclaw-runner/reports/agentic-ai-security-report.md

安全规则：
- 不读取真实 .env、私钥、Cookie、生产配置和用户个人目录。
- 不输出完整密钥。
- 不提交 security-report.md。
- Claude Code 不 push，push 由 OpenClaw 验收后执行。

回报：
- 仓库当前 commit
- git status --short
- 任务包路径
- repo_url / repo_path / report_path
- 是否有阻塞问题
```

换仓库时至少同步修改：

```text
repo_url
repo_path
task_file
report_path
仓库目录名
```

---

## 实验四：只读试跑

先验证链路，不允许修改文件。

发给龙虾：

```text
请通过 OpenClaw Sessions API 调度 Claude Code 做只读试跑。

调用要求：
- runtime="acp"
- agentId="claude"
- mode="run"
- thread=false
- cwd="/srv/openclaw-runner/repos/agentic-ai"
- prompt="请输出 pwd 和 git status --short，不要修改文件。"

等价形态：
sessions_spawn(
  runtime="acp",
  agentId="claude",
  mode="run",
  thread=false,
  cwd="/srv/openclaw-runner/repos/agentic-ai",
  prompt="请输出 pwd 和 git status --short，不要修改文件。"
)

回报：
- status
- childSessionKey
- pwd
- git status --short
- 是否有文件被修改
- 如失败，贴核心报错
```

通过标准：

```text
childSessionKey: agent:claude:acp:...
pwd: /srv/openclaw-runner/repos/agentic-ai
git status --short: 空
```

如果 `pwd` 不对或工作区不干净，不进入最终巡检。

---

## 实验五：验证补漏投递

发给龙虾：

```text
请使用上一轮返回的 childSessionKey 做 sessions_send 只读测试。

要求：
- sessionKey 使用完整 agent:claude:acp:...
- 不修改文件
- 不 push

等价形态：
sessions_send(
  sessionKey="<childSessionKey>",
  prompt="只读测试：请再次输出 pwd 和 git status --short，不要修改文件。"
)

回报：
- sessions_send 是否成功
- pwd
- git status --short
- 是否有文件被修改
- 如失败，贴核心报错
```

注意：`childSessionKey` 是投递目标，不是记忆保证。真正补漏时，prompt 必须带上：

- 上一轮输出
- 当前 Git Diff
- 验收缺失项
- 本轮允许修改范围

不要只写“继续修一下”。

---

## 实验六：执行最终安全巡检

只读试跑和补漏投递通过后，执行最终实验。

发给龙虾：

```text
请使用 /root/projects/github-secret-auditor-skill/SKILL.md 执行一次全自动 GitHub 密钥泄露巡检。

目标仓库：
https://github.com/lemons101/agentic-ai.git

要求：
1. 全程自动化执行，不要让我手动复制 session、手动执行命令、手动拼接 prompt 或手动验收。
2. OpenClaw 自动读取 Skill、准备仓库、生成任务包、通过 ACP Sessions API 调度 Claude Code、验收修复、commit、push，并通过飞书汇报。
3. Claude Code 负责仓库内敏感信息巡检和代码修复；OpenClaw 不要手工替代 Claude Code 修复。
4. 安全巡检先判断仓库是否存在泄露，再按仓库实际结构做最小安全修复。
5. 不要把修复固定成 .env.example / README / .gitignore 三件套。
6. 最终只回复巡检结果、是否修复、是否推送、commit、风险摘要、风险备注和下一步建议。

最终回复格式：

状态：passed / failed
目标仓库：lemons101/agentic-ai
是否调用 Claude Code：yes / no
调用方式：acp / failed
是否已推送到 GitHub：yes / no
commit：<commit hash>
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

---

## 结果判断

### 通过并修复

应看到：

- `状态：passed`
- `是否调用 Claude Code：yes`
- `调用方式：acp`
- 有 commit hash
- 飞书报告包含风险摘要、修改文件、push 状态和下一步建议

同时检查：

- GitHub 上有普通修复 commit。
- commit 中没有 `security-report.md`。
- diff 只包含授权仓库内修复。
- 报告不输出完整密钥。

### 未发现泄露

也可能是成功结果。重点看：

- 说明巡检通过。
- 不强行改文件。
- 不强行 commit。
- 有报告或结果摘要。

### failed

必须返回明确原因，例如：

- ACP runtime 不可用
- GitHub 权限不足
- 工作区不干净
- Claude Code 无法写文件
- `sessions_send` visibility / agent-to-agent 权限不足

---

## 验收标准

完成实验时，应满足：

- Claude Code 安装并验证通过。
- Skill 安装到 `/root/projects/github-secret-auditor-skill`。
- `/acp doctor` 返回 healthy。
- 只读试跑返回正确 `pwd` 和空的 `git status --short`。
- 最终巡检通过 ACP Sessions API 调度 Claude Code。
- Claude Code 只在授权仓库内工作。
- 不读取真实 `.env`、私钥、Cookie、生产配置和用户个人目录。
- Claude Code 不 push；OpenClaw 验收后才 commit / push。
- 飞书报告或 `/srv/openclaw-runner/reports` 有巡检结果。
- 报告未提交进 GitHub 仓库。

---

## 常见问题

- `claude: command not found`：检查 `npm config get prefix`，确认 npm 全局 bin 在 PATH 中。
- 当前终端能运行 `claude`，OpenClaw 不能：多半是 gateway 运行用户或 PATH 不一致。
- `claude -p "只回复 OK"` 失败：检查认证、API Key 或中转配置。
- `/acp doctor` 失败：检查 ACPX 后端配置，并重启 gateway。
- `/acp` 在 shell 中失败：正常，`/acp` 只能在飞书 / OpenClaw 对话框执行。
- `/acp spawn` 失败：检查 `--cwd` 是否存在且为授权仓库。
- Claude Code 只能分析不能写：检查 ACPX 写入权限和文件系统权限。
- 改了 `approve-all` 仍不能写：重启 gateway，重新创建 session。
- `sessions_send` forbidden：检查 `tools.sessions.visibility=all` 和 `tools.agentToAgent.enabled=true`。
- 补漏像忘了上一轮：`sessions_send` prompt 必须显式带上上一轮输出、当前 diff 和缺失项。
- Skill 仍问“是否确认修改”：说明没有读到 Skill 或默认自动化行为未生效。
- 报告被提交进仓库：错误，应从 commit 中移除报告文件。
- 输出完整密钥：安全事故，应撤回输出，只保留脱敏片段。

---

## 实验记录

| # | 发生在哪一步 | 预期行为 | 实际行为 | 解决方法 |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |

---

## 课后练习

### 必做 1：复现只读链路

通过 OpenClaw 调度 Claude Code，输出 `pwd` 和 `git status --short`，确认 Claude Code 进入授权仓库且没有修改文件。

交付：

- OpenClaw 对话截图
- `childSessionKey`
- `git status --short` 为空的截图

### 必做 2：跑一次安全巡检 dry-run

准备测试仓库，放入一个明显假密钥：

```text
DEMO_API_KEY=sk-demo-123456
```

通过 OpenClaw + Claude Code 完成只读巡检或 dry-run 分析，要求识别风险、给出修复建议和预期 diff，但不要 push。

交付：

- 测试仓库链接
- OpenClaw 对话记录
- 风险摘要
- dry-run 修复建议截图

### 选做 3：迁移到轻量代码质量检查

设计一个只读任务，例如：

- 检查 README 是否缺少启动说明
- 检查项目是否缺少 LICENSE
- 检查 `package.json` 是否缺少 test script

交付：

- 任务包 Markdown 或 JSON
- OpenClaw 执行截图
- 检查结果摘要
- 100-200 字复盘
