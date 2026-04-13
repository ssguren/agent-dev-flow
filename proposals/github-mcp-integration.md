# 方案：通过 GitHub MCP 支持非开发人员参与 Agent 协作

**状态**：已验证（主流程 TC-01~TC-04 于 2026-03-25 完成闭环测试）

---

## 背景与真实痛点

当前 agent-dev-flow 的所有命令依赖 Claude Code CLI，只有开发者能参与。

但非技术角色（PM、运营、QA）有自己的现实：

- 她们活在 **Claude Web / App** 里，有聊天习惯和对话记忆，不会切换到 CLI
- 她们**不熟悉 Git**，不应该要求她们改变操作方式
- 她们愿意配合——前提是**不打破她们现有的工作习惯**

所以问题不是"如何让非技术人员用 GitHub"，而是：

> **如何让 Claude Web 用户在她们熟悉的聊天界面里，通过 GitHub MCP 直接参与协作流程？**

GitHub 不是她们的界面，而是她们操作的对象。

---

## 能配置什么

Claude Web/App 支持用户自行添加 MCP Server。GitHub 官方 MCP Server 能力覆盖：

- 读写 Issues（创建、评论、打 label、关闭）
- 读写 PR（评论、review、label）
- 读取仓库文件内容、commit / diff
- 创建/更新文件
- 管理 Project（看板、milestone）

PM 在 Claude Web 配置好 GitHub MCP 后，**对仓库的操作能力几乎等同于一个有写权限的协作者**，只是操作界面换成了对话框。

---

## 核心思路

**非技术人员通过 Claude Web + GitHub MCP 操作，开发者通过 Claude Code CLI 操作，GitHub 仓库是双方共享的单一上下文层。**

```
PM（Claude Web + GitHub MCP）        开发者（Claude Code CLI）
            ↓                                    ↓
   写 Issue / Comment / label            读 Issue / PR diff / 文件
   更新需求文档（via MCP）               读文档生成接口规范
   确认 Gap（打 label）                  读 confirmed gap 写代码
            ↓                                    ↓
                   GitHub 仓库（单一事实来源）
```

两边的 Claude 会话内容不同，但**操作的对象是同一个仓库**。任何一方的动作以 Issue / Comment / 文件变更的形式落地，另一方下次读取时自然可见。

---

## 关键约定：对话是草稿区，GitHub 是决策区

**Claude 对话记录 ≠ 协作记录。**

- PM 的 Claude Web 聊天记录是她个人的工作草稿，不需要放弃
- 但**影响他人的决策必须通过 MCP 写入 GitHub**，才能进入共享上下文
- GitHub Issue / label 的状态就是协作状态，任何人任何时候打开都是最新的

这和现实工作一样：你可以在本地草稿里写十遍，但只有发出去的那条消息才算数。

不需要两边实时同步，只需要约定：
- PM 的产品决策 → 落到 Issue Comment 或 label
- 开发者的技术分析 → 回复到对应 Issue
- 任何跨角色生效的内容 → 必须落到 GitHub

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

### 难点一：非技术人员会话无持久上下文

**问题**：开发者用 Claude Code 有 `CLAUDE.md` 和本地文件保底，PM 的 Claude Web 每次新对话上下文清空，需要重新喂背景。

**应对**：Claude Web Project + system prompt + 触发词

- 为每个项目创建一个 Claude Project，system prompt 写死角色、仓库地址、行为规则
- 约定触发词（如"开始今天的工作"），Claude 自动通过 MCP 拉取当前 Sprint Issues 和待处理 Gap
- PM 只需记住开场白，上下文自动加载

详见 `pm-claude-project-setup.md`。

---

### 难点二：PM 侧 Agent 推理质量

**问题**：PM 的 Agent 没有代码上下文，推理质量是否比开发者侧差？

**结论**：不是问题。

PM 需要的上下文全在文档仓库里——需求文档、API 规范、ADR、Sprint 计划——通过 MCP 完全可读。PM 的 Agent 推理质量不取决于代码，取决于文档质量。

这反过来强化了框架的核心约定：**文档仓库是整个协作体系的地基，文档维护纪律决定两边 Agent 的能力上限。**

---

### 难点三：写回 GitHub 的内容谁来审

**问题**：Agent 替 PM 写了一条 Comment，以 PM 本人账号发出，PM 没仔细看——内容可能有误但已经公开。

**两个子问题：**

- **以谁的名义发**：GitHub MCP 用配置者本人 token，写回的内容是 PM 本人账号，开发者看到的不是 bot 输出，权重正常。
- **有没有确认再发**：靠 system prompt 约束"写入前必须展示草稿等待确认"，PM 回复"确认"或"发"才执行。

本质上是铁律的延伸：**Agent 生成草稿，人确认后发出。** 在 Claude Web 场景下，确认动作是 PM 在对话里回一个字。

---

### 难点四：MCP 授权粒度粗

**问题**：GitHub MCP 是仓库级授权，无法限制 PM 的 Agent 只写 Issues、不改文件。

**应对：分仓库隔离 + system prompt 限制**

- PM 的 MCP **只授权文档仓库**，不授权代码仓库——物理上碰不到代码
- 文档仓库误操作有 git 历史可还原，损失可控
- system prompt 加约束：不允许直接修改任何 `.md` 文件，所有文档变更通过 Issue 或 Comment 提出，由开发者执行

PM 侧 Agent 的写操作窗口收窄到 Issues 和 Comments，风险极小。

---

### 其他技术难点

| 难点 | 说明 | 应对 |
|------|------|------|
| MCP 非持续监听 | GitHub MCP 是按需调用，不是 webhook | 用 GitHub Actions 作为事件驱动层，Actions 触发后调用 MCP |
| Comment 触发安全风险 | 任何人都能发 `/command` | Actions 中校验 `github.actor` 是否为 collaborator，非授权人触发直接忽略 |
| Actions 触发延迟 | GitHub Actions 冷启动约 30-60 秒 | 异步场景可接受；实时需求走 Claude Web 对话，不走 Actions |

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
