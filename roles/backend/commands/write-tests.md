# /write-tests — 生成测试骨架

## 使用说明

```
/write-tests                    # 为当前打开的文件生成测试骨架
/write-tests <file-path>        # 为指定路径的文件生成测试骨架
```

示例：
```
/write-tests
/write-tests src/main/java/com/example/domain/User.java
/write-tests src/main/java/com/example/adapter/controller/UserController.java
```

---

## Step 1：读取目标文件，识别被测对象类型

读取目标文件，从以下维度判断被测对象类型：

| 判断依据 | 被测类型 | 测试策略 |
|---|---|---|
| 位于 `domain/` 包，无 Spring 注解，含业务方法 | Domain 层（实体 / 值对象 / 领域服务） | 纯单元测试，无 Spring 上下文 |
| 实现 `Repository` 接口，位于 `infrastructure/` | Repository 层 | `@DataMongoTest` 或 `@DataJpaTest` |
| 带 `@RestController` 注解 | Controller 层 | `@WebMvcTest`，Mock Service 层 |
| 带 `@Service` 注解，位于 `application/` | ApplicationService 层 | Mockito 单元测试，视依赖决定是否集成 |
| 带 `@Service` 注解，位于 `domain/` | DomainService 层 | 纯单元测试 |

识别完成后输出：
```
目标文件：<文件路径>
被测类型：<Domain 实体 / Service / Repository / Controller>
被测类名：<ClassName>
Public 方法列表：
  - methodName(paramType): returnType
  - ...
```

若无法从上述规则判断，停下来询问用户确认后继续。

---

## Step 2：根据类型选择测试策略

### Domain 层（实体 / 值对象 / 领域服务）

**策略：纯 JUnit，不启动 Spring 容器**

- 测试类不加任何 Spring 注解
- 依赖：`junit-jupiter` + `assertj-core`
- 直接 `new` 对象构造测试数据，不 Mock 任何东西
- 只测含业务逻辑的方法，不测 getter / setter
- 测试文件放在 `src/test/java`，包路径与被测类保持一致

### Repository 层

**策略：Spring 切片测试注解，启动最小 DB 上下文**

- MongoDB：`@DataMongoTest` + `@ExtendWith(SpringExtension.class)`
- JPA / MySQL：`@DataJpaTest`（默认 H2）或 `@AutoConfigureTestDatabase(replace = NONE)` + Testcontainers
- 测试数据在每个 `@Test` 的 Arrange 阶段通过 Repository 的 `save()` 写入
- `@AfterEach` 调用 `deleteAll()` 清理，保证用例隔离
- 只测数据库操作（保存、查询、更新、删除），不测业务逻辑

### Controller 层

**策略：`@WebMvcTest`，Mock Service 层，专注 HTTP 行为**

- 注解：`@WebMvcTest(XxxController.class)`
- Service 层通过 `@MockBean` 注入
- 测试关注点（不测业务逻辑，业务逻辑由 Mock 控制）：
  - HTTP 状态码是否正确（200 / 201 / 400 / 401 / 403 / 404）
  - 请求体校验（缺少必填字段时返回 400）
  - 响应体结构（字段名、类型、嵌套层级）
  - 路径参数、查询参数传递是否正确

### Service / Application 层

**策略：根据外部依赖数量决定**

- 无外部依赖（纯内存计算）：纯 JUnit，不启动容器
- 有 Repository / 外部 API 依赖：
  - 单元测试优先：Mockito Mock 所有依赖，测试编排逻辑和异常处理
  - 集成测试（可选）：`@SpringBootTest` + Testcontainers，只覆盖最核心的 1-2 条 happy path

---

## Step 3：为每个 public 方法生成测试骨架

### 骨架规范

每个测试方法必须包含以下四个要素：

**1. `@DisplayName` 中文描述**

格式：`"当 <前置条件> 时，应 <预期行为>"`

示例：
- `"当用户不存在时，应抛出 UserNotFoundException"`
- `"当邮箱已被注册时，应返回 409 Conflict"`
- `"当传入空列表时，应返回空结果而非报错"`

**2. AAA 结构注释**

```java
// Arrange — 准备测试数据和 Mock 行为
// Act     — 执行被测方法
// Assert  — 验证结果
```

**3. 关键断言占位**

不写 `assertTrue(true)` 或 `assertNotNull(result)` 这类无信息量的断言。断言需体现具体意图：

```java
// Assert
assertThat(result.getId()).isEqualTo(expectedId);
assertThat(result.getStatus()).isEqualTo(UserStatus.ACTIVE);
// TODO: 填充具体预期值
```

**4. 边界情况覆盖**

每个 public 方法至少生成以下场景（不适用的跳过）：

| 边界类型 | 适用场景 | 测试描述示例 |
|---|---|---|
| Happy path | 所有方法 | 当输入合法时，应返回预期结果 |
| null 输入 | 参数可能为 null | 当传入 null 时，应抛出 IllegalArgumentException |
| 空集合 | 参数为 List / Set | 当传入空列表时，应返回空结果而非报错 |
| 资源不存在 | 按 ID 查询 / 操作 | 当 ID 对应资源不存在时，应抛出 NotFoundException |
| 无权限 | 涉及授权 | 当用户无管理员权限时，应抛出 AccessDeniedException |
| 幂等性 | 创建 / 更新操作 | 当重复调用时，结果应与第一次一致 |
| 超长字符串 | String 参数有长度限制 | 当名称超过最大长度时，应抛出校验异常 |

---

## 输出示例

```java
@DisplayName("UserService 单元测试")
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
    }

    // ===== createUser =====

    @Test
    @DisplayName("当用户信息合法时，应成功创建用户并返回用户 ID")
    void createUser_whenValidInput_shouldReturnUserId() {
        // Arrange
        CreateUserCommand command = // TODO: 构造合法的 CreateUserCommand
        given(userRepository.existsByEmail(anyString())).willReturn(false);
        given(userRepository.save(any())).willAnswer(inv -> inv.getArgument(0));

        // Act
        String userId = userService.createUser(command);

        // Assert
        assertThat(userId).isNotBlank();
        verify(userRepository, times(1)).save(any());
        // TODO: 验证返回的 userId 格式是否符合预期
    }

    @Test
    @DisplayName("当邮箱已被注册时，应抛出 DuplicateEmailException")
    void createUser_whenEmailAlreadyExists_shouldThrowDuplicateEmailException() {
        // Arrange
        CreateUserCommand command = // TODO: 构造邮箱已存在的场景
        given(userRepository.existsByEmail(anyString())).willReturn(true);

        // Act & Assert
        assertThatThrownBy(() -> userService.createUser(command))
            .isInstanceOf(DuplicateEmailException.class);
            // TODO: 验证异常 message 是否包含邮箱信息
    }

    @Test
    @DisplayName("当 command 为 null 时，应抛出 IllegalArgumentException")
    void createUser_whenCommandIsNull_shouldThrowIllegalArgumentException() {
        // Arrange — null 输入，无需额外准备

        // Act & Assert
        assertThatThrownBy(() -> userService.createUser(null))
            .isInstanceOf(IllegalArgumentException.class);
    }

    // ===== getUserById =====

    @Test
    @DisplayName("当用户存在时，应返回用户详情")
    void getUserById_whenUserExists_shouldReturnUserDetail() {
        // Arrange
        String userId = "test-user-id";
        User mockUser = // TODO: 构造 User 对象
        given(userRepository.findById(userId)).willReturn(Optional.of(mockUser));

        // Act
        UserDetail result = userService.getUserById(userId);

        // Assert
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(userId);
        // TODO: 验证其他字段映射是否正确
    }

    @Test
    @DisplayName("当用户不存在时，应抛出 UserNotFoundException")
    void getUserById_whenUserNotFound_shouldThrowUserNotFoundException() {
        // Arrange
        String nonExistentId = "non-existent-id";
        given(userRepository.findById(nonExistentId)).willReturn(Optional.empty());

        // Act & Assert
        assertThatThrownBy(() -> userService.getUserById(nonExistentId))
            .isInstanceOf(UserNotFoundException.class);
    }

    /*
     * 待补充的边界情况测试（请根据业务规则逐一实现）：
     *
     * 输入边界：
     * - [ ] null userId：getUserById(null) 应如何处理？
     * - [ ] 空字符串 userId：getUserById("") 的行为
     *
     * 权限边界：
     * - [ ] 无权限用户操作他人资源（如适用）
     *
     * 状态边界：
     * - [ ] 已软删除用户的查询行为
     * - [ ] 并发创建同邮箱用户（如有唯一约束，需测试并发场景）
     */
}
```

---

## 输出位置

测试文件输出到与源文件对应的测试目录：

- 源文件：`src/main/java/com/example/adapter/ArticleController.java`
- 测试文件：`src/test/java/com/example/adapter/ArticleControllerTest.java`

若测试文件已存在，输出新增的测试方法骨架，提示用户手动合并，不覆盖已有测试。

---

## 禁止事项

- **不要**生成 `assertTrue(true)` 或 `assertNotNull(result)` 这类无信息量的断言
- **不要**在 `@DisplayName` 中写英文，使用中文描述测试意图
- **不要**在一个测试方法中测试多个场景（一个方法一个场景）
- **不要**硬编码测试数据库连接字符串，使用 H2 或 Testcontainers
- **不要**跳过 AAA 结构，每个测试方法必须有 `// Arrange`、`// Act`、`// Assert` 注释
