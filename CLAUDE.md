# Agent Dev Flow — CLAUDE.md

## 铁律（所有 Agent 必须遵守）

1. **Agent 只生成和分析，人来确认和决策。** 任何影响他人工作的结论，必须有人工 confirm 才能生效。
2. **代码中禁止中文**（变量名 / 字符串 / 日志 / 异常 message）。注释、`@DisplayName`、文档文件除外。
3. **不要过度设计**。只生成当前阶段需要的内容，不为假设的未来需求预埋抽象。
4. **每次 Agent 操作后输出行动摘要**，包括：修改了什么文件、下一步是什么、需要人确认什么。

## 角色与责任矩阵

| 角色 | 负责确认 | Agent 协助内容 |
|------|---------|--------------|
| PM | 需求范围、优先级、GAP 决策 | 需求翻译、Gap 分析、Sprint 规划草稿 |
| Tech Lead | 架构决策（ADR）、技术方向 | 方案对比、风险评估、ADR 起草 |
| Backend Dev | 接口设计、实现方案 | 代码生成、接口文档、单元测试 |
| Frontend Dev | 组件设计、交互实现 | 组件骨架、类型定义、联调协议 |
| QA | 测试计划、上线验收 | 测试用例生成、边界分析、回归清单 |
| DevOps | 部署方案、上线决策 | 部署清单生成、配置检查、回滚方案 |

## 文档流转规则

每个阶段的**输出文档**是下一个阶段的**输入上下文**。Agent 在任意阶段工作时，必须先读取上游阶段的输出文档。

```
需求文档（PM）
    ↓ 输入
Gap Registry + ADR（Tech Lead）
    ↓ 输入
接口规范 + 实现计划（Backend / Frontend）
    ↓ 输入
PR Description（Developer）
    ↓ 输入
测试计划（QA）
    ↓ 输入
部署清单（DevOps）
    ↓ 输入
复盘报告（全员）
```

## 状态约定

所有文档中的待确认项使用统一状态标记：

| 标记 | 含义 |
|------|------|
| `status: raised` | 已提出，待 Agent 分析 |
| `status: be_reviewed` | BE Agent 已分析 |
| `status: pm_reviewed` | PM Agent 已翻译 |
| `status: confirmed` | **人工确认，决策已定** |
| `status: closed` | 已实现，归档 |

## 命令目录

所有命令定义在 `roles/<role>/commands/` 下，按角色分目录。共享命令在 `roles/shared/commands/`。

| 角色 | 命令路径 |
|------|---------|
| PM | `roles/pm/commands/` |
| Tech Lead | `roles/tech-lead/commands/` |
| Backend | `roles/backend/commands/` |
| Frontend | `roles/frontend/commands/` |
| QA | `roles/qa/commands/` |
| DevOps | `roles/devops/commands/` |
| 共享（多角色通用） | `roles/shared/commands/` |

详见各角色的 `profile.md` 和各阶段 workflow 文档。
