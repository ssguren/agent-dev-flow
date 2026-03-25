# /check-conventions — 检查编码规范

## 使用说明

```
/check-conventions                      # 检查 git diff（当前工作区有修改的文件）
/check-conventions <file-path>          # 检查指定文件
/check-conventions --diff               # 仅检查 git diff HEAD 中变更的行
```

示例：
```
/check-conventions
/check-conventions src/main/java/com/example/adapter/UserController.java
/check-conventions --diff
```

---

## Step 1：读取项目 CLAUDE.md，提取可检查的规范条目

按优先级从以下位置读取规范：

1. **项目 CLAUDE.md**（项目根目录或 `.claude/CLAUDE.md`）— 最高优先级
2. **角色 profile.md 中的技术栈约定**（`roles/backend/profile.md`）— 默认约定
3. **相关 ADR**（`docs/adr/`）中有明确编码约束的决策

读取完成后，构建规范条目列表，标注来源：

```
[来源: CLAUDE.md L23] 命名规范 - REST 路径使用 kebab-case 复数名词
[来源: profile.md]    响应格式 - 统一使用 ApiResponse<T>，HTTP 200
[来源: ADR-003]       依赖约束 - 不得在 domain 层引入 Spring 注解
```

若 CLAUDE.md 不存在，输出警告并改用 `roles/backend/profile.md` 的默认约定继续检查。

---

## Step 2：扫描目标文件，逐条检查规范

读取目标文件内容，对照 Step 1 提取的规范条目逐条执行检查。

---

## Step 3：必检项（无论 CLAUDE.md 是否声明，均强制检查）

### 3.1 代码中文字符检查

**检查范围**：变量名、方法名、类名、包名、注解值（如 `@RequestMapping` 路径）

以下情况**不视为违规**：
- 代码注释（`//` 或 `/* */`）中的中文
- `@DisplayName`、`@Description` 等纯文档注解中的中文
- 字符串常量（响应消息、日志描述等）中的中文，除非 CLAUDE.md 明确禁止

违规示例：
```
FAIL: 变量名包含中文字符
  位置：ArticleService.java:45
  内容：private String 标题;
  修复：改为英文命名（如 title），中文含义写在注释中
```

### 3.2 调试输出残留

检查以下模式（区分 Java / JS / Python 项目）：

**Java 项目：**
- `System.out.println`
- `System.err.println`
- `e.printStackTrace()`

**JavaScript / TypeScript 项目：**
- `console.log`
- `console.error`
- `console.warn`（生产代码中）
- `debugger`

**Python 项目：**
- `print(`（生产代码中）
- `import pdb` / `pdb.set_trace()`

违规示例：
```
FAIL: 调试输出残留
  位置：ArticleRepository.java:78
  内容：System.out.println("debug: " + article);
  修复：删除或替换为 log.debug(...)
```

### 3.3 硬编码值

检查以下类型的硬编码：

| 类型 | 检查模式 | 说明 |
|---|---|---|
| 硬编码 URL | 字符串中含 `http://` 或 `https://` | 测试文件中除外 |
| 硬编码密钥 | 变量名含 `key`、`secret`、`password`、`token` 的字符串赋值 | 高危，必须 FAIL |
| 硬编码 IP | 形如 `"192.168.x.x"` 或 `"10.x.x.x"` 的字符串 | 包括内网地址 |
| 魔法数字 | 非 0、1、-1 的数字字面量直接参与逻辑判断 | 应提取为命名常量 |

违规示例：
```
FAIL: 硬编码 URL
  位置：UserClient.java:23
  内容：String baseUrl = "http://user-service:8080";
  修复：通过 @Value("${user-service.base-url}") 注入

WARNING: 魔法数字
  位置：ArticleService.java:67
  内容：if (content.length() > 5000)
  修复：提取为常量 private static final int MAX_CONTENT_LENGTH = 5000;
```

### 3.4 命名规范（Java 项目）

| 检查项 | 规则 | 违规示例 |
|---|---|---|
| 类名 | PascalCase | `articleService`（应为 `ArticleService`） |
| 方法名 | camelCase | `Get_Article`（应为 `getArticle`） |
| 常量 | UPPER_SNAKE_CASE | `maxRetry`（应为 `MAX_RETRY`） |
| 包名 | 全小写，无下划线，无大写 | `com.example.UserService`（包名含大写） |
| REST 路径 | kebab-case，复数名词 | `/getArticle`（应为 `/articles`） |

如 CLAUDE.md 中有不同于以上规则的命名约定，以 CLAUDE.md 为准。

### 3.5 TODO 注释和注释代码

**TODO 注释：**
- 有 task ID 的 TODO 不报警（如 `// TODO: TASK-001`）
- 无 task ID 的 TODO 报 WARNING

**注释掉的代码块：**
- 超过 2 行连续注释掉的代码（Javadoc 除外）报 WARNING
- 包含关键业务逻辑的注释代码（可识别出来的）报 FAIL

**空 catch 块：**
```java
// FAIL: 空 catch 块吞掉了异常
} catch (IOException e) {}

// 不违规: 有说明原因的空 catch
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 恢复中断状态
}
```

### 3.6 分层约束（Java / DDD 架构项目）

| 违规场景 | 级别 | 说明 |
|---|---|---|
| domain 层 import `org.springframework.*` | FAIL | domain 层不得依赖 Spring |
| domain 层 import `jakarta.persistence.*` | FAIL | domain 层不得依赖 JPA |
| Controller 中包含复杂业务逻辑（if/for 超出参数校验） | WARNING | 业务逻辑应移至 Service |
| infrastructure 层被 adapter 层直接调用（跳过 application 层） | WARNING | 违反 DDD 分层原则 |

---

## Step 4：输出结果

按以下格式分三类输出，三类之间用分隔线隔开：

```markdown
## 规范检查结果

**检查文件：** ArticleService.java
**规范来源：** CLAUDE.md + roles/backend/profile.md
**检查时间：** 2025-01-01 10:00

---

### FAIL（违反强制规范，必须修改，否则不能过 Review）

| # | 位置 | 规范项 | 问题描述 | 修复建议 |
|---|---|---|---|---|
| 1 | L34 | 调试输出 | `System.out.println("debug")` | 删除或改为 `log.debug(...)` |
| 2 | L67 | 分层约束 | domain 层 import 了 `org.springframework.stereotype.Component` | 移除该 import 和注解 |

---

### WARNING（违反建议规范，建议修改，不阻塞 PR）

| # | 位置 | 规范项 | 问题描述 | 修复建议 |
|---|---|---|---|---|
| 1 | L23 | 魔法数字 | `content.length() > 5000` | 提取为命名常量 `MAX_CONTENT_LENGTH` |
| 2 | L90-95 | 注释代码 | 连续 5 行注释掉的代码 | 确认不需要后删除，或加 task-id |

---

### PASS（符合规范）

- 命名规范：类名、方法名、常量命名均符合约定
- 响应格式：所有 Controller 方法返回 ApiResponse<T>
- 无硬编码 URL 或密钥
- 无调试输出（System.out / printStackTrace）
- 无空 catch 块

---

**汇总：** 2 项 FAIL，2 项 WARNING，5 项 PASS

建议：修复 FAIL 项后再提 PR。
```

---

## 注意事项

- 测试文件（`*Test.java`、`*.spec.ts`、`*_test.go`）中**放宽**以下检查：硬编码 URL、魔法数字
- 自动生成的文件（Protobuf 生成代码、MapStruct 生成代码、Swagger 生成代码）**跳过**检查，输出提示说明跳过原因
- **不自动修改文件**，只输出问题报告，由开发者手动修复
- 每个 FAIL / WARNING 条目必须标注**具体文件名和行号**，不能只写"文件中有问题"
