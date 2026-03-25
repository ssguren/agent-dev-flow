# /check-config-diff — 对比环境配置差异

## 使用说明

```
/check-config-diff <env1> <env2>
```

示例：
```
/check-config-diff staging production
/check-config-diff dev staging
/check-config-diff staging production --whitelist docs/config-whitelist.md
```

- `env1`：基准环境（通常是来源环境，如 staging）
- `env2`：目标环境（通常是待验证的环境，如 production）
- `--whitelist`：指定已知差异白名单文件路径（可选）

---

## Step 1：读取两个环境的配置文件

从以下来源读取配置（尽量多读，合并后对比）：

| 配置来源 | 典型路径 | 说明 |
|---|---|---|
| Spring 配置 | `application-<env>.yml` / `application-<env>.properties` | 标准 Spring Boot 配置 |
| Nacos 导出 | `nacos-export-<env>.yaml` / `docs/nacos/` | Nacos 配置中心导出文件 |
| 环境变量文件 | `.env.<env>` / `k8s/<env>/configmap.yaml` | 环境变量或 K8s ConfigMap |
| Docker Compose | `docker-compose.<env>.yml` | 本地或简单部署时 |

读取完成后，将两个环境的所有配置展开为扁平的 key=value 格式，便于逐条对比：

```
# env1（staging）扁平化结果示例
server.port=8080
spring.datasource.url=jdbc:mysql://staging-db:3306/app
spring.redis.host=staging-redis
app.feature.new-ui=true
app.timeout.request=5000

# env2（production）扁平化结果示例
server.port=8080
spring.datasource.url=jdbc:mysql://prod-db:3306/app
spring.redis.host=prod-redis
app.feature.new-ui=false
app.timeout.request=3000
```

如果某个环境的配置文件无法找到或无法读取，停止并报告：

```
无法读取配置文件，请检查以下路径是否存在：
- staging: application-staging.yml — 找到
- production: application-production.yml — 未找到

请提供 production 环境的配置文件路径后重新执行。
```

---

## Step 2：分类所有差异

将所有差异分为四类：

### EXPECTED（正常的环境差异）

定义：预期中不同环境就应该有不同值的配置，属于正常运维状态。

常见的 EXPECTED 差异（作为默认白名单）：
- 数据库连接地址（`spring.datasource.url`、`spring.datasource.username`）
- Redis 地址（`spring.redis.host`、`spring.redis.port`）
- 日志级别（`logging.level.*`，生产通常 INFO，开发通常 DEBUG）
- 服务域名 / 回调地址（`app.callback-url`、`app.base-url`）
- 外部服务地址（`payment.api-url`、`sms.endpoint`）

如果提供了白名单文件（`--whitelist`），加载白名单中的额外已知差异。

EXPECTED 差异在输出中**折叠显示**（提供摘要，不展开每条）。

### SUSPICIOUS（可能遗漏同步的配置）

定义：两个环境都有这个 key，但值不同，且不在 EXPECTED / 白名单中。

这类差异最需要关注：可能是 staging 做了修改但忘记同步到 production，也可能是 production 有临时改动没有同步回来。

每条 SUSPICIOUS 差异输出：
- key 名
- env1（staging）的值
- env2（production）的值
- 风险推测（根据 key 名推断，如"超时参数差异可能影响生产稳定性"）

### MISSING（env2 中缺少 env1 有的 key）

定义：env1（staging）有这个配置 key，但 env2（production）完全没有。

这是高危情况：生产可能缺少必要配置，服务启动可能用了错误的默认值或直接报错。

MISSING 类型必须高亮显示，不能折叠。

### EXTRA（env2 中多出 env1 没有的 key）

定义：env2（production）有这个配置 key，但 env1（staging）完全没有。

可能是生产有临时修改未同步回 staging，或有遗留的废弃配置。需确认是否应同步回来或清理。

---

## Step 3：输出格式

MISSING 和 SUSPICIOUS 必须高亮显示并置顶，EXPECTED 折叠显示。

```markdown
# 配置差异报告

**对比：** staging vs production
**生成时间：** 2025-01-01 10:00
**配置文件来源：** application-staging.yml + application-production.yml

---

## MISSING（高危：production 缺少的配置）

> 以下配置在 staging 中存在，但 production 中完全缺失。
> 可能导致服务启动失败或使用错误默认值。必须确认后再部署。

| Key | staging 值 | production 值 | 风险说明 |
|---|---|---|---|
| `app.feature.payment-v2` | `true` | 未配置 | feature flag 缺失，production 可能使用旧支付流程 |
| `app.rate-limit.max-requests` | `100` | 未配置 | 限流配置缺失，production 可能无限流保护 |

**操作建议：** 在部署前补充上述配置到 production。

---

## SUSPICIOUS（可疑：两环境值不同，非预期差异）

> 以下配置两个环境均有，但值不同，且不在已知差异白名单中。
> 请逐条确认是故意差异（加入白名单）还是遗漏同步（需要同步）。

| Key | staging 值 | production 值 | 风险推测 |
|---|---|---|---|
| `app.timeout.request` | `5000` | `3000` | 超时参数差异，production 更激进，可能影响稳定性 |
| `app.cache.ttl` | `3600` | `1800` | 缓存 TTL 差异，production 缓存更新更频繁 |
| `app.feature.new-ui` | `true` | `false` | feature flag 差异，可能是故意的，也可能是遗漏开启 |

**操作建议：** 逐条与团队确认，确认为故意差异的加入白名单，确认为遗漏同步的补充同步。

---

## EXTRA（production 多出的配置）

> 以下配置在 production 中存在，但 staging 中没有。
> 可能是临时修改、历史遗留或应同步回 staging 的配置。

| Key | production 值 | 说明 |
|---|---|---|
| `app.legacy.old-feature` | `disabled` | 疑似废弃配置，建议确认是否可清理 |

---

## EXPECTED（已知正常差异，折叠）

> 以下 12 条差异属于正常的环境差异（数据库地址、日志级别等），无需关注。

<details>
<summary>展开查看 12 条 EXPECTED 差异</summary>

| Key | staging 值 | production 值 |
|---|---|---|
| `spring.datasource.url` | `jdbc:mysql://staging-db:3306/app` | `jdbc:mysql://prod-db:3306/app` |
| `spring.redis.host` | `staging-redis` | `prod-redis` |
| `logging.level.root` | `DEBUG` | `INFO` |
| ... | ... | ... |

</details>

---

## 汇总

| 类型 | 数量 | 操作 |
|---|---|---|
| MISSING（高危） | 2 | 部署前必须补充到 production |
| SUSPICIOUS（可疑） | 3 | 逐条确认，决定是否同步 |
| EXTRA | 1 | 确认是否清理或同步回 staging |
| EXPECTED | 12 | 无需操作 |

**建议：** 处理所有 MISSING 和 SUSPICIOUS 后再执行部署。
```

---

## 注意事项

- **不自动判断哪些差异可以忽略**，全部列出，由人决定
- **不自动修改任何环境的配置**，只输出差异报告
- EXPECTED 的判断依据：key 名符合常见环境变量（DB 地址、Redis 地址、日志级别、域名）或在白名单文件中明确列出
- 不在白名单中的差异，即使看起来"正常"，也归入 SUSPICIOUS，不擅自归类为 EXPECTED
- 配置中包含密钥 / 密码的 value（key 名含 `password`、`secret`、`key`、`token`），在报告中脱敏显示为 `***`，不输出实际值
