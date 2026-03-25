# FAQ — GitHub MCP 协作流程

常见问题与已知未解决问题记录。

---

## 已知问题

### Q: Claude Web 配置 GitHub MCP 后，贡献者 token 拉取不到项目 Issue

**状态**：未解决，待验证

**现象**：PM 在 Claude Web 配置 GitHub MCP 时，使用贡献者身份的 token，无法正常读取仓库 Issues。

**可能原因**：
- PAT（Personal Access Token）缺少必要的 scope：
  - 私有仓库需要 `repo` scope
  - 公开仓库需要 `public_repo` + `issues` scope
- Claude Web OAuth 授权时默认 scope 不包含 Issues 读写权限

**待验证方案**：
1. 检查 PAT scope，确认包含 `repo`（私有）或 `issues`（公开）
2. 通过 Claude Web OAuth 重新授权，手动扩展授权范围

**后续**：验证通过后更新此条目，补充解决方案。

---

## 待补充

> 测试过程中发现的其他问题将持续更新到此文件。
