# Issue 看板与详情页面设计

覆盖页面：

- `/projects/:pid/issues`：Issues.vue
- `/projects/:pid/issues/:iid`：IssueDetail.vue

## Issue 看板

### 页面目标

按状态管理 AI 编码工作流，让用户快速发现待审核、执行中、等待回答、失败和待 PR 的 Issue。

### 顶部区域

标题栏：

- 标题：Issue 看板。
- 副标题：显示当前项目、打开 Issue 数和待回答问询数。
- 主按钮：新建 Issue。
- 次按钮：导入 Git Issues。

工具栏：

- 搜索：标题、描述、标签。
- 状态筛选。
- 优先级筛选。
- 分类筛选：bug / feature / refactor / docs / infra / test。
- AI 工具筛选：opencode / codex / claude-code。
- 来源筛选：flowcode / github / gitlab / gitea。
- 视图切换：看板 / 列表。

### 看板列

看板列按 FSM 状态聚合，避免 15 个状态横向过宽。

| 列 | 包含状态 | 列宽 | 重点 |
|----|----------|------|------|
| Draft | draft | 300px | 待审核。 |
| Approved / Queued | approved, queued | 300px | 可生成计划或入队。 |
| Running | running, paused | 320px | 正在执行或暂停。 |
| Waiting Input | waiting_input | 320px | AI 等待回答。 |
| Done | done | 300px | 可创建 PR / 重试 / 关闭。 |
| PR | pr_ready, pr_created | 320px | 待创建 PR 或 PR 已创建。 |
| 已结束 / 可重开 | deployed, closed, rejected, cancelled, failed | 320px | 终态、失败、可重开。 |

列头：

- 状态名。
- 数量。
- WIP 提示：running / queued 可显示项目并发限制。

### Issue 卡片

卡片宽度填满列，最小高度 128px，padding 12px。

内容顺序：

1. 标题，最多两行。
2. 状态、优先级、分类、来源徽章。
3. AI 分析摘要行：
   - `IsBug`。
   - `NeedsCodeChange`。
   - 分类来源与置信度。
4. 元信息：
   - assignee。
   - AI 工具。
   - pendingQuestionCount。
   - 更新时间。
5. 下一步动作：
   - 审批。
   - 执行。
   - 回答。
   - 查看日志。
   - 创建 PR。

卡片状态：

- `waiting_input`：左侧 3px 黄色边，顶部显示待回答数量。
- `running`：显示实时小圆点和已运行时间。
- `failed`：显示错误摘要，动作“重试”。
- `pr_created`：显示 PR 编号和外链。

### 拖拽与状态守卫

支持拖拽改变状态，但只允许状态机可用动作。

交互：

- 拖拽到不可用列时列头变灰，并显示 tooltip：“当前状态不能直接流转到此列”。
- 放到可用列后弹出确认，明确将触发的动作。
- 危险动作如 close / reject / cancel 必须二次确认。

键盘替代：

- 卡片操作菜单提供“更改状态”，只列出当前可用动作。

### 新建 Issue

使用右侧抽屉，宽 640px。

字段：

| 字段 | 控件 | 规则 |
|------|------|------|
| 标题 | 输入框 | 必填，最多 500 字。 |
| 描述 | Markdown 编辑器 | 最小高度 260px。 |
| 优先级 | 单选 | p0 / p1 / p2 / p3。 |
| AI 工具 | 工具卡片单选 | opencode / codex / claude-code。 |
| Skill | 多选 | 从已发布或已安装 Skill 中选择。 |
| 预估工时 | NumberInput | 可选。 |
| 指派人 | Select | 可选。 |
| 标签 | 多选或创建 | 使用项目标签。 |

提交后状态为 `draft`，并触发自动分析。创建成功后抽屉关闭，卡片插入 Draft 列顶部。

### 列表视图

适合筛选和批量扫描。列：

- 标题。
- 状态。
- 优先级。
- 分类。
- AI 分析。
- AI 工具。
- 指派人。
- PR。
- 更新时间。
- 操作。

## Issue 详情

### 页面目标

承载一次 Issue 的完整生命周期：描述、AI 分析、审批、计划、执行日志、问询、评论、测试、历史、PR。

### 布局

桌面端两栏：

```
左侧主内容：标题 + Tabs
右侧工作流栏：状态、下一步动作、元信息、PR/Git 摘要
```

- 右侧栏宽 360px。
- 主内容最小宽 720px。
- 移动端右侧栏折叠到标题下方，Tabs 单列。

### 详情头部

包含：

- Issue 标题。
- 状态、优先级、分类、来源。
- ID、创建者、创建时间。
- 若来自 Git 平台，显示原始 Issue 外链。

标题可内联编辑：

- 点击编辑按钮进入标题输入。
- 保存调用 `PUT /api/v1/issues/:id`。
- running / queued 状态下仍允许补充描述，但 AI 工具字段只在 approved 前可改。

### 右侧工作流栏

顶部显示当前状态和下一步动作。

动作映射：

| 当前状态 | 主动作 | 次动作 |
|----------|--------|--------|
| draft | 审批通过 | 驳回、编辑、删除 |
| approved | 执行 | 生成/重新生成计划、编辑 |
| queued | 取消执行 | 查看队列 |
| running | 暂停 | 取消、查看日志 |
| paused | 恢复执行 | 取消 |
| waiting_input | 回答问询 | 跳过、取消 |
| done | 创建 PR | 重试、关闭 |
| failed | 重试 | 编辑、关闭 |
| cancelled | 重试 | 关闭 |
| pr_ready | 创建 PR | - |
| pr_created | 标记已部署 | 重开 |
| deployed | 关闭 | 重开 |
| closed | 重新打开 | - |
| rejected | 重新打开 | 编辑、删除、关闭 |

危险操作：

- cancel、close、delete、reject 需要确认。
- fast_exec 只在管理员且符合条件时显示在更多菜单中。

### Tabs

#### 概览

区块：

- 描述 Markdown。
- AI 分析：isBug、needsCodeChange、analysisReason、analysisCompleted。
- 分类：category、source、confidence、classifiedAt。
- 关联需求。
- 标签与指派人。

#### 执行计划

显示：

- planSummary。
- planDetail Markdown。
- planGeneratedAt。
- 操作：生成计划 / 重新生成计划。

未生成时：

- approved 状态显示主操作“生成执行计划”。
- 其他状态说明计划会在执行前自动补齐或当前状态不允许生成。

#### 执行日志

嵌入执行日志组件，详见 `07-AI问询执行日志与历史.md`。

#### AI 问询

显示当前 Issue 的问询列表摘要：

- pending 问询置顶。
- 每条显示问题、上下文、选项、回答状态。
- 主操作跳转 `/questions` 或在当前 Tab 直接回答。

#### 评论

评论支持 Markdown 与嵌套回复。

结构：

- 评论输入框固定在评论列表上方。
- 顶层评论按时间升序。
- 回复缩进 24px，最多展示两层，继续回复仍归到该线程。
- 作者可编辑/删除自己的评论。
- 软删除且有子回复时保留“此评论已删除”占位。

#### 测试用例

列出关联 TestCase：

- 名称。
- 审核状态。
- 生成状态。
- 最近运行结果。
- 覆盖率。
- 操作：查看、运行。

入口：新建测试用例，进入测试用例创建抽屉。

#### 历史

显示 Issue 字段变更历史摘要，完整设计见 `07-AI问询执行日志与历史.md`。

### PR 与 Git 摘要

右侧栏底部展示：

- gitBranch。
- gitCommitSha。
- gitPRUrl。
- PR 状态。

无 PR 时根据状态显示创建入口。

### 状态冲突处理

如果用户操作时后端返回状态冲突：

- 弹出轻提示：当前状态已更新。
- 刷新 Issue 详情。
- 在右侧栏显示新状态与可用动作。

### 交互补充

- 看板卡片点击正文区域进入详情；点击行内动作只触发该动作，不进入详情。
- 看板横向滚动时列头固定在顶部，方便用户理解当前列。
- Issue 详情页执行主动作后，右侧工作流栏立即进入 loading，主内容区不跳转，成功后再刷新状态。
- 执行、暂停、恢复、取消等动作完成后，自动在详情页顶部显示本次动作的时间和操作者。
- 评论提交失败时保留输入内容，并在评论框上方显示错误条。
- 若 Issue 有 pending 问询，详情页默认将 AI 问询 Tab 标为高亮，但不强制切换用户当前 Tab。
