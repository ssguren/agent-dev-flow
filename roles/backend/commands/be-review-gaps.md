# /be-review-gaps

后端 Agent 分析所有待处理的 Gap，填写技术分析，通过 GitHub MCP 将结果写回对应 Issue。

---

## Step 1：拉取待处理 Gap

通过 GitHub MCP 拉取以下 Issues：

- label 包含 `gap` 且包含 `raised`（新 Gap，待 BE 首次分析）
- label 包含 `gap` 且包含 `pm-reviewed`（PM 已回复，BE 需补充意见）

如果没有符合条件的 Issues，输出"当前没有待处理的 Gap"，结束执行。

---

## Step 2：加载技术上下文

对每一个 Issue，读取以下文档作为分析依据：

1. `be-docs/03-backend-architecture.md` — 后端架构参考
2. `be-docs/04-api-specification.md` — 当前接口规范
3. `be-docs/adr/` 下与该 Gap 相关的 ADR
4. 如 Gap 涉及需求背景，读取 `be-docs/specs/` 下对应文档

---

## Step 3：逐条分析，写回 Issue

对每一个 Gap Issue，在 Issue 下回复 Comment，格式如下：

```
## BE 技术分析

**技术影响**
<这个决策对后端意味着什么？需要新增实体、新服务、还是影响排期？>

**可选方案**
<如果有多种实现路径，简要列出各自的权衡>

**推荐方案**
<BE 的倾向方向，说明原因>

**依赖项**
<BE 继续推进前，需要哪些信息或决策？>

**工作量估算**
<粗略估算：小时 / 天>

---
> 以上为 BE 技术分析，决策由人工确认。
```

Comment 写入前，展示草稿并等待开发者确认后再提交。

---

## Step 4：更新 label

Comment 写入成功后，通过 MCP 更新 Issue label：

- 原 label 为 `raised` → 移除 `raised`，添加 `be-reviewed`
- 原 label 为 `pm-reviewed` → 保持 `pm-reviewed`，同时添加 `be-reviewed`

**不得将 label 改为 `confirmed` 或 `closed`，这两个状态只能由人工操作。**

---

## Step 5：输出摘要

```
BE Review 完成
==============
本次分析 N 个 Gap：

已写入分析（be-reviewed）：
- #<issue-number> <issue-title>
- ...

等待 PM 回复（pm-reviewed，BE 已补充意见）：
- #<issue-number> <issue-title>

等待人工确认（双方分析齐全）：
- #<issue-number> <issue-title>

下一步：PM 查看 be-reviewed Issues 并回复产品决策。
```

---

## 禁止事项

- 不写决策结论，决策由人工确认
- 不将 label 改为 `confirmed` 或 `closed`
- 技术分析语言保持 PM 可读，避免过度技术化
