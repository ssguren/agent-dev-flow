# Command: gen-sprint-retro

**角色**：Tech Lead Agent
**用途**：基于本次 Sprint 的 Gap Registry、Bug 列表、部署记录，生成 Sprint 复盘草稿，按 `workflow/07-postmortem.md` 中定义的结构输出，供 Tech Lead 和团队在复盘会议上讨论。

**调用方式**：
```
/gen-sprint-retro
/gen-sprint-retro sprint-3
```

**参数说明**：
- `sprint-id`（可选）：Sprint 编号，例如 `sprint-3`；如未提供，从 `workflow/03-sprint-plan.md` 读取当前 Sprint 编号。

---

## Step 1：读取输入文件

按以下顺序读取，全部读取完毕后再开始分析，不允许跳过任何存在的文件。

### 必读文件

1. **复盘结构模板**
   路径：`workflow/07-postmortem.md`
   目的：获取复盘文档的结构定义，确保输出格式完全符合规范。此文件必须最先读取，后续输出必须与其结构严格对齐。

2. **Sprint 计划**
   路径：`workflow/03-sprint-plan.md`（或对应 sprint 的计划文件，如 `workflow/03-sprint-plan-sprint-3.md`）
   目的：获取 Sprint 目标原文、承诺交付的功能列表、计划时间范围、Sprint ID。

3. **Gap Registry**
   路径：`docs/gap-registry.md`
   目的：筛选出本次 Sprint 期间新增和关闭的 Gap 条目，统计 Gap 处理情况，识别 Carry-over 条目。

### 可选文件（如存在则读取，不存在则在输出中标注"未提供，需人工补充"）

4. **Bug 列表**
   路径：`docs/bugs/bugs-<sprint-id>.md`（优先），或扫描 `docs/bugs/` 目录下本 Sprint 对应的文件
   目的：分析 Bug 数量、严重程度分布（Critical / High / Medium / Low）、根因类型分布、Bug 高发区域。

5. **部署记录**
   路径：`docs/deployments/deploy-<sprint-id>.md`（优先），或扫描 `docs/deployments/` 目录下本 Sprint 对应的文件
   目的：了解部署是否顺利，是否有回滚、Hotfix 或部署窗口超时，Checklist 执行情况。

6. **任务完成记录**
   路径：`workflow/tasks-<sprint-id>.md`
   目的：对照任务列表逐条确认完成状态，计算交付完成率。若此文件不存在，则以 Sprint 计划文件中的功能列表勾选状态为准。

---

## Step 2：分析维度

读取完毕后，按以下维度进行结构化分析。分析结果作为复盘草稿各节的内容依据。**分析只基于文档中可以读到的数据，不推测、不捏造。**

### 2.1 交付完成率

- 统计 Sprint 计划中承诺交付的功能点总数。
- 统计实际完成的功能点数量（以任务列表或 Sprint 计划中的勾选状态 `[x]` 为准）。
- 计算完成率 = 完成数 / 承诺数 × 100%。
- 如果完成率低于 80%，标记为风险项，在复盘草稿中用 `⚠️ 风险` 标注。
- 列出未完成的功能点清单，每条注明原因（如果能从文档中读取到），否则标注"原因未记录，需人工补充"。

### 2.2 Gap 处理情况

- 从 Gap Registry 中筛选本 Sprint 期间新增的 Gap 条目（依据条目中的 Sprint 字段或创建日期与本 Sprint 时间范围比对）。
- 从 Gap Registry 中筛选本 Sprint 期间关闭（status: closed）的 Gap 条目。
- 统计：新增 Gap 数、已关闭 Gap 数、仍未关闭（Carry-over）Gap 数。
- 对新增 Gap 按影响范围分类：功能缺失 / 接口偏差 / 性能问题 / 其他。
- 识别重复出现的 Gap 类型（同一类问题在多个 Sprint 中均有记录），标记为"系统性问题"，在复盘草稿中重点提出。

### 2.3 Bug 分布

- 按严重程度分类：Critical / High / Medium / Low，统计各分类数量和占比。
- 按根因类型分类（从 Bug 记录中提取，例如：逻辑错误 / 边界条件遗漏 / 接口理解偏差 / 环境问题 / 其他）。
- 识别 Bug 高发区域（哪个模块或功能点 Bug 最多）。
- 如果 Bug 列表文件未提供，此部分全部标注"本 Sprint Bug 数据未提供，需人工补充"，不做任何推测。

### 2.4 部署顺利度

- 确认部署是否按计划窗口执行（有无延误）。
- 是否有回滚操作（记录回滚次数和原因）。
- 是否有 Hotfix（记录 Hotfix 数量和原因）。
- 部署前的 Checklist 执行情况（如存在于部署记录中）。
- 如果部署记录文件未提供，此部分全部标注"本 Sprint 部署记录未提供，需人工补充"，不做任何推测。

---

## Step 3：输出格式

输出路径：`docs/retro/retro-<sprint-id>.md`

**写入前必须打印以下提示并等待确认**：
```
即将写入 docs/retro/retro-<sprint-id>.md，是否确认？[y/N]
```
收到 `y` 后再执行写入，收到其他输入则终止。

输出内容严格遵循 `workflow/07-postmortem.md` 中定义的复盘结构。在此基础上，按以下模板填充分析结果：

```markdown
# Sprint <sprint-id> 复盘报告

**复盘日期**：[需人工填写]
**参与人**：[需人工填写]
**Sprint 周期**：YYYY-MM-DD 至 YYYY-MM-DD（从 Sprint 计划读取）
**主持人**：[需人工填写]

---

## 1. Sprint 目标回顾

（从 Sprint 计划中摘录 Sprint 目标原文，保持原文，不改写）

---

## 2. 交付完成率

**承诺功能点**：N 个
**实际完成**：N 个
**完成率**：XX%

### 已完成功能点

（列表）

### 未完成功能点

| 功能点 | 未完成原因 |
|--------|-----------|
| ...    | （从文档读取；读不到则写"原因未记录，需人工补充"）|

---

## 3. Gap 处理情况

**本 Sprint 新增 Gap**：N 条
**本 Sprint 关闭 Gap**：N 条
**Carry-over Gap**：N 条

### 新增 Gap 分类

| Gap ID | 标题 | 影响范围 | 状态 |
|--------|------|----------|------|
| ...    | ...  | ...      | ...  |

### 系统性问题识别

（如发现重复类型的 Gap，在此列出并描述模式；如无，写"本 Sprint 未发现系统性 Gap 模式"）

---

## 4. Bug 分布

（如 Bug 数据未提供，本节整体写"本 Sprint Bug 数据未提供，需人工补充"）

### 严重程度分布

| 级别 | 数量 | 占比 |
|------|------|------|
| Critical | | |
| High | | |
| Medium | | |
| Low | | |

### 根因分类

（列表，每条格式：根因类型 — N 个 Bug）

### Bug 高发区域

（描述哪个模块或功能点 Bug 最集中）

---

## 5. 部署情况

（如部署记录未提供，本节整体写"本 Sprint 部署记录未提供，需人工补充"）

**部署状态**：顺利 / 有问题（从记录读取）
**回滚次数**：N 次
**Hotfix 数量**：N 个
**异常说明**：（如有，逐条列出；无则写"无"）

---

## 6. 做得好的地方（What Went Well）

[需人工填写]

Agent 不填写此节。此节必须由团队在复盘会议上共同讨论填写，反映团队真实感受，不能由数据分析代替。

---

## 7. 需要改进的地方（What Didn't Go Well）

### Agent 预填（基于数据分析，供讨论参考）

（根据 Step 2 分析结果，列出客观发现的问题点，例如：完成率不足、Bug 高发区域、系统性 Gap 等。每条说明数据依据。）

### 需人工补充

[需人工填写]

（留空，复盘会议上填写团队成员的主观感受和数据之外的问题）

---

## 8. 下次怎么做（Action Items / 预防措施）

[需人工填写]

**Agent 不填写此节。** 根因分析和预防措施必须由团队共同讨论后由人来写，Agent 无法代替团队判断真正的根因。

填写格式要求（update-from-retro 命令将解析此格式）：

```
- [ ] [PREVENT] <预防措施描述>
      目标写入：Gap Registry / CLAUDE.md / 两者
      负责人：[待填写]
      截止日期：[待填写]
```

---

## 9. 遗留问题（Carry-over）

（Agent 自动从 Gap Registry 中列出本 Sprint 结束时仍未关闭的 Gap 条目）

| Gap ID | 标题 | 优先级 | 计划在哪个 Sprint 处理 |
|--------|------|--------|----------------------|
| ...    | ...  | ...    | [需人工填写]|
```

---

## 输出后的人工提示

写入文件后，在终端打印以下提示，引导人工后续操作：

```
复盘草稿已写入 docs/retro/retro-<sprint-id>.md

需要人工填写的部分：
  第 1 节  复盘日期、参与人、主持人（文件头部）
  第 6 节  做得好的地方（What Went Well）—— 团队讨论后填写
  第 7 节  需要改进的地方（人工补充部分）
  第 8 节  下次怎么做（Action Items / 预防措施）—— 最重要，格式见文件内说明
  第 9 节  Carry-over Gap 的"计划处理 Sprint"列

数据缺失提示：
  （逐条列出 Step 1 中未找到的可选文件，例如：
   - docs/bugs/bugs-<sprint-id>.md 未找到，第 4 节需人工补充 Bug 数据
   - docs/deployments/deploy-<sprint-id>.md 未找到，第 5 节需人工补充部署记录）

复盘草稿经团队确认并完成人工填写后，运行以下命令将预防措施落地：
  /update-from-retro docs/retro/retro-<sprint-id>.md
```
