# 测试计划 — GitHub MCP 协作流程验证

**版本**：v0.1
**分支**：`feature/github-mcp-integration`
**目标**：验证 PM（Claude Web）与开发者（Claude Code CLI）通过 GitHub Issues 完成一次完整的 Gap 协作闭环

---

## 测试背景

本次测试基于 `proposals/github-mcp-integration.md` 中设计的协作方案，验证以下核心假设：

1. PM 在 Claude Web 配置 GitHub MCP 后，可以通过对话操作 GitHub Issues，不需要直接使用 GitHub 界面
2. 开发者在 Claude Code CLI 中，通过 `/be-review-gaps` 命令可以直接读取 GitHub Issues 并将分析结果写回
3. 两边的操作以 GitHub Issues 为共享上下文，能够正确衔接，不需要额外的同步操作
4. 全流程的状态流转（label 变更）与设计一致

---

## 测试范围

本次只测试 **Gap 协作主流程**，不涉及 Sprint 任务、部署 Checklist 等其他模块。

---

## 参与角色

| 角色 | 工具 | 职责 |
|------|------|------|
| PM | Claude Web + GitHub MCP | 创建 Gap Issue，回复产品决策 |
| 开发者（BE） | Claude Code CLI + GitHub MCP | 运行 `/be-review-gaps`，提供技术分析 |

---

## 测试用例

### TC-01：PM 创建 Gap Issue

**前置条件**：PM 已完成 Claude Project 配置（见 `onboarding-pm-web.md`）

**步骤**：
1. PM 在 Claude Web Project 对话中说"我发现了一个需求问题，帮我创建一个 Gap"
2. PM 描述问题背景，Claude 生成 Issue 草稿
3. PM 确认后，Claude 通过 MCP 创建 Issue，自动打上 label: `gap`, `raised`

**预期结果**：
- GitHub 上出现一个新 Issue，label 为 `gap` + `raised`
- Issue 内容结构清晰，包含问题描述、影响范围、优先级
- PM 的 Claude 对话中有明确的"已创建"确认

**验证点**：
- [ ] Issue 成功创建
- [ ] label 正确：`gap` + `raised`
- [ ] Issue 内容可读，BE 能理解问题

---

### TC-02：开发者运行 /be-review-gaps

**前置条件**：TC-01 通过，GitHub 上有 label: `gap, raised` 的 Issue

**步骤**：
1. 开发者在 Claude Code CLI 中运行 `/be-review-gaps`
2. Claude 通过 MCP 拉取 label: `gap, raised` 的 Issues
3. Claude 读取相关文档，生成技术分析草稿
4. 开发者确认后，Claude 将分析以 Comment 形式写回 Issue
5. Claude 将 Issue label 从 `raised` 改为 `be-reviewed`

**预期结果**：
- Issue 下出现 BE 技术分析 Comment
- label 变更为 `be-reviewed`，`raised` 被移除
- 技术分析包含：技术影响、可选方案、推荐方案、依赖项、工作量估算

**验证点**：
- [ ] MCP 成功拉取到 TC-01 创建的 Issue
- [ ] 技术分析 Comment 内容完整
- [ ] label 正确更新为 `be-reviewed`

---

### TC-03：PM 回复产品决策

**前置条件**：TC-02 通过，Issue 上有 BE 技术分析 Comment

**步骤**：
1. PM 在 Claude Web 说"看看需要我确认的"
2. Claude 通过 MCP 拉取 label: `be-reviewed` 的 Issues
3. Claude 展示 BE 分析内容，引导 PM 给出产品决策方向
4. PM 确认后，Claude 将产品回复以 Comment 写回 Issue
5. Claude 将 label 更新为 `pm-reviewed`

**预期结果**：
- Issue 下出现 PM 产品回复 Comment
- label 变更为 `pm-reviewed`

**验证点**：
- [ ] PM 侧能正确读取 BE 的技术分析
- [ ] 产品回复 Comment 内容完整
- [ ] label 正确更新为 `pm-reviewed`

---

### TC-04：人工确认决策

**前置条件**：TC-03 通过，Issue 上双方分析均已完成

**步骤**：
1. 任一方在 GitHub 界面手动将 label 改为 `confirmed`，或在 Issue 中补充最终决策内容

**预期结果**：
- Issue label 包含 `confirmed`
- Issue 中有明确的决策记录

**验证点**：
- [ ] `confirmed` label 由人工操作打上，非 Agent 自动触发
- [ ] 决策内容清晰，可作为后续开发的依据

---

## 异常场景测试

| 场景 | 预期行为 |
|------|---------|
| PM 未配置 GitHub MCP 就说"今天有什么待处理的" | Claude 提示 MCP 未连接，给出配置引导 |
| `/be-review-gaps` 执行时没有待处理 Issue | 输出"当前没有待处理的 Gap"，正常结束 |
| BE 分析草稿开发者拒绝确认 | 不写入 GitHub，重新生成或放弃 |
| PM 要求 Claude 直接打 `confirmed` label | Claude 拒绝，提示该操作必须由人工完成 |

---

## 通过标准

- TC-01 ~ TC-04 全部通过 → 主流程验证通过
- 异常场景无意外写入 → 安全性验证通过

主流程通过后，进入下一阶段：自动化触发（GitHub Actions）设计。

---

## 测试记录

| 用例 | 执行时间 | 结果 | 备注 |
|------|---------|------|------|
| TC-01 | 2026-03-25 | ✅ 通过 | PM（YanhongPan）通过 Claude Web 创建 Issue #1；label 为 `gap`+`high`（优先级 label 由创建者自选，非流程控制标准） |
| TC-02 | 2026-03-25 | ✅ 通过（降级） | BE 技术分析已写入 Issue #1 Comment，`be-reviewed` label 已添加；MCP 对协作私有仓写入失败，降级为直接 GitHub API 调用 |
| TC-03 | 2026-03-25 | ✅ 通过 | PM（YanhongPan）通过 Claude Web 读取 BE 技术分析，回复产品决策（以 V2 为准，接入 Google OAuth），写回 Issue Comment，label 更新为 `pm-reviewed` |
| TC-04 | 2026-03-25 | ✅ 通过 | YanhongPan 手动将 label 改为 `confirmed`，Issue 关闭（state: completed）；confirmed 操作为人工触发，符合设计约束 |

### 整体结论

**主流程验证通过（2026-03-25）**

TC-01 ~ TC-04 全部在同一天完成闭环。GitHub Issues 作为双方共享上下文层的机制可行：

- PM（Claude Web）和 BE（Claude Code CLI）均通过 MCP 操作同一 Issue，无需额外同步
- label 状态机（gap → be-reviewed → pm-reviewed → confirmed → closed）完整流转
- "Agent 生成草稿，人确认后发出"的铁律在双边均得到执行

**下一阶段**：自动化触发（GitHub Actions）设计，即 `proposals/github-mcp-integration.md` 中的"最小落地路径·第一步"。

### 已知问题

| # | 问题 | 状态 |
|---|------|------|
| P1 | GitHub MCP 对非自有私有仓（协作者身份）读写均失败（Not Found / Authentication Failed），直接 API 调用正常 | 已修复：MCP 配置从 Docker 改为 `npx @modelcontextprotocol/server-github`，重启后生效 |
| P2 | `/be-review-gaps` 命令以 `raised` label 为过滤条件，但实际流程中优先级 label（`high`/`medium` 等）由创建者自选，`raised` 不会自动打上 | 待修订：命令定义需更新过滤逻辑，改为以 `gap` label 为主过滤条件，排除已有 `be-reviewed`/`confirmed`/`closed` 的 Issue |
