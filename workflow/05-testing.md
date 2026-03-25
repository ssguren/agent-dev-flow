# Stage 5 — Testing 测试阶段

## 目标
基于需求验收标准和接口规范，系统性地覆盖功能路径、边界情况和权限矩阵，而不是靠人工经验猜测测试点。

## 核心问题
传统测试计划的失败模式：
- QA 基于对需求的理解独立编写，与实现脱节
- 边界情况（空值、超长、并发、权限）靠经验，覆盖不全
- 测试通过了，但线上出现的 Bug 是"没人想到要测"的场景
- 回归测试范围靠人记，经常漏测历史功能

**Agent 的介入点**：Agent 能从接口规范机械性地推导出所有输入组合，能从权限矩阵推导出所有角色 × 操作的测试用例，这是人工容易遗漏的部分。

## 参与角色
- **QA**（主导）：测试计划，验收
- **Backend Dev**（支持）：提供接口细节，解释边界逻辑
- **Agent**：生成测试用例骨架，覆盖边界分析

## 输入
- `requirements/<sprint>/product-requirements.md`（验收标准）
- `specs/api-specification.md`（接口规范）
- 合并后的代码（PR 已 Merge）
- 历史 Bug 记录（如有）

## 输出
- `tests/<sprint>/test-plan.md` — 完整测试计划
- `tests/<sprint>/test-results.md` — 测试执行结果
- Bug Report（在 Issue 系统中，格式见模板）

## 工作流

```
Sprint 开发完成，所有 PR Merge
    ↓
/gen-test-plan          Agent 基于接口规范 + 验收标准生成测试计划草稿
    ↓
QA 审阅补充              添加业务场景测试、探索性测试
    ↓
/gen-boundary-cases     Agent 分析边界情况（空值、超长、并发、幂等性）
    ↓
/gen-permission-matrix  Agent 基于角色权限体系生成权限矩阵测试用例
    ↓
QA 确认测试计划          人工 sign-off
    ↓
执行测试                 人工执行或自动化
    ↓
Bug 录入 Issue          /format-bug 生成标准 Bug Report
    ↓
Dev 修复 → 重新测试
    ↓
测试通过 → 进入 Stage 6（部署）
```

## 命令

| 命令 | 谁运行 | 做什么 |
|------|--------|--------|
| `/gen-test-plan` | QA | Agent 读取接口规范和验收标准，生成分层测试计划（单元/集成/E2E）|
| `/gen-boundary-cases <api>` | QA | Agent 对指定接口分析边界情况（null、empty、max length、negative numbers 等）|
| `/gen-permission-matrix` | QA | Agent 基于角色定义生成角色 × 接口 × 操作的权限矩阵测试用例 |
| `/format-bug` | QA | Agent 根据描述生成标准 Bug Report（步骤 / 预期 / 实际 / 环境）|
| `/regression-scope <pr-list>` | QA | Agent 分析本次变更影响范围，生成回归测试范围 |

## 测试分层标准

| 层级 | 谁负责 | 覆盖目标 | 工具 |
|------|--------|---------|------|
| 单元测试 | Dev | 函数/方法级别，纯业务逻辑 | JUnit / Jest |
| 集成测试 | Dev | 接口 + 数据库/缓存 | Spring Test / Supertest |
| 功能测试 | QA | 业务流程，端到端场景 | 手动 / Postman |
| 回归测试 | QA | 历史功能不被破坏 | 自动化 + 手动 |
| 性能测试 | DevOps | 核心接口响应时间 | k6 / JMeter（按需）|

## Bug Report 模板

```markdown
## Bug Title
[模块] 简短描述（如：[Auth] 微信登录后 JWT 中 accountType 为 null）

## 严重级别
P0 - 核心功能不可用 / P1 - 主流程受影响 / P2 - 次要功能问题 / P3 - 体验问题

## 复现步骤
1. ...
2. ...
3. ...

## 预期结果
...

## 实际结果
...

## 环境
- 分支：
- 测试环境：
- 账号类型：

## 附件
截图 / 日志 / 请求记录
```

## 质量门禁（进入部署前必须满足）

- [ ] 所有 P0 / P1 Bug 已修复并重新验证
- [ ] 测试计划覆盖率：每条验收标准至少一条通过的测试用例
- [ ] 权限矩阵测试已执行
- [ ] 回归测试范围已执行，无新增破坏
