# Stage 7 — Postmortem 复盘阶段

## 目标
把每次 Sprint / 事故的经验教训结构化归档，让它能反馈到下一次的需求和设计阶段，而不是每次都重新犯同样的错误。

## 核心问题
传统复盘的失败模式：
- 只在出线上事故时复盘，正常 Sprint 结束没有复盘
- 复盘开个会，产出几条"下次要注意"，然后消失在飞书里
- 下一个 Sprint 开始时，上一次的教训没有人记得
- 同类问题一年内至少出现两次

**Agent 的介入点**：Agent 能从 Gap Registry、Bug List、部署日志中提取模式，生成结构化的"下次预防措施"，并自动触发 Gap Registry 中的相关项更新。

## 参与角色
- **Tech Lead**（主导 Sprint 复盘）
- **全员**（事故复盘强制参与）
- **Agent**：提取模式，生成复盘草稿

## 输入
- `gap/gap-registry.md`（本次 Sprint 的 Gap 处理情况）
- `tests/<sprint>/test-results.md`（Bug 列表）
- `deploy/<sprint>/deploy-log.md`（部署情况）
- 事故 Incident 记录（如有）

## 输出
- `postmortem/<sprint>/sprint-retro.md` — Sprint 复盘
- `postmortem/incidents/<date>-<title>.md` — 事故复盘（如有）
- 反馈到 Gap Registry：新增 `raised_by: retro` 的 Gap 条目
- 反馈到 CLAUDE.md：更新编码规范 / Agent 使用规范

## 工作流

### Sprint 复盘（每个 Sprint 结束）

```
Sprint 结束
    ↓
/gen-sprint-retro        Agent 基于 Gap Registry + Bug List + 部署记录生成复盘草稿
    ↓
团队会议 30 分钟         人工讨论，聚焦"下次怎么预防"
    ↓
/update-from-retro       Agent 将复盘结论写入 Gap Registry 和 CLAUDE.md
    ↓
Tech Lead 确认更新       人工 sign-off
```

### 事故复盘（线上 P0/P1 事故后 48 小时内）

```
事故处理完毕
    ↓
/gen-incident-postmortem  Agent 基于事故时间线生成五问分析草稿
    ↓
相关人员补充根因           人工填写真正的根因（Agent 分析不替代人工判断）
    ↓
生成"预防措施"清单         反馈到 Gap Registry
```

## 命令

| 命令 | 谁运行 | 做什么 |
|------|--------|--------|
| `/gen-sprint-retro` | Tech Lead | Agent 分析本次 Sprint 的 Gap 处理率、Bug 分布、部署是否顺利，生成复盘草稿 |
| `/gen-incident-postmortem <incident>` | Tech Lead | Agent 基于事故描述生成五问（5 Whys）分析框架草稿 |
| `/update-from-retro` | Tech Lead | Agent 将复盘结论写入 Gap Registry（新增预防性 Gap）和 CLAUDE.md |

## Sprint 复盘结构

```markdown
## Sprint N 复盘 — YYYY-MM-DD

### 交付情况
- 计划任务数：N，实际完成：N，推迟：N
- 推迟原因：...

### Gap 处理情况
- 本次 Sprint 的 Gap 总数：N
- confirmed 率：N%
- 未解决原因：...

### 质量情况
- P0/P1 Bug 数：N（上线前修复 N，上线后发现 N）
- 最常见 Bug 类型：...

### 部署情况
- 是否顺利：是 / 否
- 翻车原因（如有）：...

### 根因分析
- 本次最大的摩擦点是什么？
- 根因（翻译成本 / 上下文断裂 / 决策延迟 / 其他）：

### 下次预防措施
- [ ] 措施 1（责任人：XX，更新到：CLAUDE.md / Gap Registry / 流程文档）
- [ ] 措施 2
```

## 事故复盘结构（五问法）

```markdown
## Incident Postmortem — YYYY-MM-DD <简短标题>

### 时间线
| 时间 | 事件 |
|------|------|

### 影响范围
- 影响用户数 / 功能 / 时长：

### 根因分析（5 Whys）
1. 为什么出现了这个现象？
2. 为什么导致了上述原因？
3. 为什么...
4. 为什么...
5. 根本原因：

### 预防措施
| 措施 | 类型 | 责任人 | 截止日期 |
|------|------|--------|---------|
| ... | 代码/流程/监控/文档 | ... | ... |

### 反馈到 Gap Registry
- 新增 Gap ID：GAP-XXXX（预防性技术改进）
```

## 反馈闭环

复盘的价值在于**反馈**，不在于记录。每次复盘必须产生至少一条对上游的修改：

| 发现 | 反馈到哪里 |
|------|----------|
| 需求描述不清晰导致返工 | 需求文档模板（`templates/product-requirements.md`）|
| 某类 Bug 反复出现 | 开发阶段质量门禁 + CLAUDE.md 编码规范 |
| 部署失败因为忘了某个步骤 | 部署清单模板（`templates/deploy-checklist.md`）|
| Agent 生成的内容质量差 | 对应命令文件（`commands/` 下的 `.md`）|
| PM-BE 翻译成本高 | Gap Registry 流程或 `/be-review-gaps` 命令 |
