# Stage 3 — Development 开发阶段

## 目标
基于接口规范和任务列表，高效实现功能，保持代码质量和团队约定一致性。

## 参与角色
- **Backend Dev**（主导）：后端实现
- **Frontend Dev**（主导）：前端实现
- **Tech Lead**（支持）：重大实现问题裁决

## 输入
- `specs/api-specification.md`
- `docs/<sprint>/implementation-plan.md`（任务列表）
- 项目的 `CLAUDE.md`（编码规范）

## 输出
- 代码（PR 形式提交）
- `pr-description.md`（每个 PR 的上下文说明）
- 必要时补充 ADR（实现阶段发现的新架构问题）

## 工作流

```
开发者开始一个任务
    ↓
/task-start <task-id>    Agent 读取任务描述 + 相关接口规范，生成实现骨架
    ↓
开发者实现              人工写业务逻辑，Agent 辅助生成样板代码
    ↓
/write-tests             Agent 基于接口规范生成单元测试骨架
    ↓
本地测试通过
    ↓
/write-pr-desc           Agent 生成 PR 描述（含上下文、变更说明、测试说明）
    ↓
提交 PR → 进入 Stage 4（Review）
```

## 命令

| 命令 | 谁运行 | 做什么 |
|------|--------|--------|
| `/task-start <id>` | Developer | Agent 读取任务上下文，生成实现骨架和注意事项清单 |
| `/write-tests` | Developer | Agent 基于当前文件的接口/函数，生成单元测试 / 集成测试骨架 |
| `/write-pr-desc` | Developer | Agent 读取 git diff，生成标准 PR 描述（问题 / 变更 / 测试 / 注意事项） |
| `/check-conventions` | Developer | Agent 检查当前代码是否符合项目 CLAUDE.md 中的编码规范 |
| `/draft-adr <topic>` | Developer | 实现中发现新架构问题时，起草 ADR 供 Tech Lead 确认 |

## 编码规范（默认约定）

以下为跨项目通用约定，具体项目可在其 CLAUDE.md 中覆盖：

- 代码中禁止中文（变量名 / 字符串 / 日志 / 异常 message）
- 每个 PR 只做一件事，不混入重构和功能
- 测试文件与实现文件同目录或对应的 test 目录
- 不提交 TODO 注释，用 Issue / 任务列表替代
- 接口变更必须同步更新接口规范文档

## 共享上下文约定（多人协作同一项目时）

当多个开发者同时在一个项目中使用 Agent 时，存在"各自 Agent 上下文不一致"的问题。
解决方式：

1. **项目 CLAUDE.md 是共享约定的单一来源**。所有 Agent 在开始工作前必须读 CLAUDE.md。
2. **接口变更走文档**，不走口头。Agent 读接口规范文档，不读"我们上次讨论了什么"。
3. **遇到 CLAUDE.md 中没有规定的情况**，不要自行决定 — 提出来，Tech Lead 确认后更新 CLAUDE.md。

## 质量门禁（提 PR 前必须满足）

- [ ] 本地测试全部通过
- [ ] 没有 `System.out.println` / `console.log` 残留调试输出
- [ ] 没有硬编码的 IP / 密码 / API Key
- [ ] PR 描述已填写（`/write-pr-desc` 生成并人工检查）
- [ ] 接口变更已同步到 `specs/api-specification.md`
