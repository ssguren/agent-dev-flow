# Stage 1 — Requirements 需求阶段

## 目标
把"产品想法"转化为"有技术上下文的结构化需求"，同时把"技术约束"反馈给"产品决策"。

## 参与角色
- **PM**（主导）：写需求，做决策
- **Tech Lead / BE**（支持）：提出技术 Gap，评估可行性

## 输入
- 产品想法、用户故事、竞品分析
- 上一个 Sprint 的复盘报告（`postmortem/`）

## 输出
- `requirements/<sprint>/product-requirements.md` — 结构化需求文档
- `gap/gap-registry.md` — 所有 PM-BE 未对齐项
- 关键 GAP 确认后 → 触发 ADR（转 Stage 2）

## 工作流

```
PM 写需求
    ↓
/pr-changed          BE Agent 识别技术 Gap，写入 gap-registry.md
    ↓
/be-review-gaps      BE Agent 填写技术分析 → status: be_reviewed
    ↓
/pm-review-gaps      PM Agent 翻译技术分析 → status: pm_reviewed
    ↓
人工对齐会议         双方逐条确认 → /confirm-gap → status: confirmed
    ↓
影响架构的 Gap      → 触发 Stage 2（设计）
```

## 命令

| 命令 | 谁运行 | 做什么 |
|------|--------|--------|
| `/pr-changed` | BE | 需求文档变更后，自动识别新 Gap |
| `/be-review-gaps` | BE | Agent 填写技术影响分析 |
| `/pm-review-gaps` | PM | Agent 翻译为产品语言 |
| `/confirm-gap <ID> "决策"` | PM or BE（人工） | 记录正式决策 |

## 需求文档规范

每条需求至少包含：
- **用户故事**：As a [role], I want [action], so that [value]
- **验收标准**：明确的、可测试的条件（Acceptance Criteria）
- **排除范围**：明确不做什么（Out of Scope）
- **依赖项**：依赖哪些外部系统或其他功能

## 质量门禁（进入下一阶段前必须满足）

- [ ] 所有 `priority: high` 的 Gap 已 `confirmed`
- [ ] 每条需求有明确的验收标准
- [ ] Sprint 范围已锁定（无 `raised` 状态的 high priority 未解决项）
