# /gen-rollback-plan — 生成部署回滚方案

## 使用说明

```
/gen-rollback-plan
```

无需额外参数，命令自动读取本次变更信息（从 deploy checklist 或 git diff）生成完整回滚方案。建议在执行 `/gen-deploy-checklist` 之后运行本命令，以便获得更准确的变更信息。

---

## Step 1：分析变更类型

读取以下来源（按优先级）获取本次变更信息：

1. 已生成的 deploy checklist（`docs/deploy/` 目录下的最新文件）
2. `git diff <上一个稳定 tag>...HEAD --name-only`
3. 最近合并的 PR 描述

将变更分为以下类型（一次部署可能包含多种类型）：

- **纯代码变更**：只有业务逻辑修改，无 DB 和配置变更
- **DB schema 变更**：有 migration 文件，可能包含加列、删列、改类型等
- **配置变更**：修改了 `application*.yml`、Nacos 配置、环境变量
- **依赖外部服务变更**：调用了新的外部服务接口，或修改了现有外部服务的调用方式

---

## Step 2：对每类变更生成对应回滚步骤

### 纯代码变更

**可回滚性：可回滚**

回滚方案（选其一）：

```bash
# 方案 A：部署上一个稳定 tag（推荐，最快）
# 将 <previous-tag> 替换为上一个稳定版本号
kubectl set image deployment/service service=registry.example.com/service:<previous-tag>
kubectl rollout status deployment/service --timeout=5m

# 方案 B：通过 git revert 生成新版本
git revert <commit-sha>
git push origin main
# 等待 CI/CD 构建并部署新镜像

# 验证回滚生效
kubectl rollout status deployment/service
curl -s https://api.example.com/health | jq '.version'
```

### DB Schema 变更

**需逐条判断可逆性：**

| 操作类型 | 可逆性 | 回滚思路 |
|---|---|---|
| 加列（可空） | 可回滚 | `ALTER TABLE ... DROP COLUMN ...` |
| 加列（NOT NULL，有默认值） | 可回滚 | `ALTER TABLE ... DROP COLUMN ...` |
| 加索引 | 可回滚 | `DROP INDEX ...` |
| 加表 | 可回滚 | `DROP TABLE ...`（注意数据丢失风险） |
| 删列 | **不可回滚** | 数据已丢失，需 forward-fix |
| 修改列类型 | **通常不可回滚** | 数据转换可能有精度损失 |
| 加 NOT NULL 约束（已有 null 数据） | **不可回滚** | 历史数据已变更 |
| 数据迁移脚本 | **通常不可回滚** | 数据已被修改，需 forward-fix |

**可逆 DB 变更的回滚命令示例：**

```sql
-- 回滚：删除新增的列
ALTER TABLE users DROP COLUMN phone;

-- 回滚：删除新增的索引
DROP INDEX idx_users_phone ON users;
```

执行 DB 回滚前必须：
1. 先完成代码回滚（先回滚代码，再执行 DB 回滚 SQL）
2. 备份当前数据（`mysqldump` 或数据库快照）
3. 在 staging 环境验证回滚 SQL 执行无误

**不可逆 DB 变更 — forward-fix 方向：**

```
WARNING: 不可逆操作
以下 DB 变更无法通过回滚恢复，需要 forward-fix：

- 删除了列 users.legacy_field（数据已永久丢失）
  forward-fix 思路：若业务需要该数据，需从备份导入；若不需要，评估影响后保持现状

- 修改了列 orders.amount 类型（DECIMAL(10,2) → DECIMAL(15,4)）
  forward-fix 思路：确认数据精度是否正确，编写补丁脚本修复异常数据
```

### 配置变更

**可回滚性：可回滚（具体步骤取决于配置管理方式）**

```bash
# Nacos 配置回滚：在 Nacos 控制台操作
# 路径：Nacos 控制台 → 配置管理 → 配置列表 → 目标配置 → 历史版本 → 选择回滚目标版本

# 环境变量回滚（Kubernetes ConfigMap）
kubectl apply -f k8s/configmap-previous.yaml
kubectl rollout restart deployment/service

# 验证配置生效
kubectl exec -it <pod-name> -- env | grep TARGET_CONFIG_KEY
```

如果配置未版本化（无 Nacos / 未 git 跟踪），在输出中标注：

```
WARNING: 配置未版本化
此配置变更无历史版本可切换，需要手动恢复旧值。
请确认旧的配置值（检查 git history 或联系上次修改人）后再操作。
```

### 依赖外部服务变更

**可回滚性：依赖对方接口的向后兼容性**

```
如果是新增调用外部接口：
  代码回滚即可（旧代码不会调用新接口）

如果是修改了现有外部接口的调用参数：
  代码回滚后确认旧版本的调用参数外部服务仍能接受
  如外部服务已废弃旧接口格式：需 forward-fix，代码回滚无法解决

如果有 feature flag：
  降级步骤：关闭 feature flag（如 use-new-external-api=false）
  验证：确认流量已切回旧逻辑
```

---

## Step 3：明确标注不可逆变更

对所有不可逆变更，在输出文档顶部用以下格式汇总：

```
## 不可逆变更汇总（部署前必须确认）

本次部署包含以下不可逆变更，一旦执行无法通过回滚恢复：

1. [不可逆] DB 删列：users 表删除 legacy_field 列
   影响：该列数据永久丢失，无法通过回滚恢复
   是否已备份：[待确认]

2. [不可逆] 数据迁移：orders.status 从数字枚举改为字符串枚举
   影响：历史数据已被修改，无法通过 SQL 完全还原
   forward-fix 思路：如迁移结果有误，需编写反向脚本并人工校验

操作：请 Tech Lead 确认以上变更已知晓，且已完成数据备份，再执行部署。
```

---

## Step 4：输出格式

```markdown
# 回滚方案 — [服务名] [版本号] [日期]

## 不可逆变更汇总

[如有不可逆变更，在此列出；如无，标注"本次部署无不可逆变更"]

---

## 回滚触发条件（满足任意一条，启动回滚评估）

- 错误率 > 5%，持续 3 分钟
- P99 响应时间 > 5 秒，持续 5 分钟
- 核心业务接口（登录 / 下单 / 支付）成功率 < 95%
- 监控告警连续触发 3 次以上

（以上为建议值，请根据项目实际 SLA 调整）

## 决策人

- go/no-go（是否回滚）：Tech Lead [姓名] / 值班负责人 [联系方式]
- 不可逆操作二次确认：[姓名] [联系方式]

---

## 回滚执行步骤

### 第 1 步：代码回滚（预计耗时 5 分钟）

```bash
kubectl set image deployment/service service=registry.example.com/service:v1.2.3
kubectl rollout status deployment/service --timeout=5m
```

验证：
```bash
curl -s https://api.example.com/health | jq '.version'
# 预期：{"version":"v1.2.3","status":"UP"}
```

### 第 2 步：DB 回滚（预计耗时 2 分钟，仅在有可逆 DB 变更时执行）

先备份当前数据：
```bash
mysqldump -u root -p dbname > backup-before-rollback-$(date +%Y%m%d%H%M).sql
```

执行回滚 SQL：
```sql
ALTER TABLE users DROP COLUMN phone;
DROP INDEX idx_users_phone ON users;
```

验证：
```sql
SHOW COLUMNS FROM users;
-- 确认 phone 列已不存在
```

### 第 3 步：配置回滚（预计耗时 3 分钟，仅在有配置变更时执行）

```bash
kubectl apply -f k8s/configmap-v1.2.3.yaml
kubectl rollout restart deployment/service
kubectl exec -it <pod-name> -- env | grep CHANGED_CONFIG_KEY
```

---

## 预计总耗时

| 步骤 | 耗时 |
|---|---|
| 代码回滚 | 5 分钟 |
| DB 回滚（如需） | 2 分钟 |
| 配置回滚（如需） | 3 分钟 |
| 验证确认 | 5 分钟 |
| **合计** | **约 10-15 分钟** |

## 数据影响说明

- 回滚过程中新写入的数据：不受代码回滚影响，数据仍在 DB 中
- DB 回滚影响的数据范围：[根据具体 migration 说明]
- 用户感知：滚动重启期间可能有短暂请求失败（约 30 秒内）
```
