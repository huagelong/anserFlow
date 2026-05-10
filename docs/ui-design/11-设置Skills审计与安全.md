# 设置、Skills、审计与安全页面设计

覆盖页面：

- `/me`：Profile.vue
- `/projects/:pid/settings`：Settings.vue
- `/projects/:pid/skills`：Skills.vue
- `/projects/:pid/audit-log`：AuditLog.vue
- 相关安全入口：API Key、登录日志、异常告警

## 个人中心页

### 页面目标

集中管理当前用户自己的资料、密码、设备和登录安全信息。该页是用户级入口，不承载项目配置和组织配置。

### 页面结构

- 左侧二级导航宽 220px。
- 右侧内容区最大 920px。
- 页面头部显示“个人中心”标题与“保存资料”按钮。

二级导航：

- 基础资料。
- 安全与密码。
- 已登录设备。
- 登录历史。
- 个人审计。

### 基础资料

字段：

- 用户名，只读。
- 昵称，可编辑。
- 邮箱，只读。
- 界面语言。
- 头像地址。

数据来源与保存：

- 进入页面调用 `GET /api/v1/auth/me`。
- 保存资料调用 `PUT /api/v1/auth/me`。
- 用户名在账号创建后冻结，成员本人和管理员都不能修改。

### 安全与密码

字段：

- 当前密码。
- 新密码。
- 确认新密码。
- 修改后退出其他设备：布尔开关，默认开启。

交互：

- 单独保存，不与基础资料混用同一次提交。
- 新密码规则在输入框下方或卡片顶部提示，至少满足长度和复杂度要求。
- 提交成功后显示 toast；若勾选退出其他设备，则保留当前设备并吊销其他会话。

### 已登录设备

- 展示最近活跃设备摘要，至少包含设备名、来源、系统架构、最近活跃时间、是否当前设备。
- 提供“前往我的设备”入口，对接当前用户设备列表页或抽屉。

### 登录历史

- 展示最近登录尝试摘要。
- 失败记录单独使用危险色状态，不整行染红。
- 提供“查看全部”入口，对接 `GET /api/v1/users/me/login-logs`。

### 个人审计

- 展示当前用户最近敏感操作摘要。
- 提供“查看全部”入口，对接 `GET /api/v1/users/me/audit-logs`。

## 项目设置页

### 页面目标

集中配置项目级能力：AI 工具、Git 凭证、标签、API Key、自动执行策略、测试策略和监控去重窗口。

### 布局

左侧设置导航宽 220px，右侧内容区最大 920px。

Tabs / 菜单：

- 基本信息。
- 自动化。
- AI 工具。
- Git 凭证。
- 标签。
- API Key。

### 基本信息

字段：

- 项目名称。
- slug。
- 描述 Markdown。
- Git provider、repoUrl、defaultBranch 的摘要。

保存调用 `PUT /api/v1/projects/:id`。

删除项目：

- 放在页面底部危险区。
- 二次确认，输入项目 slug。
- 调用软删除接口。

### 自动化设置

只展示 `Project.settings` 定义项：

| 设置 | 控件 |
|------|------|
| autoClassify | Switch |
| autoExecute | Switch |
| autoCreatePR | Switch |
| requireApproval | Switch |
| testRequireApproval | Switch |
| enableAIQuestions | Switch |
| aiQuestionTimeout | NumberInput，秒 |
| maxRetries | NumberInput |
| maxConcurrentExecutions | NumberInput |
| monitorDedupWindow | NumberInput，秒 |
| branchNaming | 输入框 |
| prTemplate | Markdown 编辑器 |
| testFramework | Select |
| testCoverageTarget | Slider + NumberInput |

保存后顶部显示成功条，设置立即生效。

### AI 工具配置

列表卡片：

- 工具名：opencode / codex / claude-code。
- endpoint。
- model。
- 状态：未配置 / 已配置 / 连接失败。
- 最近测试时间。

新增/编辑抽屉：

- toolName。
- endpoint。
- apiKey，安全输入。
- model。
- maxTokens。
- customInstructions。

动作：

- 测试连接。
- 保存。
- 删除。

测试连接中显示 loading，不保存表单。

### 标签管理

表格：

- 名称。
- 颜色。
- 使用次数。
- 创建时间。
- 操作。

新增/编辑：

- 名称。
- 颜色选择器，提供预设色，不允许过浅导致文字不可读。

删除：

- 二次确认。

### API Key

展示组织级 API Key 管理，并在项目配置区展示 `.flowcode/config.yaml` 中项目级 key 的同步摘要；不提供文档未定义的项目 key 轮换界面。

- 列表不展示完整 key，只显示前后缀。
- 创建后仅弹窗展示一次完整 key。
- 支持吊销。

字段：

- 名称。
- 创建时间。
- 最近使用。
- 权限范围摘要。
- 操作：复制前后缀、吊销。

## Skills 管理页

### 页面目标

管理组织内部 Skill：搜索、创建、编辑、发布、废弃，并将 published Skills 同步到项目仓库。

### 页面结构

标题栏：

- 标题：Skills。
- 主按钮：创建 Skill。
- 次按钮：同步到项目。

工具栏：

- 搜索名称 / slug / 内容。
- 状态筛选：draft / published / deprecated。
- 分类筛选。

### Skill 列表

卡片或表格均可。桌面默认表格：

| 列 | 内容 |
|----|------|
| Skill | 名称、slug、用途摘要。 |
| 状态 | published / draft / deprecated。 |
| 信任等级 | Trusted / Community / Untrusted。 |
| 更新时间 | 时间。 |
| 操作 | 查看、编辑、发布、废弃、删除。 |

### Skill 编辑器

使用左右双栏：

- 左：Markdown 编辑器。
- 右：实时预览。

宽屏下双栏各 50%；窄屏用 Tabs。

必填章节检测：

- `# Skill: {name}`。
- `## 用途`。
- `## 何时触发`。
- `## API 调用`。
- `## 响应`。

缺失章节在编辑器顶部显示校验条。

### 发布与废弃

发布：

- 弹窗展示校验结果。
- 确认后调用 publish。

废弃：

- 二次确认。
- deprecated 状态保留只读查看，不默认删除。

同步到项目：

- 调用 `POST /api/v1/projects/:pid/skills/sync`。
- 结果显示写入 `.flowcode/config.yaml` 和 `.flowcode/skills/*.md` 的摘要。
- 同步失败显示错误并允许重试。

## 审计日志页

### 页面目标

追踪 Web、CLI、API、Webhook 触发的敏感操作，以及登录历史和异常告警。

### 顶部 Tabs

- 操作审计。
- 登录日志。
- 异常告警。

### 操作审计

工具栏：

- action。
- source：cli / web / api / webhook。
- targetType：issue / test_case / project / skill。
- userId。
- 时间范围。

表格列：

- 时间。
- 操作。
- 来源。
- 操作人。
- 目标。
- 详情。
- IP / deviceId。

详情抽屉：

- changes JSON diff。
- target 链接。
- 设备信息。

### 登录日志

工具栏：

- status：success / failed。
- source：cli / web。
- userId。
- 时间范围。

表格列：

- 时间。
- 用户。
- 状态。
- 来源。
- IP。
- deviceName。
- failReason。

失败登录行使用危险色徽章，但不整行染红。

### 异常告警

展示近 24 小时异常：

- brute_force。
- new_device。
- 异地登录。

卡片内容：

- severity。
- 类型。
- IP / 设备。
- 影响用户。
- firstSeen / lastSeen。

操作：

- 查看登录日志。
- 进入设备管理。

## 安全交互规范

- 密钥类字段默认隐藏。
- 密码修改成功后，如选择退出其他设备，必须明确提示“当前设备保留，其他设备已失效”。
- 创建 API Key 后只展示一次完整内容。
- SSH 私钥和 accessToken 绝不在列表中显示。
- 吊销设备、删除凭证、吊销 key、删除项目、删除 Skill 都使用危险确认。
- 审计日志不可删除、不可编辑。

## 空态与错误态

- 无 AI 工具：显示“添加 AI 工具配置”。
- 无 Skill：显示“创建第一个 Skill”。
- 无审计日志：显示筛选范围内暂无记录。
- 测试连接失败：显示 endpoint、错误消息和重试按钮。

## 交互补充

- 设置页切换左侧菜单时保留当前滚动位置；有未保存表单时先触发离开确认。
- AI 工具测试连接不保存表单；连接成功只更新当前抽屉内的测试状态。
- 创建 API Key 后完整 key 只在一次性弹窗显示；关闭弹窗后列表只展示前后缀。
- Skill 编辑器发布前先执行必填章节检查；检查不通过时定位到第一个缺失章节。
- Skills 同步运行中时，同步按钮 loading，并在完成后展示写入文件摘要和 Git 提交结果。
- 审计日志详情抽屉只读，任何 changes 内容都不提供编辑入口。
