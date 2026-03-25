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

### Q: GitHub MCP `list_issues` 对私有仓库返回 Not Found，但 PAT 有权限

**状态**：已验证（workaround 可用）

**现象**：GitHub MCP 已 connected，PAT 配置了 `repo` scope，但调用 `mcp__github__list_issues` 时返回 `Not Found: Resource not found`；同时 `search_repositories` 也搜索不到该私有仓库。

**根本原因**：`@modelcontextprotocol/server-github` 的部分工具（如 `list_issues`、`search_repositories`）在处理私有仓库时存在局限，MCP server 层面的请求可能未正确携带认证或走了不同的 API 路径。

**验证方式**：直接用 PAT 调用 GitHub REST API，可以正常返回数据：

```bash
curl -s -H "Authorization: token <PAT>" \
  "https://api.github.com/repos/<owner>/<repo>/issues?state=all"
```

**Workaround**：在需要读取私有仓库 Issues 时，通过 `Bash` 工具用 `curl` + PAT 直接请求 GitHub API，不依赖 MCP 工具。

**注意**：PAT 存储在 `~/.claude/settings.json` 的 `mcpServers.github.env.GITHUB_PERSONAL_ACCESS_TOKEN` 中，可直接读取后传入 curl。

---

## 待补充

> 测试过程中发现的其他问题将持续更新到此文件。
