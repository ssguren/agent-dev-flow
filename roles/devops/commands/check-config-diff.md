# /check-config-diff

对比两个环境的配置差异。

## 用法

```
/check-config-diff <env1> <env2>
```

示例：
```
/check-config-diff staging production
/check-config-diff dev staging
```

---

## 执行步骤

### Step 1：读取两个环境的配置文件

从以下位置读取配置（按优先级顺序）：

| 配置类型 | 路径约定 |
|---|---|
| Nacos 导出文件 | `nacos-configs/<env>/<service>.yml` |
| Spring 多环境配置 | `src/main/resources/application-<env>.yml` |
| 环境变量文件 | `.env.<env>` 或 `.env.<env>.local` |
| Docker Compose 环境 | `docker-compose.<env>.yml` 中的 `environment` 段 |
| Kubernetes ConfigMap | `k8s/<env>/configmap.yaml` |

如果文件不可访问，停下来要求用户提供配置内容，**不要凭空推断配置值**。

将两个环境的配置展开为扁平的 key-value 对后进行比较（处理嵌套 YAML 时使用点分路径，如 `spring.data.redis.host`）。

### Step 2：对每个差异分类

对两个环境中每一个存在差异的 key，按以下规则分类：

#### EXPECTED — 预期的环境差异

符合以下特征的差异归为 EXPECTED：
- 明显指向不同基础设施的值：数据库地址、Redis 地址、服务 URL、域名
- 日志级别（dev 用 DEBUG，生产用 INFO 或 WARN）
- 环境标识字段（`spring.profiles.active`、`app.env`）
- 监控/链路追踪上报地址

EXPECTED 类差异仍然全部输出，不隐藏，但在报告中归为低优先级。

#### SUSPICIOUS — 可疑差异（可能是遗漏同步）

符合以下特征的差异归为 SUSPICIOUS：
- 功能开关（feature flag）在两个环境值不同，但看不出有意区分的理由
- 超时时间、重试次数、限流阈值在两个环境不一致
- 第三方 API 密钥格式相似但值不同（可能是忘记更新）
- 业务相关参数（金额上限、分页大小）在两个环境不同

**SUSPICIOUS 不代表一定有问题，但需要人工确认是有意为之还是遗漏。**

#### MISSING — env2 中缺少 env1 有的 key

env1 中存在、env2 中完全不存在的 key。

**这是最高优先级。** 缺少配置项通常会导致运行时报错（启动失败、NullPointerException、功能不可用）。

#### EXTRA — env2 中多出 env1 没有的 key

env2 中存在、env1 中完全不存在的 key。

可能原因：
- env2 中有临时配置未清理
- env2 先于 env1 配置了新功能
- 误操作留下的无用配置

---

### Step 3：输出差异清单

**不要自动判断哪些差异"没问题"，全部输出，由人工决定。**

---

## 输出格式

```markdown
# 配置对比：<env1> → <env2>

**生成时间**：YYYY-MM-DD HH:MM
**配置来源**：[列出读取的具体文件路径]

---

## MISSING — <env1> 有、<env2> 没有（高优先级）

这些 key 在 env2 中缺失，可能导致运行时错误。

| Key | env1 中的值 | 建议操作 |
|---|---|---|
| `wechat.app-secret` | `***`（已脱敏） | 确认 env2 是否需要此配置，如需要则添加 |
| `payment.callback-url` | `https://staging.example.com/pay/callback` | 添加 env2 对应的 callback URL |

---

## SUSPICIOUS — 两个环境都有、但值差异可疑（需人工确认）

这些差异可能是遗漏同步，也可能是有意为之。请逐一确认。

| Key | env1 的值 | env2 的值 | 可疑原因 |
|---|---|---|---|
| `app.order.max-retry` | `3` | `5` | 重试次数在两个环境不一致，是否有意？ |
| `app.feature.new-checkout` | `false` | `true` | 功能开关在两个环境不同，是否是生产先开启了？ |
| `jwt.expiry-seconds` | `7200` | `3600` | Token 有效期不同，是否有意为之？ |

---

## EXTRA — <env2> 有、<env1> 没有（需确认是否应同步回去）

| Key | env2 中的值 | 说明 |
|---|---|---|
| `app.debug-mode` | `false` | env1 中不存在此 key，是否需要同步到 env1？ |

---

## EXPECTED — 预期的环境差异（正常，无需操作）

| Key | env1 的值 | env2 的值 |
|---|---|---|
| `spring.data.mongodb.uri` | `mongodb://dev-mongo:27017/mydb` | `mongodb://prod-mongo:27017/mydb` |
| `spring.data.redis.host` | `dev-redis` | `prod-redis` |
| `logging.level.root` | `DEBUG` | `INFO` |
| `spring.profiles.active` | `dev` | `production` |

---

## 汇总

| 类别 | 数量 | 优先级 |
|---|---|---|
| MISSING | N | 高 — 部署前必须处理 |
| SUSPICIOUS | N | 中 — 需要人工逐一确认 |
| EXTRA | N | 低 — 建议确认是否清理 |
| EXPECTED | N | 无需操作 |

---

## 建议操作

1. **MISSING 项**：在部署前，由运维人员在 env2 中添加缺失的配置项。
2. **SUSPICIOUS 项**：由 Tech Lead 或对应模块的负责人逐一确认是有意差异还是遗漏同步。
3. **EXTRA 项**：确认是否需要同步到 env1，或清理 env2 中的无用配置。
4. 处理完毕后，重新执行 `/check-config-diff` 验证差异已收敛。
```

---

## 安全规则

- **密钥、密码、token 等敏感值必须脱敏**：在输出中显示为 `***`，不输出原文。
  - 识别标志：key 名包含 `secret`、`password`、`passwd`、`token`、`key`、`credential`、`api-key` 等
- **不修改任何配置文件**：`/check-config-diff` 只输出差异报告，不执行任何写操作。
- **不判断差异是否安全**：所有 SUSPICIOUS 和 MISSING 差异必须输出，不能因为"看起来没问题"就省略。
- **分类不确定时归为 SUSPICIOUS**：当无法确定某个差异属于 EXPECTED 还是 SUSPICIOUS 时，选择 SUSPICIOUS（保守原则）。
