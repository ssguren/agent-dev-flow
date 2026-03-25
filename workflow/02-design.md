# Stage 2 — Design 设计阶段

## 目标
把需求阶段确认的功能范围，转化为可执行的技术方案，并归档所有架构决策。

## 参与角色
- **Tech Lead**（主导）：架构决策、ADR
- **Backend Dev**（支持）：接口设计、数据模型
- **Frontend Dev**（支持）：组件设计、交互协议

## 输入
- `gap/gap-registry.md`（status: confirmed 的条目）
- `requirements/<sprint>/product-requirements.md`

## 输出
- `adr/ADR-xxx-<topic>.md` — 架构决策记录
- `specs/api-specification.md` — 接口规范
- `specs/data-model.md` — 数据模型设计
- `docs/<sprint>/implementation-plan.md` — 开发任务拆分

## 工作流

```
Tech Lead 提出设计方向
    ↓
/draft-adr <topic>      Agent 起草 ADR（背景 + 选项 + 分析）
    ↓
Tech Lead 审阅 ADR       填写"决策"章节，人工确认
    ↓
/draft-api-spec          Agent 基于 ADR + 需求生成接口规范草稿
    ↓
Backend Dev 审阅         补充细节，确认字段、状态码、错误码
    ↓
/split-tasks             Agent 基于接口规范拆分后端任务列表
    ↓
/split-tasks --fe        Agent 拆分前端任务列表
    ↓
Tech Lead 确认任务列表   锁定 Sprint 任务
```

## 命令

| 命令 | 谁运行 | 做什么 |
|------|--------|--------|
| `/draft-adr <topic>` | Tech Lead | Agent 起草 ADR，列出背景、可选方案、优劣分析，留白决策字段给人填 |
| `/draft-api-spec` | Backend Dev | Agent 基于需求文档和 ADR 生成接口规范草稿 |
| `/review-api-spec` | Frontend Dev | Agent 从前端视角检查接口规范是否满足 UI 需求 |
| `/split-tasks` | Tech Lead | Agent 将接口规范拆分为后端 TODO 列表（带优先级和依赖关系） |
| `/split-tasks --fe` | Tech Lead | 同上，针对前端 |

## ADR 规范

**触发条件**：凡满足以下任一条件的决策，必须写 ADR：
- 影响多个模块的架构决策
- 技术选型（框架、中间件、第三方服务）
- 推翻或修改过去的决策
- 有明显的方案取舍（选 A 就放弃 B）

**ADR 结构**：
```markdown
## 背景 Context
## 决策 Decision  ← 只有人能填这一节
## 可选方案 Options
## 结果 Consequences
## 决策记录 Record
```

**ADR 命名**：`ADR-<三位序号>-<英文短描述>.md`，如 `ADR-005-payment-order-design.md`

## 质量门禁（进入下一阶段前必须满足）

- [ ] 所有影响 Sprint 实现的 Gap 已有对应 ADR 或决策记录
- [ ] 接口规范已经前端 review（`/review-api-spec` 完成）
- [ ] 任务列表已拆分，每条任务有明确的验收条件
- [ ] 无悬空依赖（没有"等 XX 决定了再说"的任务）
