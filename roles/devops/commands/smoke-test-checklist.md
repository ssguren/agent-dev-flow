# /smoke-test-checklist — 生成生产冒烟测试清单

## 使用说明

```
/smoke-test-checklist
```

无需额外参数。命令自动读取接口规范，生成部署后可在 10 分钟内完成的生产冒烟测试清单。

---

## Step 1：读取接口规范，识别三类核心接口

从以下位置读取接口规范：
1. OpenAPI / Swagger 文件
2. `docs/api-spec.md`
3. Controller 代码（作为补充）

识别以下三类核心接口：

**认证链路**：登录、Token 验证、Token 刷新、登出

**核心业务操作**：该系统最关键的 1-3 个用户操作（根据接口规范判断；如无法判断，列出候选接口并询问确认）

**数据读写**：典型的列表查询、详情查询、创建操作（验证数据库连通性和读写链路）

---

## Step 2：生成分层冒烟清单

### L1 — 基础连通（预计 2 分钟）

验证服务本身和核心依赖是否存活：

```
[ ] 健康检查端点
    curl -s https://api.example.com/actuator/health | jq '.status'
    预期：{"status":"UP"}
    响应时间阈值：< 500ms

[ ] 数据库连接
    从健康检查响应中确认 DB 状态：jq '.components.db.status'
    预期："UP"

[ ] Redis 连接（如有）
    jq '.components.redis.status'
    预期："UP"

[ ] 关键外部服务连通（如有，如支付网关、短信服务）
    curl -s https://api.example.com/actuator/health | jq '.components.<service-name>.status'
    预期："UP"
```

### L2 — 认证链路（预计 3 分钟）

验证认证系统完整可用：

```
[ ] 登录（核心，无法跳过）
    curl -s -X POST https://api.example.com/api/v1/auth/login \
      -H "Content-Type: application/json" \
      -d '{"username":"smoke-test@example.com","password":"<test-password>"}' | jq '.'
    预期：HTTP 200，响应含 access_token 和 refresh_token
    响应时间阈值：< 2000ms

[ ] Token 验证（使用上一步获取的 token）
    curl -s https://api.example.com/api/v1/auth/me \
      -H "Authorization: Bearer <access_token>" | jq '.data.id'
    预期：返回当前用户 ID，非空

[ ] Token 刷新（如有 refresh_token 机制）
    curl -s -X POST https://api.example.com/api/v1/auth/refresh \
      -H "Content-Type: application/json" \
      -d '{"refresh_token":"<refresh_token>"}' | jq '.data.access_token'
    预期：返回新的 access_token，非空

[ ] 无效 Token 被拒绝
    curl -s https://api.example.com/api/v1/auth/me \
      -H "Authorization: Bearer invalid-token" | jq '.code'
    预期：HTTP 401 或响应码为未认证错误码
```

### L3 — 核心业务 happy path（预计 5 分钟）

本次 Sprint 新增或修改的最关键 1-3 个用户操作，只测 happy path，不测边界：

（以下为示例，实际内容根据接口规范生成）

```
[ ] [本次 Sprint 新增，重点关注] 创建文章
    curl -s -X POST https://api.example.com/api/v1/articles \
      -H "Authorization: Bearer <access_token>" \
      -H "Content-Type: application/json" \
      -d '{"title":"冒烟测试文章","content":"测试内容"}' | jq '.data.id'
    预期：HTTP 200，返回新创建的文章 ID，非空
    响应时间阈值：< 2000ms

[ ] 查询文章列表
    curl -s "https://api.example.com/api/v1/articles?page=1&size=10" \
      -H "Authorization: Bearer <access_token>" | jq '.data.total'
    预期：HTTP 200，total >= 1（上一步创建的文章应出现）
    响应时间阈值：< 1000ms

[ ] 查询文章详情
    curl -s "https://api.example.com/api/v1/articles/<article-id>" \
      -H "Authorization: Bearer <access_token>" | jq '.data.title'
    预期：HTTP 200，title = "冒烟测试文章"
    响应时间阈值：< 500ms
```

---

## 输出格式

```markdown
# 生产冒烟测试清单 — [服务名] [版本号] [日期]

**预计总时间：10 分钟**
**执行人：**
**执行时间：**

---

## L1 — 基础连通（预计 2 分钟）

- [ ] 健康检查端点
  ```bash
  curl -s https://api.example.com/actuator/health | jq '.status'
  ```
  预期：`"UP"` | 阈值：< 500ms | 结果：___

- [ ] 数据库连接
  ```bash
  curl -s https://api.example.com/actuator/health | jq '.components.db.status'
  ```
  预期：`"UP"` | 结果：___

- [ ] Redis 连接
  （同上，查看 redis 组件状态）
  预期：`"UP"` | 结果：___

---

## L2 — 认证链路（预计 3 分钟）

- [ ] 登录
  ```bash
  curl -s -X POST https://api.example.com/api/v1/auth/login \
    -H "Content-Type: application/json" \
    -d '{"username":"smoke-test@example.com","password":"<test-password>"}'
  ```
  预期：HTTP 200，含 access_token | 阈值：< 2000ms | 结果：___

- [ ] Token 验证
  ```bash
  curl -s https://api.example.com/api/v1/auth/me \
    -H "Authorization: Bearer <上一步的 access_token>"
  ```
  预期：返回用户 ID | 结果：___

- [ ] Token 刷新
  （使用 refresh_token 换新 access_token）
  预期：返回新 access_token | 结果：___

---

## L3 — 核心业务（预计 5 分钟）

[本次 Sprint 新增或修改的接口，标注"新增，重点关注"]

- [ ] **[本次 Sprint 新增，重点关注]** 创建文章
  ```bash
  curl -s -X POST https://api.example.com/api/v1/articles \
    -H "Authorization: Bearer <access_token>" \
    -H "Content-Type: application/json" \
    -d '{"title":"冒烟测试文章","content":"测试内容"}'
  ```
  预期：HTTP 200，返回 article.id | 阈值：< 2000ms | 结果：___

- [ ] 查询文章列表
  ```bash
  curl -s "https://api.example.com/api/v1/articles?page=1&size=10" \
    -H "Authorization: Bearer <access_token>"
  ```
  预期：HTTP 200，total >= 1 | 阈值：< 1000ms | 结果：___

---

## 冒烟结果汇总

- L1 通过：是 / 否
- L2 通过：是 / 否
- L3 通过：是 / 否
- 整体结论：通过 / 不通过（原因：___）
- 执行人签字：___
- 完成时间：___
```

---

## 注意事项

- 冒烟测试账号（`smoke-test@example.com`）需提前在生产环境创建，并在 KeePass / 密码管理器中记录
- 冒烟测试创建的数据（如测试文章）需在测试结束后手动清理，或使用专用测试租户隔离
- L1 任何一项不通过：停止冒烟，回报 Tech Lead，等待处置
- L2 / L3 任何一项不通过：记录具体失败信息，报告 Tech Lead，由 Tech Lead 决定是否触发回滚评估
- 冒烟清单中标注"本次 Sprint 新增，重点关注"的接口为高风险接口，失败时优先排查
