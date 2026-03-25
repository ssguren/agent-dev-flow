# FAQ — GitHub MCP 协作流程

常见问题与已知未解决问题记录。

---

## 已知问题

### Q: GitHub MCP 拉取不到项目 Issue

**状态**：已解决

**现象**：无论是 Claude Web 还是 Claude Code CLI，配置 GitHub MCP 后无法读取仓库 Issues。

**根本原因**：GitHub MCP 未配置，或配置时提供的 PAT scope 不足。

**解决方案**：

**Claude Web（PM 侧）**：
在 Claude Project 的 Integrations 里添加 GitHub MCP，授权时确保 PAT 包含 `repo` scope（私有仓库）或 `public_repo` + `issues` scope（公开仓库）。

**Claude Code CLI（开发者侧）**：
```bash
claude mcp add --transport stdio github -- npx -y @modelcontextprotocol/server-github
```
配置时提供 GitHub PAT，scope 需包含 `repo`。

**注意**：有代码提交权限（git push）不等于配置了 GitHub MCP。两者是独立的——git 用 SSH key，MCP 用 PAT。

---

## 待补充

> 测试过程中发现的其他问题将持续更新到此文件。
