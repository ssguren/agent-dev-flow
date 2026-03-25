# /smoke-test-checklist

生成部署后生产环境冒烟测试清单。

## 用法

```
/smoke-test-checklist
```

---

## 执行步骤

### Step 1：识别核心链路接口

从接口规范（OpenAPI 或 `docs/api-spec.md`）中识别接口，并按层级分类：

**识别标准：**
- L1 基础连通：独立于业务的基础设施检查点
- L2 核心认证：所有业务操作的前置条件
- L3 核心业务：系统存在的最主要价值，用户最频繁使用的 1-3 个操作

**原则：每层测试选最小验证集，一次请求能验证最多信息。**

---

### Step 2：生成分层测试清单

#### L1 — 基础连通（目标：1 分钟内完成）

每项测试内容：
- 健康检查端点（`/actuator/health` 或 `/health`）
- 从健康检查响应中确认数据库连接状态
- 从健康检查响应中确认缓存（Redis）连接状态
- 关键外部服务连通性（如支付网关 ping、短信服务可达性）

健康检查一次请求能同时验证上述多个依赖，优先读取健康检查的详细响应。

#### L2 — 核心认证（目标：2 分钟内完成）

必须包含的场景：

1. 正常登录：使用合法凭证登录，验证 token 返回
2. Token 验证：使用有效 token 访问受保护接口，验证 200
3. 无效 Token 拒绝：使用无效 token，验证 401

#### L3 — 核心业务（目标：5 分钟内完成）

从接口规范中选出 1-3 个最关键的用户操作，标准：
- 用户最频繁执行的操作（DAU 最高的路径）
- 系统的核心价值主张（没有这个功能系统没有意义）
- 本次部署直接涉及的功能（新增或修改的核心接口）

**如果接口数量过多，只取最关键的 3 个，总时间控制在 10 分钟内。**

---

### Step 3：为每条测试生成可执行请求示例

每条测试项包含：
- 可直接运行的 curl 命令（使用环境变量占位，不硬编码地址和密码）
- 预期响应的关键字段（不只是状态码，还有响应体的关键值）
- 超时阈值（超过此时间视为异常，建议 L1 < 1s，L2 < 2s，L3 < 5s）
- 结果记录位（留给执行人填写）

格式示例：
```bash
curl -s -X POST "$BASE_URL/api/v1/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "$SMOKE_TEST_USER", "password": "$SMOKE_TEST_PASS"}'
# 预期：HTTP 200，响应体包含 "token" 字段
# 超时阈值：2 秒
# 结果：___
```

---

## 输出格式

```markdown
# 冒烟测试清单 — [服务名] [日期]

**目标环境**：production
**时间预算**：10 分钟
**执行人**：___
**执行时间**：___
**部署版本**：___

---

## L1 — 基础连通（目标：1 分钟）

- [ ] 健康检查
  ```bash
  curl -s "$BASE_URL/actuator/health"
  ```
  预期：HTTP 200，`status: "UP"`，`db.status: "UP"`，`redis.status: "UP"`
  超时阈值：1 秒
  结果：___

- [ ] [如有额外的外部服务检查点，列在此处]

---

## L2 — 核心认证（目标：2 分钟）

- [ ] 正常登录
  ```bash
  curl -s -X POST "$BASE_URL/api/v1/auth/login" \
    -H "Content-Type: application/json" \
    -d '{"username": "$SMOKE_TEST_USER", "password": "$SMOKE_TEST_PASS"}'
  ```
  预期：HTTP 200，响应体包含 `token` 字段（非空字符串）
  超时阈值：2 秒
  结果：___（将 token 存入 $TOKEN 变量）

- [ ] Token 验证（使用上步获得的 token）
  ```bash
  curl -s "$BASE_URL/api/v1/users/me" \
    -H "Authorization: Bearer $TOKEN"
  ```
  预期：HTTP 200，响应体包含当前用户信息
  超时阈值：2 秒
  结果：___

- [ ] 无效 Token 拒绝
  ```bash
  curl -s "$BASE_URL/api/v1/users/me" \
    -H "Authorization: Bearer invalid_token_for_smoke_test"
  ```
  预期：HTTP 401
  超时阈值：1 秒
  结果：___

---

## L3 — 核心业务（目标：5 分钟）

[根据接口规范生成，每个核心操作一条，标注本次部署新增/修改的接口]

- [ ] [核心操作 1]
  ```bash
  [可执行的 curl 命令]
  ```
  预期：[HTTP 状态码]，响应体关键字段：[字段=预期值]
  超时阈值：[X] 秒
  结果：___

---

## 最终结论

- [ ] L1 全部通过
- [ ] L2 全部通过
- [ ] L3 全部通过

**综合结论**：PASS / FAIL

**如果 FAIL**：
- 立即通知 Tech Lead
- 参考 `gen-rollback-plan` 生成的回滚方案评估是否回滚
- 记录失败的具体步骤和错误信息
```

---

## 环境变量约定

生成的 curl 命令统一使用以下环境变量，执行前在终端中 export：

```bash
export BASE_URL="https://api.example.com"         # 生产环境 base URL
export SMOKE_TEST_USER="smoke@example.com"         # 专用冒烟测试账号
export SMOKE_TEST_PASS="xxx"                        # 从密钥管理系统获取
export TOKEN=""                                     # 登录后填入
```

**不要在 checklist 文档中硬编码密码或生产环境 token。**

---

## 规则

- 总耗时必须控制在 10 分钟内：如果 L3 有太多新接口，只选最关键的 3 个
- 每条测试的预期值必须具体（不能只写"应该成功"），至少包含状态码 + 响应体关键字段
- L1 和 L2 是必测的，不能因为时间紧张而跳过
- 冒烟测试发现问题，不是"测试失败"，是"部署门控生效"，是正常流程
