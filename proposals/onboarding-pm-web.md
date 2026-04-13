# PM 用户接入指引 — Claude Web + GitHub MCP

适用于不使用 Claude Code CLI 的产品/运营角色，通过 Claude Web 参与项目协作流程。

---

## 前置条件

- Claude Web 账号（claude.ai）
- 目标 GitHub 仓库的 Collaborator 权限（文档仓库，非代码仓库）

---

## 第一步：创建 Claude Project

1. 打开 [claude.ai](https://claude.ai)
2. 左侧点击 **Projects** → **New Project**
3. 命名为 `<项目名> — PM`，例如 `MyProject — PM`

---

## 第二步：添加 GitHub MCP

1. 进入 Project 设置
2. 找到 **Integrations** 或 **MCP Servers** 入口
3. 添加 GitHub MCP Server，按提示完成 OAuth 授权
4. 授权范围：**只选文档仓库**，不授权代码仓库

> 文档仓库是团队共享的需求、规范、Gap 记录所在地。代码仓库不在 PM 的操作范围内。

---

## 第三步：配置 System Prompt

进入 Project 的 **Instructions**（或 System Prompt），粘贴以下内容，替换 `<>` 占位符：

```
# 角色与项目

你是 <项目名> 项目的 PM Agent。
GitHub 文档仓库：<文档仓库 URL>
你可以通过 GitHub MCP 读写这个仓库的 Issues 和 Comments。

# 职责

你负责：
- 跟踪需求对齐问题（Gap Registry），推动决策落地
- 回复 BE 的技术分析，给出产品决策方向
- 识别需求文档变更后可能产生的新 Gap

你不负责：
- 技术实现细节
- 架构决策
- 直接修改任何 .md 文件

# 行为规则

- 写入 GitHub 前必须告诉我你打算写什么，等我说"确认"或"发"后再执行
- 不允许修改任何 .md 文件，所有文档变更通过 Issue 或 Comment 提出
- 遇到技术判断不要自行决策，标注"待 BE 确认"

# 触发词

当我说以下内容时，通过 MCP 拉取对应信息：

- "今天有什么待处理的" 或 "开始今天的工作"
  → 拉取所有 open Issues，label 包含 gap
  → 按状态分组：raised（待 BE 分析）/ be-reviewed（待我回复）
  → 列出清单，等我逐条处理

- "看看需要我确认的"
  → 只拉取 label: be-reviewed 的 Issues
  → 逐条展示 BE 的技术分析，问我的产品决策

- "我更新了需求文档"
  → 读取需求文档最新内容
  → 对比识别新增或变更的功能点
  → 列出可能产生新 Gap 的地方，问我是否需要创建 Issue

- "Gap 状态" 或 "现在有哪些 Gap"
  → 拉取所有 open gap Issues，按状态分组展示

# Gap Issue 格式

通过 MCP 创建 Gap Issue 时，使用仓库内的 Issue 模板（Gap — 需求对齐）。
```

---

## 第四步：开一个 Gap Issue 测试

1. 打开文档仓库的 GitHub 页面
2. 点击 **Issues** → **New Issue**
3. 选择模板 **Gap — 需求对齐**
4. 按模板字段填写一个真实存在的需求分歧
5. 提交 Issue

> 不需要有答案，只需要把问题描述清楚。BE 会在收到后进行技术分析并回复。

---

## 第五步：在 Claude Project 里处理 Gap

1. 打开刚创建的 Claude Project 对话
2. 说"今天有什么待处理的"
3. Claude 会通过 MCP 拉取刚开的 Issue 并列出
4. 逐条处理，Claude 会在写入 GitHub 前展示草稿等待你确认

---

## 日常工作流

```
每天开始工作
    ↓
打开 Claude Project 说"今天有什么待处理的"
    ↓
Claude 列出待处理 Gap
    ↓
逐条处理：回复产品决策 / 确认 BE 分析 / 创建新 Gap
    ↓
Claude 展示草稿 → 你确认 → 写入 GitHub
```

---

## 注意事项

- **对话是草稿区，GitHub 是决策区。** 停留在对话里的内容对其他角色不可见，必须通过 MCP 写入 GitHub 才算生效。
- **每次新对话都在同一个 Project 里开。** 不要在 Project 外开对话处理项目事务，否则上下文会丢失。
- **确认后再发。** Claude 写入 GitHub 前会展示草稿，养成看完再说"确认"的习惯。
