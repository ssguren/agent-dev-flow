# Agent Dev Flow

[English](README.md)

基于角色的 Claude Code slash command 框架，为软件开发全流程提供结构化的人机协作方案。Agent 负责生成，人负责确认。

---

## 背景

软件开发至少涉及六个角色，跨越七个交接节点——需求 → 设计 → 开发 → Review → 测试 → 部署 → 复盘。每一次交接都在泄漏上下文、决策和责任。

三个根因：

1. **翻译成本** — PM 用产品语言写需求，工程师用技术语言做设计，中间没有结构化的桥梁。
2. **上下文断裂** — 架构决策停留在聊天记录或口头约定，没有可靠的单一来源。
3. **决策无存档** — 六个月后再问"为什么这样设计"，没有人能给出可靠的答案。

本框架为每个角色提供专属的 Agent 人设和命令，系统性地解决上述问题。

---

## 工作原理

每个角色在 `roles/` 下有独立目录，包含：

- `profile.md` — Agent 人设，定义职责、行为准则和边界
- `commands/` — 该角色可执行的 slash command 定义（`.md` 文件）

文档流即上下文链：

```
需求文档（PM）
    ↓
Gap Registry + ADR（Tech Lead）
    ↓
接口规范 + 实现计划（Backend / Frontend）
    ↓
PR Description（Developer）
    ↓
测试计划（QA）
    ↓
部署清单（DevOps）
    ↓
复盘报告（全员）
```

---

## 角色与命令

| 角色 | 命令 |
|------|------|
| **PM** | `pr-changed`, `pm-review-gaps` |
| **Tech Lead** | `split-tasks`, `gen-sprint-retro`, `update-from-retro` |
| **Backend** | `draft-api-spec`, `task-start`, `write-tests`, `check-conventions`, `be-review-gaps` |
| **Frontend** | `review-api-spec`, `task-start` |
| **QA** | `gen-test-plan`, `gen-boundary-cases`, `gen-permission-matrix`, `regression-scope`, `format-bug` |
| **DevOps** | `gen-deploy-checklist`, `gen-rollback-plan`, `smoke-test-checklist`, `check-config-diff` |
| **共享（多角色通用）** | `review-pr`, `draft-adr`, `write-pr-desc`, `confirm-gap`, `task-start` |

---

## 使用方式

### 方式 A — 复制命令到你的项目

将 `roles/<role>/commands/*.md` 复制到项目的 `.claude/commands/` 目录，Claude Code 会自动识别为 slash command。

```bash
# 示例：为项目添加 DevOps 命令
cp agent-dev-flow/roles/devops/commands/*.md your-project/.claude/commands/
```

### 方式 B — 作为模板参考

在启动角色专属 Claude Code 会话时，将 `roles/<role>/profile.md` 作为系统提示上下文，然后运行该角色 `commands/` 下的命令。

### 方式 C — 使用 Gap Registry 模式

适用于 PM 与后端之间的需求对齐：

1. PM 更新需求文档 → 运行 `/pr-changed` → 新 Gap 状态为 `raised`
2. BE Agent 运行 `/be-review-gaps` → 填写技术分析 → 状态变为 `be_reviewed`
3. PM Agent 运行 `/pm-review-gaps` → 填写产品回复 → 状态变为 `pm_reviewed`
4. 人工运行 `/confirm-gap <ID> "决策内容"` → 状态变为 `confirmed`
5. 功能上线后 → 手动改为 `closed`

---

## 核心规则

- **Agent 只生成，人来确认。** 任何影响他人的决策，必须有人工 sign-off 才能生效。
- **代码中禁止中文。** 注释、`@DisplayName` 和文档文件除外。
- **文档流是上下文链。** 每个 Agent 在生成下游产出前，必须先读取上游阶段的输出文档。
- **分类不确定时，选择保守选项。** （例如：配置差异不确定时标记为 SUSPICIOUS 而非 EXPECTED）

---

## 目录结构

```
agent-dev-flow/
├── README.md               # 英文版说明
├── README.zh.md            # 中文版说明（本文件）
├── CLAUDE.md               # 所有 Agent 的铁律
├── background.md           # 完整设计思路
├── workflow/
│   ├── 00-overview.md      # 全流程图 + 角色×命令矩阵
│   ├── 01-requirements.md
│   ├── 02-design.md
│   ├── 03-development.md
│   ├── 04-review.md
│   ├── 05-testing.md
│   ├── 06-deployment.md
│   └── 07-postmortem.md
└── roles/
    ├── shared/commands/    # 跨角色通用命令
    ├── pm/
    ├── tech-lead/
    ├── backend/
    ├── frontend/
    ├── qa/
    └── devops/
```
