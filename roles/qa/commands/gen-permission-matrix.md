# /gen-permission-matrix — 生成权限矩阵测试用例

## 使用说明

```
/gen-permission-matrix
```

无需额外参数，命令自动从项目文档中提取角色定义和接口权限要求。

---

## Step 1：从项目 CLAUDE.md 或 ADR 中提取角色定义

从以下位置（按优先级）读取角色列表和权限定义：

1. 项目 CLAUDE.md 中的角色权限说明
2. `docs/roles.md` 或 `docs/permissions.md`
3. ADR（Architecture Decision Records）中与权限相关的决策
4. 代码中的权限注解（`@PreAuthorize`、`@RolesAllowed`、`@Secured` 等）

提取内容：
- 角色列表（如：FREE / PREMIUM / ADMIN，或 USER / OWNER / ADMIN）
- 每个角色的基本权限描述
- 是否存在资源所有权（owner vs non-owner）概念
- 是否存在组织 / 租户层级权限

如果找不到角色定义文档，输出问题清单并停止（见"规范缺失处理"）。

---

## Step 2：从接口规范中读取每个接口的权限要求

从接口规范（OpenAPI 文件或 `docs/api-spec.md`）提取：

- HTTP 方法和路径（如 `GET /api/v1/users/{id}`）
- 所需角色 / 权限（`security` 字段、接口注释或文档说明）
- 是否涉及资源所有权检查（如"只能操作自己的资源"）
- 是否为公开接口（无需认证）

---

## Step 3：生成两部分输出

### 第一部分：权限矩阵表格

- 行 = 每个接口（HTTP 方法 + 路径）
- 列 = 每个角色 + 一列"匿名用户（未认证）"
- 单元格 = 预期 HTTP 状态码（200 / 401 / 403）

示例（以 FREE / PREMIUM / ADMIN 三角色体系为例）：

| 接口 | 匿名 | FREE | PREMIUM | ADMIN |
|---|---|---|---|---|
| `GET /api/v1/articles` | 401 | 200 | 200 | 200 |
| `POST /api/v1/articles` | 401 | 403 | 200 | 200 |
| `GET /api/v1/articles/{id}` | 401 | 200* | 200 | 200 |
| `PUT /api/v1/articles/{id}` | 401 | 200* | 200* | 200 |
| `DELETE /api/v1/articles/{id}` | 401 | 403 | 403 | 200 |
| `GET /api/v1/admin/users` | 401 | 403 | 403 | 200 |

`*` 表示有资源所有权限制（见"资源所有权测试"章节）。

### 第二部分：403 场景详细测试用例

对矩阵中每个"403"单元格，生成对应的具体测试步骤：

| 用例 ID | 接口 | 测试角色 | 测试步骤 | 预期结果 |
|---|---|---|---|---|
| PM-001 | `POST /api/v1/articles` | FREE | 1. 使用 FREE 角色 token 调用接口<br>2. 传入合法的完整请求体 | HTTP 403，响应含 `permission_denied` |
| PM-002 | `DELETE /api/v1/articles/{id}` | PREMIUM | 1. 使用 PREMIUM 角色 token<br>2. 传入有效的文章 ID（非自己创建）| HTTP 403，响应含 `permission_denied` |
| PM-003 | `GET /api/v1/admin/users` | FREE | 1. 使用 FREE 角色 token 调用接口 | HTTP 403，响应含 `permission_denied` |

**重要**：403 测试必须使用合法的请求体，确保不因请求格式问题返回 400，从而掩盖真正的权限校验结果。

---

## Step 4：资源所有权测试（额外章节）

如果系统存在"用户只能操作自己资源"的权限模型，单独生成本章节。

### 所有权矩阵

| 场景 | 操作者 | 目标资源 | 预期结果 |
|---|---|---|---|
| owner 读自己的资源 | 用户 A | 用户 A 创建的资源 | 200 |
| non-owner 读他人资源 | 用户 B | 用户 A 创建的资源 | 403 |
| owner 修改自己的资源 | 用户 A | 用户 A 创建的资源 | 200 |
| non-owner 修改他人资源 | 用户 B | 用户 A 创建的资源 | 403 |
| owner 删除自己的资源 | 用户 A | 用户 A 创建的资源 | 200 |
| non-owner 删除他人资源 | 用户 B | 用户 A 创建的资源 | 403 |
| 高权限角色跨用户操作 | ADMIN | 用户 A 创建的资源 | 200（若 ADMIN 可跨用户，按规范定义） |

### 所有权测试详细步骤

```
场景：non-owner 读取他人资源

前置条件：
1. 创建用户 A（目标角色），通过登录获取 token-A
2. 创建用户 B（同角色），通过登录获取 token-B
3. 使用 token-A 创建资源 R，记录 resource-id

测试步骤：
1. 使用 token-B 调用 GET /api/v1/resources/{resource-id}

预期结果：
- HTTP 403
- 响应体包含 "permission_denied" 或 "access_denied"
- 不能返回 200（权限校验未生效）
- 不能返回 404（404 会泄露资源存在性信息，除非安全需求明确要求用 404 隐藏资源）

---

场景：non-owner 修改他人资源

前置条件：同上

测试步骤：
1. 使用 token-B 调用 PUT /api/v1/resources/{resource-id}
2. 请求体使用合法格式

预期结果：
- HTTP 403
- 资源数据未发生变化（可用 token-A 查询验证）
```

---

## 规范缺失处理

如果无法从文档中提取以下信息，输出问题清单并停止：

```
权限矩阵生成中断，需要补充以下信息：

1. 找不到角色定义文档（CLAUDE.md、docs/roles.md 均未包含角色列表）
2. 接口 PUT /api/v1/orders/{id} 未定义权限要求
3. 接口规范未说明是否存在资源所有权检查
4. 角色 PREMIUM 的权限边界未明确（哪些接口有权，哪些没有）

请补充以上信息后重新执行 /gen-permission-matrix
```

---

## 完整输出结构

```
# 权限矩阵测试用例

## 1. 角色定义摘要
（来源文件 + 角色列表及说明）

## 2. 接口 × 角色 权限矩阵
（Markdown 表格，含 * 标注说明）

## 3. 403 场景测试用例
（表格 + 每个用例的具体测试步骤）

## 4. 资源所有权测试
（仅在存在所有权模型时输出）
  4.1 所有权矩阵
  4.2 详细测试步骤

## 5. 规范问题清单
（发现的规范模糊或缺失项，供确认）
```
