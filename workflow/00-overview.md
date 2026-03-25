# Agent Dev Flow — 全流程概览

## 流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          AGENT DEV FLOW                                  │
│                  "Agent 分析，人决策，文档传递上下文"                      │
└─────────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │  01 需求     │───▶│  02 设计     │───▶│  03 开发     │───▶│  04 Review  │
  │ Requirements│    │   Design    │    │ Development │    │   Review    │
  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
         │                  │                  │                  │
  PM主导  │           TL主导  │          Dev主导  │         人工判断  │
         │                  │                  │                  │
  /pr-changed         /draft-adr         /task-start        /review-pr
  /be-review-gaps     /draft-api-spec    /write-tests       [BLOCK]/[SUGGEST]
  /pm-review-gaps     /split-tasks       /write-pr-desc     /draft-adr
  /confirm-gap        /review-api-spec   /check-conventions
         │                  │                  │                  │
         ▼                  ▼                  ▼                  ▼
  gap-registry.md      ADR + API Spec    PR + Tests         Merge or Rework
         │
         └──────────────────────────────────────────────────────────────┐
                                                                        │
  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
  │  07 复盘     │◀───│  06 部署     │◀───│  05 测试     │◀──────────────┘
  │  Postmortem │    │ Deployment  │    │   Testing   │
  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
         │                  │                  │
  全员参与│           DevOps主导│          QA主导  │
         │                  │                  │
  /gen-sprint-retro    /gen-deploy-         /gen-test-plan
  /gen-incident-pm     checklist            /gen-boundary-cases
  /update-from-retro   /gen-rollback-plan   /gen-permission-matrix
                       /smoke-test-list     /regression-scope
                       /check-config-diff   /format-bug
         │
         └──────────────▶ Gap Registry（预防性条目）
                          CLAUDE.md（规范更新）
                          ▶ 反馈回 Stage 1（下一个 Sprint）
```

## 各阶段一览

| Stage | 主导角色 | 核心输出 | Agent 命令 | 质量门禁 |
|-------|---------|---------|-----------|---------|
| 01 需求 | PM | gap-registry.md (confirmed) | `/pr-changed` `/be-review-gaps` `/pm-review-gaps` `/confirm-gap` | 所有 high Gap 已 confirmed |
| 02 设计 | Tech Lead | ADR + API Spec + 任务列表 | `/draft-adr` `/draft-api-spec` `/split-tasks` | 所有任务无悬空依赖 |
| 03 开发 | Dev | PR (code + tests) | `/task-start` `/write-tests` `/write-pr-desc` | 本地测试通过，无硬编码 |
| 04 Review | 人工 | Approved PR | `/review-pr` | 所有 [BLOCK] 已解决 |
| 05 测试 | QA | test-plan + test-results | `/gen-test-plan` `/gen-boundary-cases` `/gen-permission-matrix` | P0/P1 Bug 清零 |
| 06 部署 | DevOps | deploy-log | `/gen-deploy-checklist` `/gen-rollback-plan` | 冒烟测试通过 |
| 07 复盘 | Tech Lead | sprint-retro + Gap 反馈 | `/gen-sprint-retro` `/update-from-retro` | 至少一条预防措施写入上游 |

## 两个核心文档贯穿全流程

```
gap-registry.md          ← 需求阶段生成，贯穿到复盘阶段
    需求 Gap 追踪            每个 Sprint 结束时 closed 条目归档

CLAUDE.md（各项目）       ← 复盘阶段更新，下一 Sprint 生效
    编码规范 / Agent 规范     每次发现问题都反馈到这里
```

## 角色 × 命令矩阵

| 命令 | PM | Tech Lead | Backend | Frontend | QA | DevOps |
|------|:--:|:---------:|:-------:|:--------:|:--:|:------:|
| `/pr-changed` | | | ✅ | | | |
| `/be-review-gaps` | | | ✅ | | | |
| `/pm-review-gaps` | ✅ | | | | | |
| `/confirm-gap` | ✅ | ✅ | | | | |
| `/draft-adr` | | ✅ | ✅ | ✅ | | |
| `/draft-api-spec` | | | ✅ | | | |
| `/review-api-spec` | | | | ✅ | | |
| `/split-tasks` | | ✅ | | | | |
| `/task-start` | | | ✅ | ✅ | | |
| `/write-tests` | | | ✅ | ✅ | | |
| `/write-pr-desc` | | | ✅ | ✅ | | |
| `/review-pr` | | ✅ | ✅ | ✅ | | |
| `/gen-test-plan` | | | | | ✅ | |
| `/gen-boundary-cases` | | | | | ✅ | |
| `/gen-permission-matrix` | | | | | ✅ | |
| `/gen-deploy-checklist` | | | | | | ✅ |
| `/gen-rollback-plan` | | | | | | ✅ |
| `/gen-sprint-retro` | | ✅ | | | | |
