# 方案：通过 GitHub MCP 支持非开发人员参与 Agent 协作

**状态**：草案，讨论中
**背景**：当前 agent-dev-flow 的所有命令依赖 Claude Code CLI，非开发人员（PM、运营、QA）无法直接参与。本方案探索以 GitHub 原生界面为交互层，让所有角色通过 Issue、PR、Comment 驱动 Agent 工作流。

---

## 核心思路

**非开发人员通过 GitHub 原生界面操作，Agent 通过 GitHub MCP 监听并响应。**

不需要安装 Claude Code，不需要懂命令行。PM 开 Issue，QA 在 PR 里评论，DevOps 勾 Checklist——Agent 在后台完成分析和生成工作，结果以 Comment 形式回到 GitHub。

---

## 模块设计

### 模块一：Gap Registry → GitHub Issues

#### 现状
Gap Registry 是一个 `gap-registry.md` 文件，需要手动编辑，非开发人员难以参与。

#### 改造方案

| GitHub 元素 | 对应含义 |
|------------|---------|
| Issue | 一条 Gap 记录 |
| Label: `gap` | 标识这是一个 Gap |
| Label: `be-reviewed` / `pm-reviewed` / `confirmed` / `closed` | Gap 状态 |
| Comment | BE 技术分析 / PM 产品回复 |
| Issue 关闭 | Gap closed |

**工作流：**

```
PM 在 GitHub 开 Issue，打 label: gap
        ↓
GitHub Actions 检测到新 gap issue
        ↓
触发 Claude Agent（via MCP）读取 Issue 内容
        ↓
Agent 回复 Comment：技术影响分析 + 推荐方案
        ↓  label 自动变更为 be-reviewed
PM 在 Issue 里回复产品决策
        ↓  label 手动改为 pm-reviewed
任一方打 label: confirmed，填写决策内容
        ↓
Agent 监听到 confirmed → 自动创建 ADR（如需要）
        ↓
功能上线后关闭 Issue → status: closed
```

**Issue 模板（`.github/ISSUE_TEMPLATE/gap.yml`）：**

```yaml
name: Gap / 需求对齐
labels: ["gap"]
body:
  - type: textarea
    id: description
    label: 问题描述
    placeholder: 描述产品需求与技术实现之间的分歧
  - type: textarea
    id: impact
    label: 影响范围
    placeholder: 影响哪些功能、哪些接口、哪个 Sprint？
  - type: dropdown
    id: priority
    label: 优先级
    options: [high, medium, low]
```

---

### 模块二：PR / Issue Comment 触发命令

非开发人员在 PR 或 Issue 里发 Comment，格式为 `/command`，Actions 解析后调用 Agent。

**支持的命令：**

| Comment 命令 | 执行角色 | 对应现有命令 |
|-------------|---------|------------|
| `/be-review-gaps` | BE Agent | `roles/backend/commands/be-review-gaps.md` |
| `/pm-review-gaps` | PM Agent | `roles/pm/commands/pm-review-gaps.md` |
| `/review-api-spec` | FE Agent | `roles/frontend/commands/review-api-spec.md` |
| `/gen-test-plan` | QA Agent | `roles/qa/commands/gen-test-plan.md` |
| `/check-conventions` | BE Agent | `roles/backend/commands/check-conventions.md` |
| `/gen-deploy-checklist` | DevOps Agent | `roles/devops/commands/gen-deploy-checklist.md` |

**触发流程：**

```
用户在 PR/Issue 发 Comment: /gen-test-plan
        ↓
GitHub Actions: issue_comment 事件触发
        ↓
解析 Comment，提取命令名
        ↓
校验权限（仅 collaborator 可触发）
        ↓
调用 Claude API，注入对应 command.md 作为 system prompt
        ↓
Agent 读取 PR diff / Issue 内容（via MCP）
        ↓
结果以 Comment 回复到同一 PR/Issue
```

---

### 模块三：Sprint 任务 → GitHub Issues + Project

**现状**：`split-tasks` 输出为本地 markdown 文件，只有开发者能看。

**改造方案**：

Tech Lead 运行 `split-tasks` → Agent 通过 MCP 自动在 GitHub 批量创建 Issue：

```
Issue 标题:  [BE-001] 实现用户登录接口
Labels:      backend, sprint-3
Milestone:   Sprint 3
Body:        任务描述 + 验收条件 + 依赖关系
Assignee:    （留空，开发者自行认领）
```

PM 和非技术角色在 **GitHub Project 看板**实时查看 Sprint 进度，无需任何额外工具。

---

### 模块四：部署 Checklist → Issue Checklist

**现状**：`gen-deploy-checklist` 输出为本地 markdown 文件。

**改造方案**：

DevOps Agent 生成清单后，通过 MCP 创建 Issue：

```markdown
## 部署清单 — v1.2.0 — 2026-03-25

- [ ] 数据库 migration 已执行
- [ ] Nacos 配置已更新
- [ ] 冒烟测试通过（L1 + L2）
- [ ] 回滚方案已准备
- [ ] 值班人员已确认
```

运维人员在 GitHub 网页逐项勾选 → 全部完成后打 label: `ready-to-deploy` → Agent 监听到后触发冒烟测试通知。

---

## 技术架构

```
GitHub 界面（Issue / PR / Comment）
        ↓ webhook
GitHub Actions
        ↓ 解析事件 + 权限校验
Claude API（携带 command.md 作为 system prompt）
        ↓ GitHub MCP
读取 Issues / PR diff / 仓库文件
        ↓
结果写回 GitHub（Comment / 新 Issue / Label 变更）
```

**关键依赖：**
- GitHub Actions：事件触发层
- Claude API：Agent 执行层
- GitHub MCP Server：仓库读写层
- GitHub Issue Templates：标准化输入层

---

## 落地难点与应对

| 难点 | 说明 | 应对 |
|------|------|------|
| MCP 非持续监听 | GitHub MCP 是按需调用，不是 webhook | 用 GitHub Actions 作为事件驱动层，Actions 触发后调用 MCP |
| Comment 触发安全风险 | 任何人都能发 `/command` | Actions 中校验 `github.actor` 是否为 collaborator，非授权人的触发直接忽略 |
| Agent 写权限 | MCP server 需要 repo 写权限 | 用专用 bot 账号的 PAT，最小权限（issues: write, pull-requests: write） |
| 跨仓库场景 | 共享文档仓库 ≠ 代码仓库 | 共享文档仓库部署一套 Actions，代码仓库部署另一套，MCP 分别授权 |

---

## 最小落地路径

**第一步（最小可用）**：Gap Registry 迁移到 GitHub Issues
- 创建 Issue 模板
- 配置 Actions：新 gap issue → BE Agent 自动回复技术分析
- 收益：PM 无需学任何工具，直接在 GitHub 开 Issue

**第二步**：Comment 触发命令
- 配置 Actions：PR/Issue comment → 解析命令 → 调用 Agent
- 先支持 `/be-review-gaps` 和 `/gen-test-plan` 两个高频命令

**第三步**：Sprint 任务 Issue 化
- `split-tasks` 增加 `--create-issues` 参数
- 输出从本地文件变为 GitHub Issues + Project

**第四步**：部署 Checklist Issue 化
- `gen-deploy-checklist` 增加 `--create-issue` 参数

---

## 待讨论问题

1. **仓库结构**：Gap Registry Issues 放在共享文档仓库还是代码仓库？
2. **Actions 触发延迟**：GitHub Actions 冷启动约 30-60 秒，是否可接受？
3. **多项目复用**：这套 Actions 是做成模板仓库，还是每个项目单独维护？
4. **与现有 CLI 命令的关系**：Issue 方案和 Claude Code CLI 并行存在，还是逐步替换？
