# /task-start

开始一个具体开发任务前的准备工作：读取上下文、输出实现骨架和注意事项。

---

## 用法

```
/task-start <task-id>
/task-start
```

- 带 `<task-id>` 时，直接查找对应任务
- 不带参数时，进入交互式模式：列出当前任务列表供选择

---

## 执行步骤

### Step 1：定位任务

从以下位置查找任务（按优先级）：

1. `docs/tasks/<task-id>.md` 或 `docs/tasks/*.md`（包含 task-id 的文件）
2. `docs/tasks/backlog.md` 或 `docs/tasks/sprint.md` 中的条目
3. 若以上均无，提示用户提供任务描述和验收条件，手动输入后继续

提取内容：
- **任务描述**：这个任务要做什么
- **验收条件（AC）**：什么情况下算完成
- **相关功能模块**：影响哪些业务域

---

### Step 2：收集相关上下文

**接口规范**（`docs/api-spec/`）

查找与任务相关的接口定义，提取：
- 涉及的 endpoint 列表
- 请求/响应字段定义
- 权限要求

**ADR**（`docs/adr/`）

查找与任务技术方案相关的架构决策，提取：
- 分层约定
- 技术选型决策
- 特殊约束

**项目 CLAUDE.md**（项目根）

提取：
- 技术栈（语言、框架、版本）
- 命名规范
- 分层结构约定
- 响应格式约定

---

### Step 3：输出实现骨架

根据项目 CLAUDE.md 中的技术栈判断输出语言：

**Java / Spring Boot 项目输出示例：**

```java
// === adapter 层 ===
@RestController
@RequestMapping("/api/v1/articles")
@RequiredArgsConstructor
public class ArticleController {

    private final ArticleApplicationService articleApplicationService;

    @PostMapping
    public ApiResponse<ArticleVO> createArticle(
            @Valid @RequestBody CreateArticleCommand command) {
        // TODO: 委托给 ApplicationService
        throw new UnsupportedOperationException("未实现");
    }
}

// === application 层 ===
@Service
@RequiredArgsConstructor
public class ArticleApplicationService {

    private final ArticleRepository articleRepository;

    public ArticleVO createArticle(CreateArticleCommand command) {
        // TODO: 构建 domain 对象 → 校验 → 持久化 → 返回 VO
        throw new UnsupportedOperationException("未实现");
    }
}

// === domain 层 ===
public class Article {

    private ArticleId id;
    private String title;
    private ArticleStatus status;

    // 工厂方法
    public static Article create(String title, String content) {
        // TODO: 业务规则校验
        throw new UnsupportedOperationException("未实现");
    }

    // 业务行为
    public void publish() {
        // TODO: 状态机转换
        throw new UnsupportedOperationException("未实现");
    }
}

// === domain 层 - Repository 接口 ===
public interface ArticleRepository {
    void save(Article article);
    Optional<Article> findById(ArticleId id);
}

// === client 层 - 请求/响应 DTO ===
public record CreateArticleCommand(
    @NotBlank String title,
    String content
) {}

public record ArticleVO(
    Long id,
    String title,
    String status,
    Instant createdAt
) {}
```

**Node.js / TypeScript 项目输出示例：**

```typescript
// === Controller ===
@Controller('articles')
export class ArticleController {
  constructor(private readonly articleService: ArticleService) {}

  @Post()
  async createArticle(@Body() dto: CreateArticleDto): Promise<ApiResponse<ArticleVO>> {
    // TODO: 委托给 Service
    throw new Error('未实现');
  }
}

// === Service ===
@Injectable()
export class ArticleService {
  constructor(private readonly articleRepository: ArticleRepository) {}

  async createArticle(dto: CreateArticleDto): Promise<ArticleVO> {
    // TODO: 业务逻辑
    throw new Error('未实现');
  }
}

// === DTO ===
export class CreateArticleDto {
  @IsNotEmpty()
  title: string;

  content?: string;
}

export interface ArticleVO {
  id: number;
  title: string;
  status: string;
  createdAt: string;
}
```

骨架只包含关键类和方法签名，不要生成完整实现。所有方法体用 `throw new UnsupportedOperationException("未实现")` 或等价占位。

---

### Step 4：输出注意事项清单

基于读取到的上下文，生成针对本任务的 checklist：

```markdown
## 注意事项清单（本任务专属）

### 业务规则
- [ ] <从验收条件提取的规则 1>
- [ ] <从验收条件提取的规则 2>

### 权限与安全
- [ ] <接口名> 需要校验调用方是否有权限操作该资源（数据隔离：仅本人/本租户）
- [ ] 输入参数需进行 <校验类型> 校验

### 事务边界
- [ ] <操作> 涉及多表写入，需在 ApplicationService 方法上加 @Transactional
- [ ] 外部 HTTP 调用不要包含在事务内

### 测试要求
- [ ] 使用 `/write-tests` 生成测试骨架后补充用例
- [ ] 需覆盖验收条件中的 <AC 1>、<AC 2>

### 规范合规
- [ ] 使用 `/check-conventions` 在提 PR 前执行规范检查
- [ ] 响应格式使用 ApiResponse<T>，不要直接返回业务对象
```

只生成与本任务真实相关的条目，不要输出与任务无关的通用条目。

---

### Step 5：输出关联文件/模块

```markdown
## 关联文件与模块

| 类型 | 路径 | 说明 |
|---|---|---|
| Controller | `adapter/src/main/java/.../ArticleController.java` | 新建 |
| ApplicationService | `application/src/main/java/.../ArticleApplicationService.java` | 新建 |
| Domain Entity | `domain/src/main/java/.../Article.java` | 新建 |
| Repository 接口 | `domain/src/main/java/.../ArticleRepository.java` | 新建 |
| Repository 实现 | `infrastructure/src/main/java/.../ArticleRepositoryImpl.java` | 新建 |
| DTO | `client/src/main/java/.../CreateArticleCommand.java` | 新建 |
| 现有模块（可能影响） | `application/src/main/java/.../UserApplicationService.java` | 需要调用用户权限校验 |
```

---

## 交互式模式

不带参数执行时：

```
未指定 task-id，正在扫描任务列表...

可用任务：
1. TASK-001 · 用户注册接口
2. TASK-002 · 文章发布功能
3. TASK-003 · 评论列表分页

请输入任务编号或 task-id：
```

用户输入后继续执行 Step 1-5。
