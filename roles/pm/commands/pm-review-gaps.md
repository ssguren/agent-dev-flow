# /pm-review-gaps

PM Agent 读取 BE 已分析的 Gap，将技术分析转化为产品语言，回复产品决策方向，通过 GitHub MCP 将结果写回对应 Issue。

---

## Step 1：拉取待回复的 Gap

通过 GitHub MCP 拉取以下 Issues：

- label 包含 `gap` 且包含 `be-reviewed`（BE 已完成分析，等待 PM 回复）

如果没有符合条件的 Issues，输出"当前没有待回复的 Gap"，结束执行。

---

## Step 2：加载产品上下文

对每一个 Issue，读取以下文档作为回复依据：

1. `be-docs/specs/` 下与该 Gap 相关的需求文档
2. `be-docs/adr/` 下 Issue 中提及的 ADR
3. Issue 描述中引用的原始需求来源文档

---

## Step 3：逐条回复，写回 Issue

对每一个 Gap Issue，在 Issue 下回复 Comment，格式如下：

```
## PM 产品回复

**产品优先级**
<这个功能当前 Sprint 必须有，还是可以推迟？说明原因>

**约束说明**
<PM 可以澄清的业务约束，如排期、客户承诺、合规要求>

**倾向方案**
<基于 BE 提供的选项，PM 倾向哪个方向？说明产品侧的考量>

**还需确认的问题**
<如果 PM 无法独立决策，需要向谁确认什么？>

---
> 以上为 PM 产品回复，最终决策由人工确认。
```

**语言要求**：
- 用产品语言，避免技术术语
- 如果 BE 的方案涉及权衡，用产品视角明确说清楚（例如："方案 A 用户体验更好但开发周期长两周；方案 B 可以按时交付但功能有限制"）
- 对优先级直接表态，不模糊

Comment 写入前，展示草稿并等待 PM 确认后再提交。

---

## Step 4：更新 label

Comment 写入成功后，通过 MCP 更新 Issue label：

- 移除 `be-reviewed`，添加 `pm-reviewed`

**不得将 label 改为 `confirmed` 或 `closed`，这两个状态只能由人工操作。**

---

## Step 5：输出摘要

```
PM Review 完成
==============
本次回复 N 个 Gap：

已写入回复（pm-reviewed）：
- #<issue-number> <issue-title>
  需人工确认的核心问题：<一句话>

等待 BE 分析（尚未收到技术分析）：
- #<issue-number> <issue-title>

双方已完成，等待人工拍板：
- #<issue-number> <issue-title>
  建议决策方向：<一句话>

下一步：由相关负责人对"双方已完成"的 Gap 逐条确认，打上 confirmed label。
```

---

## 禁止事项

- 不写最终决策，决策由人工确认
- 不将 label 改为 `confirmed` 或 `closed`
- 不修改任何 .md 文件
