# Stage 6 — Deployment 部署阶段

## 目标
把"测试通过的代码"安全地搬到生产环境，同时确保配置、数据、依赖全部就绪。

## 核心问题
上线翻车的根因分布：
- ~40% 配置问题（忘改某个 key，开发环境和生产环境不一致）
- ~30% 数据问题（迁移脚本没跑，或者跑错了）
- ~20% 依赖问题（下游服务没准备好，外部审批没完成）
- ~10% 代码 Bug（这反而是最少的）

**Agent 的介入点**：生成部署清单，检查配置一致性，生成回滚方案，这些都是系统性的机械工作，最适合 Agent 来做。

## 参与角色
- **DevOps**（主导）：执行部署，做上线决策
- **Backend Dev**（支持）：确认数据迁移，解释配置变更
- **QA**（验证）：生产冒烟测试
- **Tech Lead**（上线决策最终责任人）

## 输入
- 已通过测试的代码 tag / commit
- `tests/<sprint>/test-results.md`（测试通过报告）
- `specs/api-specification.md`（接口变更清单）
- 历史 ADR（影响部署的架构决策）

## 输出
- `deploy/<sprint>/deploy-checklist.md` — 部署前检查清单
- `deploy/<sprint>/rollback-plan.md` — 回滚方案
- `deploy/<sprint>/deploy-log.md` — 部署记录

## 工作流

```
测试通过，准备上线
    ↓
/gen-deploy-checklist   Agent 基于本次变更生成部署清单
    ↓
DevOps + Dev 逐项 check  人工逐条确认（不能 Agent 代替）
    ↓
/gen-rollback-plan       Agent 生成回滚步骤（代码回滚 + 数据回滚）
    ↓
Tech Lead 上线决策       人工 go/no-go
    ↓
执行部署
    ↓
/smoke-test-checklist    Agent 生成生产冒烟测试清单
    ↓
QA 执行冒烟测试          人工执行
    ↓
通过 → 部署完成，写 deploy-log
失败 → 执行回滚 → 写 incident-log
    ↓
进入 Stage 7（复盘）
```

## 命令

| 命令 | 谁运行 | 做什么 |
|------|--------|--------|
| `/gen-deploy-checklist` | DevOps | Agent 分析本次变更（新接口、配置变更、数据迁移、依赖变更），生成逐项检查清单 |
| `/gen-rollback-plan` | DevOps | Agent 生成回滚步骤（代码版本 + 数据库状态 + 缓存清理）|
| `/smoke-test-checklist` | QA | Agent 基于核心接口列表生成生产冒烟测试清单（最小可验证集）|
| `/check-config-diff <env1> <env2>` | DevOps | Agent 对比两个环境的配置文件，输出差异清单 |

## 部署清单结构

```markdown
## 部署清单 — Sprint N

### 代码
- [ ] Tag/Commit 确认：<commit-hash>
- [ ] 分支确认：main / release

### 配置变更
- [ ] Nacos / 环境变量：<变更项列表>
- [ ] 新增配置项是否有默认值？（若无，必须在启动脚本中显式传入）

### 数据库
- [ ] 数据迁移脚本：<脚本文件列表>
- [ ] 迁移脚本是否幂等（可重跑）？
- [ ] 是否需要在迁移前备份？

### 外部依赖
- [ ] 第三方服务是否就绪：<服务列表>
- [ ] API Key / 证书是否已配置到生产环境？
- [ ] 审批是否已完成？（第三方 SDK 审批 / 支付证书 等）

### 基础设施
- [ ] 新增表 / 索引是否在生产 DB 创建？
- [ ] Redis 新增 key 空间是否有冲突风险？

### 回滚预案
- [ ] 回滚步骤已文档化
- [ ] 回滚时间估计：<N 分钟>
- [ ] 数据回滚是否可行？（不可逆的数据变更需要特别标注）
```

## Go/No-Go 判断标准

| 条件 | 结果 |
|------|------|
| 所有 P0/P1 Bug 已修复 | ✅ Go |
| 部署清单全部通过 | ✅ Go |
| 回滚方案已文档化 | ✅ Go |
| 有未解决的外部依赖 | ❌ No-Go |
| 数据迁移脚本未验证 | ❌ No-Go |
| 回滚不可行（不可逆变更）| ⚠️ 需 Tech Lead 特别批准 |

## 质量门禁（部署后必须确认）

- [ ] 冒烟测试通过
- [ ] 错误率监控正常（部署后 15 分钟内）
- [ ] 核心接口响应时间正常
- [ ] 部署日志已写入 `deploy-log.md`
