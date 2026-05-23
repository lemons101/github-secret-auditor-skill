读取 `/srv/openclaw-runner/tasks/agentic-ai-secret-audit.json`，并在当前仓库执行 GitHub 密钥泄露巡检与安全修复。

要求：

1. 只在 `/srv/openclaw-runner/repos/agentic-ai` 内工作。
2. 不读取 `.env`、`.env.*`、私钥、SSH Key、Cookie、生产配置和用户个人目录。
3. 搜索 API Key、Token、密码、Webhook URL、私钥片段和数据库连接串。
4. 发现硬编码敏感值后，改为环境变量读取。
5. 补充或更新 `.env.example`，只能写占位符，不要写真实密钥。
6. 更新 README 的环境变量配置说明。
7. 确保 `.gitignore` 忽略 `.env`、`.env.*`、`*.pem`、`*.key`。
8. 生成 `security-report.md`。
9. 最后输出修改文件清单、风险摘要、测试/静态检查结果和 Git Diff 摘要。
10. 不要 push，不要创建 PR，不要输出完整密钥。

如果疑似真实密钥已经进入 Git 历史，请在报告中标记为 `needs_rotation`，提醒用户人工吊销并轮换。
