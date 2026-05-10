# Git 与 PR 管理设计

覆盖页面与区域：

- `/projects/:pid/prs`：PullRequests.vue
- `/projects/:pid/settings` 中的 Git 凭证、Git 状态、导入 Git Issues
- IssueDetail.vue 中的 PR 摘要与手动创建 PR

## PR 列表页

### 页面目标

集中查看项目所有关联 PR 的最新状态，并快速回到对应 Issue。

### 顶部区域

标题栏：

- 标题：PR 管理。
- 副标题：通过 GitProvider 实时获取状态。
- 右侧刷新按钮。

工具栏：

- 状态筛选：open / merged / closed。
- 关联 Issue 搜索。
- 排序：createdAt / updatedAt。
- 顺序：asc / desc。

### PR 表格

| 列 | 宽度 | 内容 |
|----|------|------|
| PR | 320px | 标题、编号、外链。 |
| 状态 | 112px | open / merged / closed 徽章。 |
| 分支 | 260px | sourceBranch → targetBranch。 |
| 关联 Issue | 280px | Issue 标题、状态。 |
| 更新时间 | 160px | 相对时间。 |
| 操作 | 120px | 打开、查看 Issue。 |

行点击默认进入 Issue 详情；外链图标打开 Git 平台 PR。

空态：

- 无 PR：说明“执行完成并创建 PR 后会显示在这里”，提供跳转 Issue 看板。

## Issue 详情中的 PR 动作

状态为 `done` 或 `pr_ready` 时：

- 主动作：创建 PR。
- 弹窗字段：
  - 标题，默认使用 Issue 标题。
  - sourceBranch，默认使用 gitBranch。
  - targetBranch，默认 defaultBranch。
  - 描述预览，使用 PR 模板。

状态为 `pr_created`：

- 显示 PR URL、编号、状态。
- 提供“标记已部署”动作。

## Git 凭证设置

### 页面位置

在项目设置的 Git Tab 中。

### 页面结构

```
Git 状态卡
凭证列表
新增凭证抽屉
Git URL 校验区
手动导入 Git Issues
```

### Git 状态卡

展示：

- provider。
- repoId。
- gitRepoUrl。
- defaultBranch。
- gitVerifiedAt。
- 当前绑定凭证。

状态：

- 已连接：绿色。
- 未配置：灰色。
- 校验失败：红色，显示错误详情。

动作：

- 重新校验。
- 更换凭证。
- 编辑仓库地址。

### 凭证列表

表格列：

- 名称。
- provider。
- authMethod：http / ssh。
- tokenType 或 fingerprint。
- expiresAt。
- lastUsedAt。
- createdAt。
- 操作：设为项目凭证、删除。

安全：

- 不展示 accessToken 和 sshPrivateKey 明文。
- 删除凭证需要二次确认。

### 新增凭证

抽屉宽 640px。

字段：

| 字段 | 控件 |
|------|------|
| 名称 | 输入框 |
| provider | Select：github / gitlab / gitea |
| authMethod | 分段控件：HTTP PAT / SSH |
| accessToken | 密码输入框，仅 HTTP |
| tokenType | Select，默认 pat |
| sshPrivateKey | 多行安全输入，仅 SSH |
| sshPassphrase | 密码输入框，可选 |

保存后展示 SSH fingerprint 或凭证摘要。

## Git URL 校验

用于创建项目和项目设置。

控件：

- URL 输入。
- 协议分段：http / ssh。
- 凭证选择。
- 校验按钮。

校验成功：

- 显示 reachable、repoId、defaultBranch、branches。
- 分支选择器启用。

校验失败：

- 显示 errorCode 和 errorDetail。
- 不允许保存项目 Git 配置。

## 手动导入 Git Issues

### 边界

只支持用户手动触发导入，不做自动同步、Webhook 监听或定时任务。

### 入口

在 Issue 看板工具栏和项目设置 Git Tab 各提供入口。

### 弹窗

宽 560px。

字段：

- state：open / closed / all，默认 open。
- since：日期时间，可选。

提示：

- 已导入 Issue 会跳过。
- 导入后由 flowcode 独立管理，不回写 Git 平台。

### 导入状态

触发后弹窗切换为任务状态：

- queued。
- running。
- completed。
- failed。

结果表：

- number。
- title。
- action：imported / skipped。
- flowcodeId。
- reason。

完成后提供：

- 查看导入的 Issue。
- 关闭。

## 错误态

| 错误 | UI 处理 |
|------|---------|
| 仓库不可达 | 红色错误条，展示 errorDetail。 |
| 凭证过期 | 在凭证行显示警告，提示更新凭证。 |
| provider 不支持 | 禁止选择，不展示未定义 provider。 |
| 导入任务失败 | 结果页显示失败原因，允许重试同一参数。 |
