# Stage 4 — Review 代码审查阶段

## 目标
让每次 PR Review 都有上下文、有标准、有结论，而不是"看代码感觉不对但说不清楚"。

## 核心问题
传统 Code Review 的失败模式：
- Reviewer 不知道"为什么要这样改"（缺需求上下文）
- Reviewer 只看代码，不看接口规范（发现不了语义错误）
- Author 解释背景靠口头，不可追溯
- Review 意见太主观，缺乏项目级别的一致标准

**Agent 的介入点**：在 Reviewer 读代码之前，先提供上下文摘要 + 规范性检查，让人工 Review 聚焦在"逻辑正确性"和"设计合理性"上，而不是格式和规范问题。

## 参与角色
- **Author**（PR 提交者）：提供上下文，回应 Review 意见
- **Reviewer**（1-2 人）：做业务逻辑和设计判断
- **Agent**：做规范检查、上下文摘要、风险提示

## 输入
- PR diff（代码变更）
- `pr-description.md`（Author 填写）
- `specs/api-specification.md`（接口规范）
- `adr/`（架构决策，Reviewer 需要知道为什么这样设计）
- 项目 `CLAUDE.md`（编码规范）

## 输出
- Review 意见（线上，在 PR 评论中）
- 如有架构级问题，输出新的 ADR 草稿
- Approve / Request Changes / Block

## 工作流

```
Author 提交 PR
    ↓
/review-pr              Agent 做规范检查 + 上下文摘要 + 风险提示，输出 Review 摘要
    ↓
Reviewer 阅读摘要       快速了解背景，聚焦人工判断
    ↓
Reviewer 做 Review      重点检查逻辑正确性 + 设计合理性
    ↓
遇到架构分歧？           /draft-adr → Tech Lead 确认，不在 PR 里无限讨论
    ↓
Approve 或 Request Changes
    ↓
Author 修改 → 重跑 /review-pr → 直到 Approve
    ↓
Merge → 进入 Stage 5（测试）
```

## 命令

| 命令 | 谁运行 | 做什么 |
|------|--------|--------|
| `/review-pr` | Reviewer | Agent 读取 PR diff + 接口规范 + CLAUDE.md，输出：规范问题清单、接口一致性检查、安全风险提示、测试覆盖分析 |
| `/review-pr --context` | Reviewer | 同上，额外输出需求背景摘要（从需求文档和 ADR 提取） |
| `/explain-change <file>` | Reviewer | Agent 解释某个文件的改动意图，从 PR 描述 + git log 提取 |

## Review 清单（人工必查项）

以下是 Agent 无法替代的人工判断：

**业务逻辑**
- [ ] 实现与验收标准一致
- [ ] 边界情况处理是否合理（空值、超长、并发）
- [ ] 错误码和错误信息对用户友好

**设计合理性**
- [ ] 是否引入了不必要的复杂度
- [ ] 命名是否准确表达意图
- [ ] 是否有更简单的实现方式

**副作用**
- [ ] 数据库 schema 变更是否有迁移脚本
- [ ] 是否影响其他模块（检查接口调用方）
- [ ] 配置项变更是否在文档中更新

## Review 意见格式约定

为了减少歧义，Review 意见按严重级别分类：

| 前缀 | 含义 | 是否 Block |
|------|------|-----------|
| `[BLOCK]` | 必须修改，否则不能合并 | ✅ |
| `[SUGGEST]` | 建议修改，但不强制 | ❌ |
| `[QUESTION]` | 不理解这段代码，需要解释 | 解释后决定 |
| `[NITPICK]` | 小问题，Author 自行决定 | ❌ |

## 质量门禁（Merge 前必须满足）

- [ ] 所有 `[BLOCK]` 意见已解决
- [ ] 至少一名非 Author 的 Reviewer Approve
- [ ] Agent `/review-pr` 输出的规范问题清单已清零
- [ ] 如有数据迁移，迁移脚本已包含在 PR 中
