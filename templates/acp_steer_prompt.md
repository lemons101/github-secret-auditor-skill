读取 `/srv/openclaw-runner/tasks/agentic-ai-secret-audit.json`，并在当前仓库执行 GitHub 密钥泄露巡检与安全修复。

要求：

1. 只在 `/srv/openclaw-runner/repos/agentic-ai` 内工作。
2. 不读取 `.env`、`.env.*`、私钥、SSH Key、Cookie、生产配置和用户个人目录。
3. 搜索 API Key、Token、密码、Webhook URL、私钥片段和数据库连接串。
4. 发现硬编码敏感值后，改为环境变量读取。
5. 补充或更新 `.env.example`，只能写占位符，不要写真实密钥。
6. 更新 README 的环境变量配置说明。
7. 确保 `.gitignore` 忽略 `.env`、`.env.*`、`*.pem`、`*.key`。
8. 不要在仓库内生成或提交 `security-report.md`。巡检报告由 OpenClaw 在验收后通过飞书发送，或写入 `/srv/openclaw-runner/reports/agentic-ai-security-report.md`。
9. 最后输出修改文件清单、风险摘要、测试/静态检查结果和 Git Diff 摘要，供 OpenClaw 生成飞书报告。
10. 不要在 Claude Code 会话内 push，不要创建 PR，不要输出完整密钥。远程推送由 OpenClaw 在验收通过后执行。
11. 如果无法写文件或无法修改文件，必须明确返回具体阻塞原因，例如 ACP permission denied、文件系统权限不足、Claude Code 工具权限不足或当前会话只读。不要只做分析。
12. 最终回复前必须自检：`.gitignore` 是否已补齐、README 是否已说明环境变量、`.env.example` 是否只包含占位符。如果任一项缺失，不要声称任务完成。

如果疑似真实密钥已经进入 Git 历史，请在报告中标记为 `needs_rotation`，提醒用户人工吊销并轮换。
