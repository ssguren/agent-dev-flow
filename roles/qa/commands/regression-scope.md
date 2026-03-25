# /regression-scope — 分析变更影响范围，生成回归测试范围

## 使用说明

```
/regression-scope                               # 分析最新 merged PRs 或当前分支变更
/regression-scope <pr-url>                      # 指定单个 PR
/regression-scope <pr-url-1> <pr-url-2> ...     # 指定多个 PR
```

示例：
```
/regression-scope
/regression-scope https://github.com/org/repo/pull/123
/regression-scope https://github.com/org/repo/pull/123 https://github.com/org/repo/pull/124
```

---

## Step 1：获取变更文件列表

**如果提供了 PR URL：**
- 从 PR 描述和 diff 中提取变更文件列表
- 读取 PR 标题和描述，了解变更意图

**如果未提供 PR URL：**
- 执行 `git diff main...HEAD --name-only` 获取当前分支变更文件
- 或读取最近一次合并 commit 的变更文件列表

---

## Step 2：分析每个变更文件的类型和影响

将变更文件按以下类型分类，并推导影响范围：

### Controller 文件变更

识别标志：类名以 `Controller` 结尾，或位于 `adapter/` 包，带 `@RestController` 注解

影响推导：
- 对应的接口测试（该 Controller 暴露的所有接口）
- 涉及的业务流程测试（该接口参与的业务场景）
- 如有请求体结构变更：客户端调用方的兼容性测试

### Service / Domain 层变更

识别标志：位于 `application/`、`domain/`、`service/` 包，或类名以 `Service`、`UseCase` 结尾

影响推导：
- 该 Service 的单元测试
- 调用该 Service 的所有 Controller 所对应的接口测试
- 如果是公共 Domain Service，影响范围扩大到所有使用该 Domain 的业务流

### Repository / DB 层变更

识别标志：位于 `infrastructure/`、`repository/` 包，或有 DB migration 文件变更

影响推导：
- 数据一致性测试（写入后读取，确认字段正确持久化）
- 所有使用该 Repository 的业务流程
- 如有 schema 变更：历史数据兼容性（已有数据是否还能正常读取）

### Config / Properties 变更

识别标志：`application*.yml`、`.env*`、`bootstrap.yml`、Nacos 配置导出、feature flag 文件

影响推导：
- 全量冒烟测试（配置变更可能影响全局行为）
- 所有依赖该配置 key 的功能模块
- 如果修改了超时 / 重试 / 限流参数：相关接口的性能和稳定性测试

### 公共工具类变更（最高优先级）

识别标志：位于 `util/`、`common/`、`shared/` 包，或被多个模块引用的工具类

影响推导：
- 所有直接使用该工具类的功能模块（需通过代码搜索确认引用范围）
- 优先级最高：工具类变更的影响面通常是横切的，难以穷举

### 依赖文件变更

识别标志：`pom.xml`、`package.json`、`build.gradle`、`go.mod` 等

影响推导：
- 如果升级了关键依赖（Spring、数据库驱动、序列化库）：全量冒烟测试
- 如果引入了新依赖：该依赖相关功能的验证测试

---

## Step 3：生成三级回归范围

### MUST（直接影响，必测）

以下情况归入 MUST：
- 变更文件直接对应的功能
- 与变更接口有直接调用关系的功能
- 业务规则中明确覆盖的场景
- 任何公共工具类变更的下游功能

每条 MUST 条目附具体说明（为什么必测）。

### SHOULD（间接关联，强烈建议测）

以下情况归入 SHOULD：
- 共用变更数据模型的其他功能
- 上下游依赖关系中的相邻模块
- 历史上与变更模块存在已知耦合的功能

每条 SHOULD 条目附间接关联理由。

### CAN-SKIP（明确无关，可跳过）

以下情况可归入 CAN-SKIP：
- 与变更文件无任何代码路径关联的功能
- 已有完整单元测试覆盖且本次逻辑未变更的功能

**每条 CAN-SKIP 条目必须附具体理由，不能只写"与本次变更无关"。**

正确的 CAN-SKIP 理由示例：
- "变更文件均在 `user/` 包下，`product/` 包与 `user/` 包无任何引用关系，商品管理功能无代码路径可被影响"
- "订单支付功能依赖 `PaymentService`，本次仅变更 `ArticleService`，无共享依赖"

---

## 输出格式

```markdown
# 回归测试范围建议

## 变更摘要

- PR：#123 [用户注册流程新增手机号验证]
- 变更文件数：8
- 变更类型：接口变更、业务逻辑变更

## 变更文件分析

### 接口变更
- `UserController.java`：`POST /api/v1/users/register` 请求体新增必填字段 `phone`

### 业务逻辑变更
- `UserService.java`：注册流程新增手机号格式校验和唯一性检查

### DB 层变更
- `V20250101__add_phone_to_users.sql`：users 表新增 `phone` 列（可空，加索引）

## 回归测试范围

### MUST（必测，预计 12 个用例）

- [ ] 用户注册 - 带 phone 字段的完整注册流程（接口直接变更）
- [ ] 用户注册 - 不带 phone 字段的请求（新必填字段兼容性验证）
- [ ] 用户注册 - phone 格式非法时的错误返回
- [ ] 用户注册 - phone 重复时的错误返回
- [ ] 用户列表查询 - 确认 phone 字段是否在响应中正确返回
- [ ] 用户详情查询 - 确认 phone 字段映射正确

### SHOULD（建议测，预计 6 个用例）

- [ ] 用户更新功能（共用 UserService，新增的 phone 校验逻辑可能影响更新路径）
- [ ] 用户导出功能（使用 User 数据模型，新增字段是否在导出文件中出现）

### CAN-SKIP（可跳过）

- 文章管理功能（理由：变更文件均在 `user/` 包下，文章模块使用独立的 `ArticleService`，与 `UserService` 无共享依赖，无代码路径关联）
- 系统设置功能（理由：纯配置读取，未依赖任何本次变更的类或接口）

## 特别关注点

（列出需要人工判断或不确定的地方）
- `phone` 字段在 DB 中设为可空，但 API 规范要求必填，需确认历史存量用户数据的处理方式
- 手机号唯一性索引是否影响已有用户数据（如已有重复手机号记录）

## 建议测试顺序

1. 先执行 MUST 中的冒烟用例（注册 happy path），确认核心功能可用
2. 再执行 MUST 的完整边界用例（格式校验、重复校验）
3. 时间允许时执行 SHOULD 用例
```

---

## 局限性说明

Agent 会在输出末尾附加以下声明：

> 本回归范围建议基于静态文件分析和接口规范推导，存在以下局限：
>
> 1. 无法分析运行时的动态调用关系（如反射调用、事件总线、消息队列）
> 2. 业务隐性依赖需要人工补充（团队了解但未文档化的功能耦合）
> 3. CAN-SKIP 分类不代表"绝对安全"，仅代表"静态分析无关联"
>
> 最终回归范围由 QA 负责人确认。
