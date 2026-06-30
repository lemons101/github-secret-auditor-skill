# GitHub Secret Auditor Skill

让 OpenClaw 调度 Claude Code，对授权 GitHub 仓库完成一次可验收、可提交、可汇报的密钥泄露巡检与安全修复。

这个项目不是一个单独运行的扫描脚本，而是一个面向 OpenClaw 的 Skill。它把“安全巡检”拆成两个 Agent 的协作流程：

- OpenClaw 负责任务编排、仓库准备、ACP 调度、结果验收、commit / push 和飞书汇报。
- Claude Code 负责进入授权代码仓库，完成敏感信息搜索、最小安全修复、本地检查和 diff 摘要输出。

最终用户不需要手动拼接命令，也不需要逐步操作 Claude Code。用户只需要给出目标仓库，OpenClaw 读取本 Skill 后完成后续闭环。

## 解决什么问题

很多 AI 工具可以指出代码片段里的风险，但真实团队需要的是一个完整交付链路：

- 能进入指定 GitHub 仓库，而不是只分析贴出来的片段。
- 能识别 API Key、Token、密码、Webhook URL、私钥片段、数据库连接串等泄露风险。
- 能按项目实际结构做最小修复，而不是机械创建固定模板文件。
- 能把修复结果留在 Git Diff 中，等待 OpenClaw 验收。
- 能由 OpenClaw 在验收通过后提交、推送，并通过飞书交付巡检报告。

这个 Skill 的重点不是“让一个模型更会聊天”，而是把多 Agent 协作写成稳定的任务协议。

## 协作架构

```text
用户给出 GitHub 仓库
-> OpenClaw 读取 github-secret-auditor Skill
-> OpenClaw clone / pull 授权仓库
-> OpenClaw 生成任务包
-> OpenClaw 获取 single-flight lock，确认没有活动 child session
-> OpenClaw 通过 ACP Sessions 调度 Claude Code
-> Claude Code 在仓库内巡检、修复、验证
-> Claude Code 输出修改清单、风险摘要、测试结果和 diff 摘要
-> OpenClaw 验收修复范围与安全边界
-> OpenClaw commit / push
-> OpenClaw 发送飞书巡检报告
-> OpenClaw 关闭/回收 ACP child session 并释放 lock
```

## Agent 分工

| 角色 | 主要职责 | 不做什么 |
| --- | --- | --- |
| OpenClaw | 读 Skill、准备仓库、生成任务包、调度 Claude Code、验收 diff、提交推送、发送报告 | 不跳过 Claude Code 手工替代巡检 |
| Claude Code | 在授权仓库内搜索敏感信息、修改真实代码、补充必要配置模板、输出验证结果 | 不 commit、不 push、不创建 PR、不输出完整密钥 |

这种分工让 OpenClaw 成为“流程负责人”，Claude Code 成为“代码现场执行者”。一个管任务流，一个管代码现场，协作结果才具备可追踪、可验收和可复用的价值。

## 安全边界

本 Skill 默认遵守以下边界：

- 只处理用户明确授权的 GitHub 仓库。
- 不读取 `.env`、真实环境配置、SSH Key、私钥、Cookie、生产配置和用户个人目录。
- 可以读取或更新 `.env.example`、示例配置和公开模板，但不得写入真实密钥。
- 不在仓库内生成或提交 `security-report.md`。
- 不强制创建 `.env.example`、README、`.gitignore` 三件套；只有修复确实需要时才补充。
- 不清理 Git 历史，不替用户轮换外部平台凭证。
- 同一仓库同一分支同一时刻只允许一个活动 ACP child session；每次巡检结束、失败或超时都必须回收会话。
- 疑似历史泄露会写入风险备注，由用户或安全负责人继续处理。

## 交付结果

一次完整运行后，OpenClaw 应交付：

- 巡检状态：`passed` 或 `failed`
- 是否调用 Claude Code
- 是否已推送到 GitHub
- commit hash
- 修改文件清单
- 风险摘要
- 已完成修复
- 残余风险
- 历史泄露或外部凭证风险备注
- 下一步建议

用户最终看到的是巡检后的结果，而不是后台调度细节。

## 项目结构

```text
github-secret-auditor-skill/
├── SKILL.md
├── README.md
├── lesson19-lab.md
├── templates/
│   ├── run_skill_prompt.md
│   └── acp_steer_prompt.md
└── references/
```

核心文件说明：

- `SKILL.md`：Skill 主协议，定义触发场景、默认行为、Agent 分工、安全规则和验收标准。
- `templates/run_skill_prompt.md`：给 OpenClaw / 龙虾的一句话启动模板。
- `templates/acp_steer_prompt.md`：OpenClaw 投递给 Claude Code 的巡检修复任务提示。
- `lesson19-lab.md`：第 19 节课程配套实验手册，用于课堂复现和教学练习。

## 推荐入口

面向 OpenClaw 的最小用户入口：

```text
请使用 github-secret-auditor Skill 全自动巡检并修复 https://github.com/lemons101/agentic-ai.git
```

如果要更换巡检仓库，只需要替换 GitHub 仓库地址。具体操作过程和课堂实验步骤见 `lesson19-lab.md`。

## 课程定位

本项目是《AI 业务流架构师：从 OpenClaw 到 Claude Code 深度实战》第 19 节的最终项目之一。

它展示的不是单点工具能力，而是一种更通用的多 Agent 协作范式：

```text
业务 Agent 负责理解目标、组织流程、验收结果
执行 Agent 负责进入专业现场、完成具体动作、返回证据
```

安全巡检只是本节课选择的落地场景。同样的协作方式，也可以迁移到代码质量巡检、依赖升级、配置治理、测试补全、数据资产巡检等更复杂的工程任务中。
