# Command: task-start（Frontend 版）

**角色**：Frontend Dev Agent
**用途**：开始一个具体前端开发任务前的准备工作。读取任务上下文，生成 TypeScript 类型定义、组件骨架、Service 骨架，让开发者直接进入编码状态而不是从空白文件开始。

**调用方式**：
```
/task-start <task-id>
/task-start
```

**参数说明**：
- 带 `<task-id>` 时，直接查找对应任务
- 不带参数时，进入交互式模式：列出当前任务列表供选择

**输出原则**：只生成骨架和接口定义，不生成完整实现。所有 `// TODO` 项明确标注需要人工补充的内容。

---

## Step 1：定位任务

从以下位置查找任务（按优先级）：

1. `docs/tasks/<task-id>.md` 或 `docs/tasks/*.md`（包含 task-id 的文件）
2. `docs/tasks/backlog.md` 或 `docs/tasks/sprint.md` 中的条目
3. 若以上均无，提示用户提供任务描述和验收条件，手动输入后继续

从任务文档中提取：
- **任务描述**：这个任务做什么页面/功能
- **验收条件（AC）**：什么情况下算完成，特别关注 UI 交互细节
- **相关接口**：任务描述中提及的接口或功能域名称
- **相关设计稿**：是否有 Figma 链接或设计稿路径

---

## Step 2：加载相关上下文

**必读文件**：

1. **项目 CLAUDE.md**（项目根）
   提取：前端框架（React / Vue 3 / 其他）、状态管理方案、HTTP 客户端、目录结构约定、组件规范、禁止事项。
   **这是最高优先级的上下文，所有后续生成必须符合项目约定。**

2. **接口规范**（`docs/api-spec/` 或 `specs/api-specification.md`）
   查找与该任务相关的接口定义，提取：
   - 涉及的 endpoint 列表（METHOD + path）
   - 所有请求和响应字段（类型、是否必填、枚举值）
   - 分页、排序、过滤参数
   - 接口的认证要求

   若接口规范不存在或找不到与任务相关的接口，停止并提示：
   ```
   接口规范未找到或缺少与本任务相关的接口定义。
   请先运行 /review-api-spec 确认接口规范已就绪，或手动提供接口定义后继续。
   ```

**可选文件**（存在则读取）：

3. **ADR（架构决策记录）**（`docs/adr/`）
   查找与前端相关的决策记录（状态管理方案、组件库选型、路由方案等）。

4. **设计稿/原型**（`docs/design/` 或文档中注明的 Figma 链接）
   提取：页面/组件结构、交互状态、表单字段和前端校验规则。

读取完成后输出：
```
任务：<task-id> · <任务描述>
技术栈：<从 CLAUDE.md 读取：框架 / 状态管理 / HTTP 客户端>
相关接口：<N> 个（<METHOD path>, ...）
设计稿：<已找到 / 未找到>
```

---

## Step 3：生成 TypeScript 类型定义

在生成任何组件或 Service 骨架之前，先从接口规范推导对应的 TypeScript 类型定义。

类型定义路径：`src/types/api/<feature>.ts`（或项目 CLAUDE.md 中约定的路径）

生成内容包括：
- 枚举类型（使用 `as const` 对象而非 TypeScript `enum`，除非项目约定使用 `enum`）
- 接口响应类型（以 `VO` 或 `Dto` 后缀，遵循项目约定）
- 接口请求类型（以 `Params` 或 `Request` 后缀，遵循项目约定）
- 通用响应包装类型（如项目已有则引用，不重复定义）

**React / TypeScript 示例**（根据实际接口规范生成，以下为结构示例）：

```typescript
// src/types/api/<feature>.ts
// 从接口规范 <spec-file-path> 生成
// 生成时间：<date>

// 枚举值
export const <EnumName> = {
  <VALUE_A>: '<value_a>',
  <VALUE_B>: '<value_b>',
} as const;
export type <EnumName> = typeof <EnumName>[keyof typeof <EnumName>];

// 响应类型
export interface <EntityName>VO {
  id: number;
  // TODO: 根据接口规范填充字段，每个字段加注释说明类型来源
}

export interface <EntityName>ListVO {
  list: <EntityName>VO[];
  total: number;
  page: number;
  pageSize: number;
  totalPages: number;
}

// 请求类型
export interface Create<EntityName>Params {
  // TODO: 根据接口规范填充字段
}

export interface <EntityName>ListParams {
  page?: number;
  pageSize?: number;
  // TODO: 根据接口规范填充过滤和排序参数
}
```

---

## Step 4：生成骨架代码

根据 Step 2 中从项目 CLAUDE.md 读取到的技术栈，选择对应的骨架模板生成以下文件：

### 4.1 API Service 骨架

路径：`src/features/<feature>/services/<feature>.service.ts`（或项目约定路径）

```typescript
// src/features/<feature>/services/<feature>.service.ts
import { http } from '@/utils/http'; // TODO: 替换为项目实际 HTTP 客户端路径
import type {
  ApiResponse,
  // TODO: 引入 Step 3 生成的类型
} from '@/types/api/<feature>';

export const <feature>Service = {
  // TODO: 根据接口规范列出每个 endpoint 的调用函数
  // list: (params: <Entity>ListParams): Promise<ApiResponse<<Entity>ListVO>> =>
  //   http.get('/api/v1/<resource>', { params }),
};
```

### 4.2 React 项目：自定义 Hook 骨架

路径：`src/features/<feature>/hooks/use-<feature>-list.ts`（列表类任务）

```typescript
// src/features/<feature>/hooks/use-<feature>-list.ts
import { useQuery } from '@tanstack/react-query';
import { <feature>Service } from '../services/<feature>.service';
import type { <Entity>ListParams } from '@/types/api/<feature>';

export const <FEATURE>_QUERY_KEYS = {
  all: ['<feature>'] as const,
  list: (params: <Entity>ListParams) => ['<feature>', 'list', params] as const,
  detail: (id: number) => ['<feature>', 'detail', id] as const,
};

export function use<Entity>List(params: <Entity>ListParams) {
  return useQuery({
    queryKey: <FEATURE>_QUERY_KEYS.list(params),
    queryFn: () => <feature>Service.list(params),
    select: (res) => res.data,
  });
}
```

### 4.3 React 项目：组件骨架

路径：`src/features/<feature>/components/<ComponentName>.tsx`

```typescript
// src/features/<feature>/components/<ComponentName>.tsx
import type { FC } from 'react';
import { useState } from 'react';
import { use<Entity>List } from '../hooks/use-<feature>-list';

interface <ComponentName>Props {
  // TODO: 根据设计稿和需求确认需要哪些 props
}

export const <ComponentName>: FC<<ComponentName>Props> = () => {
  const [page, setPage] = useState(1);
  const { data, isLoading, isError, error } = use<Entity>List({ page, pageSize: 20 });

  // Loading 状态
  if (isLoading) {
    return <div>Loading...</div>; // TODO: 替换为项目的 Skeleton/Spinner 组件
  }

  // Error 状态
  if (isError) {
    return <div>加载失败</div>; // TODO: 替换为项目的 ErrorMessage 组件
  }

  // 空状态
  if (!data?.list?.length) {
    return <div>暂无数据</div>; // TODO: 替换为项目的 EmptyState 组件
  }

  // 正常内容
  return (
    <div>
      {/* TODO: 根据设计稿实现列表 UI */}
      {data.list.map((item) => (
        <div key={item.id}>
          {/* TODO */}
        </div>
      ))}
      {/* TODO: 分页组件，使用 data.total / data.totalPages */}
    </div>
  );
};
```

### 4.4 Vue 3 项目：Store 骨架

路径：`src/features/<feature>/stores/<feature>.store.ts`

```typescript
// src/features/<feature>/stores/<feature>.store.ts
import { defineStore } from 'pinia';
import { ref } from 'vue';
import { <feature>Service } from '../services/<feature>.service';
import type { <Entity>ListParams, <Entity>ListVO } from '@/types/api/<feature>';

export const use<Entity>Store = defineStore('<feature>', () => {
  const list = ref<<Entity>ListVO | null>(null);
  const isLoading = ref(false);
  const error = ref<string | null>(null);

  async function fetchList(params: <Entity>ListParams) {
    isLoading.value = true;
    error.value = null;
    try {
      const res = await <feature>Service.list(params);
      list.value = res.data;
    } catch (e) {
      error.value = (e as Error).message;
    } finally {
      isLoading.value = false;
    }
  }

  return { list, isLoading, error, fetchList };
});
```

### 4.5 状态管理骨架（如涉及跨组件状态）

仅当任务明确涉及跨组件共享状态时生成（例如：购物车数量、全局通知未读数）。否则不生成，避免过度设计。

生成时在文件顶部注明：
```typescript
// 此 store 用于管理 <说明跨组件共享的状态是什么>
// 如果状态只在单个组件内使用，请改用 useState / ref，删除此文件
```

---

## Step 5：输出注意事项清单

基于本任务的特定接口和验收条件，生成专属的注意事项清单（不是通用模板，每一条都应对应任务具体情况）：

```markdown
## 注意事项清单（<task-id> 专属）

### API 调用
- [ ] 所有接口调用封装到 `features/<feature>/services/<feature>.service.ts`，不直接在组件中调用
- [ ] （根据任务接口情况，列出具体需要注意的 API 调用细节）

### 类型安全
- [ ] TypeScript 类型已从接口规范生成（见 Step 3 输出），使用前确认字段是否与接口实际返回一致
- [ ] 禁止使用 `any`，枚举值使用 `as const` 对象
- [ ] （列出接口中可能为 null/undefined 的字段，提示空值判断）

### 状态处理
- [ ] 实现 Loading 状态：<说明该任务的 Loading 表现>
- [ ] 实现 Error 状态：<说明该任务的 Error 表现>
- [ ] 实现空状态：<说明该任务的 Empty 表现>
- [ ] （如有表单）按钮提交时禁用，防止重复提交

### 潜在风险
- [ ] （列出基于接口规范发现的潜在前端实现风险，例如：时间格式需确认、枚举值映射需维护等）

### 验收条件对照
- [ ] AC 1：<从任务文档提取的验收条件>
- [ ] AC 2：<从任务文档提取的验收条件>
- [ ] ...
```

---

## 输出关联文件清单

```markdown
## 本任务涉及的文件

| 操作 | 文件路径 | 说明 |
|------|---------|------|
| 新建 | `src/types/api/<feature>.ts` | TypeScript 类型定义（Step 3 生成） |
| 新建 | `src/features/<feature>/services/<feature>.service.ts` | API 调用封装（Step 4 生成） |
| 新建 | `src/features/<feature>/hooks/use-<feature>-list.ts` | 数据获取 Hook（Step 4 生成） |
| 新建 | `src/features/<feature>/components/<ComponentName>.tsx` | 组件骨架（Step 4 生成） |
| 修改 | `src/router/index.ts` | 添加新路由（如适用） |
| 复用 | `src/utils/http.ts` | 全局 HTTP 客户端（无需修改） |
```

---

## 交互式模式

不带参数执行时：

```
未指定 task-id，正在扫描任务列表...

可用任务：
1. <TASK-ID> · <任务描述>
2. <TASK-ID> · <任务描述>
...

请输入任务编号或 task-id：
```

用户输入后继续执行 Step 1-5。
