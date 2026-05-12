# 04 - API 设计

## 4.1 设计原则

- **RESTful 风格**，资源路径清晰
- **统一响应格式**：`{ code, data, message, timestamp }`
- **WebSocket** 用于执行日志实时推送
- **分页**统一使用 cursor-based (对看板场景更友好)

## 4.2 统一响应格式

```json
{
  "code": 0,
  "data": {},
  "message": "ok",
  "timestamp": 1715000000000
}
```

错误码规范：

| code | 含义 |
|------|------|
| 0 | 成功 |
| 40001 | 参数校验失败 |
| 40100 | 未登录 |
| 40300 | 无权限 |
| 40400 | 资源不存在 |
| 40900 | 状态冲突（如重复操作） |
| 50000 | 服务内部错误 |
| 50001 | AI 工具调用失败 |
| 50002 | Git 操作失败 |

## 4.3 接口清单

### 4.3.0 健康检查

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/health` | 服务健康检查（Docker healthcheck 使用） |

> 返回 `{ "code": 0, "data": { "status": "ok" }, "message": "ok" }` ，无需鉴权。

### 4.3.1 认证模块 `/api/v1/auth`

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/auth/register` | 注册 |
| POST | `/api/v1/auth/login` | 登录，返回 JWT |
| POST | `/api/v1/auth/refresh` | 刷新 token |
| GET | `/api/v1/auth/me` | 获取当前用户信息 |
| PUT | `/api/v1/auth/me` | 更新用户信息（avatarUrl，不含 username） |
| PUT | `/api/v1/auth/password` | 修改当前用户密码 |

### 4.3.1a 密码管理

| 方法 | 路径 | 说明 |
|------|------|------|
| PUT | `/api/v1/auth/password` | 校验当前密码并更新为新密码 |

请求示例：

```json
{
  "currentPassword": "old-password",
  "newPassword": "NewPassword#2026",
  "logoutOtherDevices": true
}
```

说明：

- `username` 为唯一账号标识，账号创建后冻结；成员本人和管理员都不能通过任何更新接口修改。
- `logoutOtherDevices=true` 时，修改密码成功后吊销当前用户其他设备登录态，仅保留当前请求对应设备。
- 失败时返回字段级错误或统一错误消息，不回显密码规则之外的敏感细节。

### 4.3.1b API 密钥管理 `/api/v1/api-keys`

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/api-keys` | 创建 API Key |
| GET | `/api/v1/api-keys` | API Key 列表 |
| DELETE | `/api/v1/api-keys/:id` | 吊销 API Key |

### 4.3.2 项目管理 `/api/v1/projects`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/projects` | 用户的项目列表（分页） |
| POST | `/api/v1/projects` | 创建项目 |
| GET | `/api/v1/projects/:id` | 项目详情 |
| PUT | `/api/v1/projects/:id` | 更新项目（名称/描述/Git配置等） |
| DELETE | `/api/v1/projects/:id` | 删除项目（软删除） |
| GET | `/api/v1/projects/:id/stats` | 项目统计（Issue数/完成率等） |
| GET | `/api/v1/projects/:id/history` | **项目变更历史** |

**创建/更新项目请求体**：

```json
{
  "name": "MySaaS",
  "slug": "my-saas",
  "description": "全栈 SaaS 应用，包含用户认证、支付、数据分析",
  "gitProtocol": "http",
  "gitProvider": "github",
  "gitRepoUrl": "https://github.com/user/my-saas",
  "gitRepoId": "user/my-saas",
  "defaultBranch": "main",
  "gitCredentialId": "cred_xxx",
  "settings": {
    "autoExecute": false,
    "autoCreatePR": true,
    "defaultAITool": "claude-code",
    "branchNaming": "flowcode/{{issueId}}-{{slug}}"
  }
}
```

> **gitCredentialId**：从项目已配置的凭证列表中选取（`GET /api/v1/projects/:pid/git/credentials`）。创建时可先不填，后续通过更新接口绑定。

> **创建流程**：前端先调用 `POST /api/v1/git/validate-url` 校验 Git 地址可达性 → 获得 `defaultBranch` / `repoId` → 填充完整请求体 → 调用创建项目接口。校验不通过则阻断创建。

### 4.3.3 Issue 管理 `/api/v1/issues`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/projects/:pid/issues` | Issue 列表（支持筛选/排序） |
| POST | `/api/v1/projects/:pid/issues` | 创建 Issue |
| GET | `/api/v1/issues/:id` | Issue 详情（含分析结果、执行计划、执行日志摘要） |
| PUT | `/api/v1/issues/:id` | 更新 Issue |
| DELETE | `/api/v1/issues/:id` | 删除 Issue |
| POST | `/api/v1/issues/:id/plan` | **生成/重新生成执行计划（须先完成分析）** |
| POST | `/api/v1/issues/:id/execute` | **触发 AI 执行（若无计划则先生成计划）** |
| POST | `/api/v1/issues/:id/continue` | **基于原计划继续 vibe coding** |
| POST | `/api/v1/issues/:id/cancel` | 取消执行 |
| POST | `/api/v1/issues/:id/retry` | **重试执行**（从 failed/cancelled/done 回到 draft） |
| POST | `/api/v1/issues/:id/skip` | **跳过执行**（opencode 判定无需代码变更时，直接到 done） |
| POST | `/api/v1/issues/:id/pause` | **暂停执行**（running → paused） |
| POST | `/api/v1/issues/:id/resume` | **恢复执行**（paused → running） |
| POST | `/api/v1/issues/:id/deploy` | **标记已部署**（pr_created → deployed） |
| POST | `/api/v1/issues/:id/close` | **关闭 Issue**（done/failed/cancelled/deployed/pr_created 均可关闭） |
| POST | `/api/v1/issues/:id/reopen` | **重新打开**（支持从 closed/cancelled/done/deployed/pr_created 重开，回到 draft） |
| GET | `/api/v1/issues/:id/logs` | 获取执行日志列表 |
| PUT | `/api/v1/issues/:id/priority` | 调整优先级 |
| PUT | `/api/v1/issues/:id/category` | 调整分类 |
| GET | `/api/v1/issues/:id/comments` | **Issue 评论列表** |
| POST | `/api/v1/issues/:id/comments` | **添加评论** |
| PUT | `/api/v1/comments/:id` | **编辑评论** |
| DELETE | `/api/v1/comments/:id` | **删除评论（软删除）** |
| GET | `/api/v1/issues/:id/questions` | **获取 Issue 的 AI 问询列表** |
| PUT | `/api/v1/issues/:id/questions/:qid/answer` | **回答 AI 问询** |
| PUT | `/api/v1/issues/:id/questions/:qid/skip` | **跳过 AI 问询** |

**AI 问询列表** `GET /api/v1/issues/:id/questions`：

```json
// 响应
{
  "code": 0,
  "data": [
    {
      "id": "q_abc123",
      "issueId": "iss_xyz",
      "question": "检测到项目同时存在 src/components/ 和 components/ 目录，应将新组件放在哪个目录下？",
      "context": "- 当前文件: App.tsx\n- src/components/ 包含 12 个组件\n- components/ 包含 3 个组件（较旧）",
      "options": ["src/components/ (推荐)", "components/", "统一到 src/components/ 并迁移旧组件"],
      "status": "pending",
      "askedAt": "2026-05-09T14:30:00Z"
    }
  ],
  "message": "ok"
}
```

**回答问询** `PUT /api/v1/issues/:id/questions/:qid/answer`：

```json
// 请求
{
  "answer": "统一到 src/components/ 并迁移旧组件",
  "selectedOption": 2
}
```

```json
// 响应
{
  "code": 0,
  "data": {
    "id": "q_abc123",
    "status": "answered",
    "answer": "统一到 src/components/ 并迁移旧组件",
    "answeredAt": "2026-05-09T14:35:00Z"
  },
  "message": "ok"
}
```

**跳过问询** `PUT /api/v1/issues/:id/questions/:qid/skip`：

```json
// 请求 (无请求体或可选原因)
{
  "reason": "让 AI 自行决策"
}
```

```json
// 响应
{
  "code": 0,
  "data": {
    "id": "q_abc123",
    "status": "skipped",
    "answeredAt": "2026-05-09T14:35:00Z"
  },
  "message": "ok"
}
```

**Issue 列表查询参数**：

**更新 Issue** `PUT /api/v1/issues/:id`：

> Issue 创建后可随时编辑补充内容。`description` 支持全文替换或增量追加。

```json
// 请求（所有字段可选，仅传需更新的字段）
{
  "title": "修复登录页面样式错乱并优化移动端适配",
  "description": "补充：发现 iOS Safari 下 input 框圆角异常，需要额外处理 -webkit-appearance",
  "appendDescription": true,
  "priority": "p1",
  "tags": ["frontend", "bug", "ios"],
  "assigneeId": "user_xxx",
  "aiTool": "claude-code",
  "estimatedHours": 2.5
}
```

| 可编辑字段 | 类型 | 说明 |
|-----------|------|------|
| title | string | 标题 |
| description | string | 描述（Markdown）。`appendDescription=true` 时追加而非替换 |
| appendDescription | bool | 设为 true 时，description 内容追加到现有描述末尾 |
| priority | string | p0 / p1 / p2 / p3 |
| tags | []string | 标签数组 |
| assigneeId | UUID | 指派执行者 |
| aiTool | string | 更换 AI 工具（仅在 AI 执行前可修改） |
| estimatedHours | float | 预估工时 |

> **不可编辑字段**（系统管理）：`status`（走工作流流转）、`category`（走独立 API）、`gitBranch`/`gitPRUrl`/`gitCommitSha`（AI 执行后自动写入）、`sourcePlatform`/`sourceIssueId`（导入后不可变）。

> **系统生成字段**：`isBug`、`needsCodeChange`、`analysisReason`、`analysisCompleted`、`planSummary`、`planDetail`、`planCompleted`、`planGeneratedAt` 由分析/计划流程自动维护，不接受普通更新接口直接写入。

#### 评论 `comments`

多用户可在 Issue 下协作讨论，支持 Markdown 和嵌套回复（楼中楼）。Issue 关闭后仍可评论。

**评论列表** `GET /api/v1/issues/:id/comments`：

```json
// 响应
{
  "code": 0,
  "data": [
    {
      "id": "cmt_1",
      "body": "这个按钮在 Safari 下确实有问题，我复现了。建议加 `-webkit-appearance: none`",
      "author": { "id": "user_1", "nickname": "小明", "avatarUrl": "..." },
      "parentId": null,
      "editedAt": null,
      "createdAt": "2026-05-06T10:30:00Z",
      "replies": [
        {
          "id": "cmt_2",
          "body": "同意，我来修。另外 iOS 的 input 圆角也需要处理",
          "author": { "id": "user_2", "nickname": "小红", "avatarUrl": "..." },
          "parentId": "cmt_1",
          "editedAt": null,
          "createdAt": "2026-05-06T10:35:00Z",
          "replies": []
        }
      ]
    }
  ],
  "message": "ok"
}
```

**添加评论** `POST /api/v1/issues/:id/comments`：

```json
// 请求
{
  "body": "建议优先处理登录页的样式问题，影响面最大。",
  "parentId": null
}
```

```json
// 回复某条评论
{
  "body": "@小明 好的，我先修登录页。",
  "parentId": "cmt_1"
}
```

```json
// 响应
{
  "code": 0,
  "data": {
    "id": "cmt_3",
    "body": "建议优先处理登录页的样式问题，影响面最大。",
    "author": { "id": "user_1", "nickname": "小明" },
    "parentId": null,
    "createdAt": "2026-05-06T11:00:00Z"
  },
  "message": "ok"
}
```

**编辑评论** `PUT /api/v1/comments/:id`：

> 仅作者可编辑自己的评论。编辑后 `editedAt` 自动更新。

```json
// 请求
{
  "body": "建议优先处理登录页的样式问题，影响面最大。（已确认 Safari 14+ 均受影响）"
}
```

**删除评论** `DELETE /api/v1/comments/:id`：

> 仅作者可删除。软删除，有子回复时保留占位（"此评论已删除"）。

```json
// 响应
{
  "code": 0,
  "message": "ok"
}
```

### 4.3.5 执行管理 `/api/v1/executions`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/executions/:logId` | 获取执行日志详情 |
| GET | `/api/v1/executions/:logId/stdout` | 获取完整标准输出 |
| WS | `/ws/executions/:issueId` | **WebSocket 实时日志流** |

### 4.3.6 Git 管理 `/api/v1/git`

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/git/validate-url` | **校验 Git 地址可达性**（创建项目前置） |
| POST | `/api/v1/projects/:pid/git/connect` | 配置 Git 凭证（PAT 或 SSH 密钥） |
| GET | `/api/v1/projects/:pid/git/credentials` | **凭证列表** |
| DELETE | `/api/v1/projects/:pid/git/credentials/:id` | **删除凭证** |
| GET | `/api/v1/projects/:pid/git/status` | Git 连接状态 |
| GET | `/api/v1/projects/:pid/git/branches` | 仓库分支列表 |
| POST | `/api/v1/projects/:pid/git/pr` | 手动创建 PR |
| GET | `/api/v1/issues/:id/pr-status` | 查看 Issue 关联 PR 状态 |
| POST | `/api/v1/projects/:pid/git/import-issues` | **从 Git 平台导入 Issues** |
| GET | `/api/v1/projects/:pid/git/import-status` | **查询导入结果** |

**校验 Git 地址** `POST /api/v1/git/validate-url`：

```json
// 请求
{
  "url": "https://github.com/user/my-saas",
  "protocol": "http",
  "credentialId": "cred_xxx"
}
```

```json
// 请求 (SSH)
{
  "url": "git@github.com:user/my-saas.git",
  "protocol": "ssh",
  "credentialId": "cred_yyy"
}
```

```json
// 响应
{
  "code": 0,
  "data": {
    "reachable": true,
    "defaultBranch": "main",
    "branches": ["main", "develop", "feat/login"],
    "repoId": "user/my-saas"
  },
  "message": "ok"
}
```

错误响应：

```json
{
  "code": 40001,
  "data": {
    "reachable": false,
    "errorCode": "GIT_UNREACHABLE",
    "errorDetail": "无法访问仓库: remote: Repository not found"
  },
  "message": "Git 地址不可达"
}
```

**配置 Git 凭证** `POST /api/v1/projects/:pid/git/connect`：

> 一个项目可以配置多个凭证，每次调用创建一个新凭证。用 `name` 区分。

```json
// HTTP 方式
{
  "name": "GitHub 主仓库",
  "authMethod": "http",
  "accessToken": "ghp_xxxxxxxxxxxxxxxxxxxx",
  "tokenType": "pat",
  "provider": "github"
}

// SSH 方式
{
  "name": "公司 GitLab",
  "authMethod": "ssh",
  "sshPrivateKey": "-----BEGIN OPENSSH PRIVATE KEY-----\nb3BlbnNzaC1rZXktdjEAAAAA...\n-----END OPENSSH PRIVATE KEY-----",
  "sshPassphrase": "",
  "provider": "github"
}
```

```json
// 响应
{
  "code": 0,
  "data": {
    "id": "cred_xxx",
    "name": "GitHub 主仓库",
    "fingerprint": "SHA256:xxxx"  // SSH 模式返回指纹
  },
  "message": "ok"
}
```

> SSH 私钥传入后，服务端验证格式与指纹，加密后写入 `GitCredential.sshPrivateKey`。指纹通过响应返回供确认。

**凭证列表** `GET /api/v1/projects/:pid/git/credentials`：

```json
// 响应
{
  "code": 0,
  "data": [
    {
      "id": "cred_1",
      "name": "GitHub 主仓库",
      "provider": "github",
      "authMethod": "http",
      "expiresAt": "2027-01-01T00:00:00Z",
      "lastUsedAt": "2026-05-06T10:30:00Z",
      "createdAt": "2026-01-01T00:00:00Z"
    },
    {
      "id": "cred_2",
      "name": "GitHub SSH",
      "provider": "github",
      "authMethod": "ssh",
      "fingerprint": "SHA256:xxxx",
      "lastUsedAt": null,
      "createdAt": "2026-04-01T00:00:00Z"
    }
  ],
  "message": "ok"
}
```

> 列表不返回 `accessToken` 和 `sshPrivateKey` 明文。

**删除凭证** `DELETE /api/v1/projects/:pid/git/credentials/:id`：

```json
// 响应
{
  "code": 0,
  "message": "ok"
}
```

**导入 Issues** `POST /api/v1/projects/:pid/git/import-issues`：

> 需先配置 Git 凭证并在项目中绑定 (`gitCredentialId`)。导入为幂等操作，已存在的 Issue 自动跳过。

```json
// 请求
{
  "state": "open",
  "since": "2026-01-01T00:00:00Z"
}
```

| 参数 | 必填 | 说明 |
|------|:--:|------|
| state | 否 | `open` / `closed` / `all`，默认 `open` |
| since | 否 | ISO 8601 时间，仅导入此时间之后的 Issue（增量导入） |

```json
// 响应（异步任务，立即返回任务ID）
{
  "code": 0,
  "data": {
    "taskId": "task_import_xxx",
    "status": "queued"
  },
  "message": "导入任务已创建"
}
```

**查询导入结果** `GET /api/v1/projects/:pid/git/import-status?taskId=task_import_xxx`：

```json
// 响应
{
  "code": 0,
  "data": {
    "taskId": "task_import_xxx",
    "status": "completed",
    "imported": 12,
    "skipped": 3,
    "total": 15,
    "details": [
      { "number": 1, "title": "Add login page",   "action": "imported", "flowcodeId": "iss_abc" },
      { "number": 3, "title": "Update README",     "action": "imported", "flowcodeId": "iss_def" },
      { "number": 2, "title": "Fix navbar z-index","action": "skipped",  "reason": "already_exists" }
    ]
  },
  "message": "ok"
}
```

### 4.3.7 AI 工具配置 `/api/v1/ai-tools`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/projects/:pid/ai-tools` | AI 工具配置列表 |
| POST | `/api/v1/projects/:pid/ai-tools` | 添加 AI 工具配置 |
| PUT | `/api/v1/ai-tools/:id` | 更新配置 |
| DELETE | `/api/v1/ai-tools/:id` | 删除配置 |
| POST | `/api/v1/ai-tools/:id/test` | **测试连接** |

### 4.3.8 标签管理 `/api/v1/labels`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/projects/:pid/labels` | 标签列表 |
| POST | `/api/v1/projects/:pid/labels` | 创建标签 |
| PUT | `/api/v1/labels/:id` | 更新标签 |
| DELETE | `/api/v1/labels/:id` | 删除标签 |

### 4.3.9 Skills 管理 `/api/v1/skills`

**Skills CRUD**：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/skills` | Skills 列表 (支持搜索/分类筛选) |
| GET | `/api/v1/skills/:slug` | Skill 详情 (含 Markdown 内容) |
| POST | `/api/v1/skills` | 创建自定义 Skill |
| PUT | `/api/v1/skills/:slug` | 更新 Skill |
| DELETE | `/api/v1/skills/:slug` | 删除 Skill |
| POST | `/api/v1/skills/:slug/publish` | 发布 (status=published) |
| POST | `/api/v1/skills/:slug/deprecate` | 标记为废弃 |
| POST | `/api/v1/projects/:pid/skills/sync` | 将 published Skills 写入项目 Git 仓库 `.flowcode/skills/` |

### 4.3.9a 项目配置 `.flowcode/config.yaml`

AI 工具读取 Skill 文件前，先加载项目根目录的配置文件，获取 API 地址和认证信息：

```yaml
# .flowcode/config.yaml — 项目级 flowcode 配置
apiUrl: "https://flowcode.mycompany.com"   # flowcode 部署地址
apiKey: "fc_sk_proj_abc123xyz"             # 项目 API Key (项目范围，可读写，可轮换)
projectId: "proj-abc123"                   # 当前项目 ID
```

**生成时机**：创建项目时自动生成并 commit 到仓库（`apiKey` 为项目级可读写密钥，可随时轮换或吊销）。

### 4.3.9b Skill 文件格式

Skill 存于 DB 用于管理，**导出到项目目录** 供外部 AI 工具直接读取。

AI 工具使用方式：
1. 读取 `.flowcode/config.yaml` 获取 `apiUrl` + `apiKey` + `projectId`
2. 读取 `.flowcode/skills/*.md`，替换其中 `{API_URL}` `{API_KEY}` `{PROJECT_ID}` 变量
3. 按 Skill 指令发起 HTTP 调用

```
.flowcode/
├── config.yaml              ← ① 先读这个：apiUrl + apiKey + projectId
└── skills/
    ├── issue-create.md      ← ② 再读这些，替换变量
    ├── issue-search.md
    ├── issue-status.md
    ├── workflow-next.md
    └── git-pr-create.md
```

**Skill 内可用变量**：

| 变量 | 来源 | 示例 |
|------|------|------|
| `{API_URL}` | config.yaml apiUrl | `https://flowcode.mycompany.com` |
| `{API_KEY}` | config.yaml apiKey | `fc_sk_proj_abc123xyz` |
| `{PROJECT_ID}` | config.yaml projectId | `proj-abc123` |
| `{ORG_ID}` | 已废弃 | — |

**单个 Skill 文件示例** (`issue-create.md`)：

```markdown
# Skill: 创建需求 (Issue)

## 用途
当需要创建新的需求/Issue 时使用此 Skill。

## 何时触发
- 用户说"创建一个需求"、"新建 Issue"
- 用户描述了新功能但未关联已有 Issue
- 从需求文档中提取出新的开发任务

## API 调用

POST {API_URL}/api/v1/projects/{PROJECT_ID}/issues
X-API-Key: {API_KEY}
Content-Type: application/json

{
  "title": "{用户描述的需求标题}",
  "description": "{用户描述的需求详情}",
  "priority": "p2"
}

## 响应
成功返回 201，响应体包含 issue ID，后续操作使用此 ID。

## 备注
- title 必填，不超过 200 字符
- description 支持 Markdown
- priority 可选 p0/p1/p2/p3，默认 p2
```

**Skill 内容模板**（创建时自动填充骨架）：

| 章节 | 必填 | 说明 |
|------|------|------|
| `# Skill: {name}` | ✅ | 标题 |
| `## 用途` | ✅ | 一句话说明 |
| `## 何时触发` | ✅ | AI 判断是否调用此 Skill 的条件 |
| `## API 调用` | ✅ | HTTP 方法 + URL + 请求体示例 + 变量占位 |
| `## 响应` | ✅ | 成功/失败响应说明 |
| `## 备注` | | 注意事项 |

### 4.3.9c 外部 AI 工具集成流程

```
┌─ 项目仓库 ─────────────────────────────────────────────┐
│                                                          │
│  .flowcode/                                              │
│  ├── config.yaml              ← API 地址 + 密钥          │
│  └── skills/                                             │
│      ├── issue-create.md      ← skill 指令文件           │
│      ├── issue-search.md                                │
│      ├── test-result.md                                 │
│      └── ...                                            │
│                                                          │
└──────────────────────┬───────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   opencode         codex       claude-code
        │              │              │
        │  ① 读取 .flowcode/config.yaml
        │  ② 读取 .flowcode/skills/*.md，替换 {API_URL} 等变量
        │  ③ 按 Skill 指令调用 flowcode API
        │
        ▼
  flowcode 执行操作 → 返回结果 → AI 工具拿到结果继续编码
```

**关键点**：
- **域名问题**：Skill 文件用 `{API_URL}` 占位，AI 工具从 `config.yaml` 读取实际地址
- Skill 文件是纯 Markdown，**零依赖**，任何 AI 工具都能读
- 变量 `{API_KEY}`, `{PROJECT_ID}` 同理从 `config.yaml` 获取
- flowcode 通过 `POST /api/v1/projects/:pid/skills/sync` 将 `config.yaml` + published Skills 写入项目 Git 仓库

### 4.3.9d sync 实现

```go
// POST /api/v1/projects/:pid/skills/sync 时执行
func (s *SkillService) SyncToRepo(ctx context.Context, projectID string) error {
    project := s.getProject(projectID)

    // 1. 写入 config.yaml
    configYaml := fmt.Sprintf(`apiUrl: "%s"
apiKey: "%s"
projectId: "%s"
`, s.cfg.BaseURL, project.APIKey, project.ID)
    s.git.WriteFile(projectID, ".flowcode/config.yaml", []byte(configYaml), "sync config")

    // 2. 写入所有 published Skills
    skills := s.getPublishedSkills()
    for _, sk := range skills {
        filename := fmt.Sprintf(".flowcode/skills/%s.md", sk.Slug)
        s.git.WriteFile(projectID, filename, []byte(sk.Content), "sync skills")
    }

    return s.git.CommitAndPush(projectID, "chore: sync flowcode config and skills")
}
```

### 4.3.10 PR 列表 `/api/v1/prs`

整合该项目所有 Issue 关联的 PR，通过 `GitProvider` 实时获取最新状态：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/projects/:pid/prs` | **项目 PR 列表**（支持状态筛选） |

**查询参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| status | string | 状态过滤: open / merged / closed |
| issueId | UUID | 按关联 Issue 过滤 |
| sort | string | 排序: createdAt / updatedAt (默认 createdAt) |
| order | string | asc / desc (默认 desc) |

**响应示例**：

```json
{
  "code": 0,
  "data": [
    {
      "prId": "pr_001",
      "prNumber": 42,
      "prTitle": "feat: 用户登录页 (iss_abc123)",
      "prUrl": "https://github.com/user/repo/pull/42",
      "sourceBranch": "flowcode/iss_abc123",
      "targetBranch": "main",
      "status": "open",
      "issueId": "iss_abc123",
      "issueTitle": "用户登录页",
      "createdAt": "2026-05-06T10:30:00Z",
      "updatedAt": "2026-05-06T11:00:00Z"
    }
  ]
}
```

> status 不依赖 Webhook 延迟：调用 `GitProvider` 实时拉取最新状态。

## 4.4 OpenAPI 文档生成

API 设计完成后，通过代码注解自动生成 OpenAPI 3.0 (Swagger) 文档：

### 工具选型：swaggo/swag

```bash
# 安装
go install github.com/swaggo/swag/cmd/swag@latest

# 从 Gin Handler 注解生成 docs/
swag init -g cmd/server/main.go -o docs/swagger --parseDependency
```

### 注解示例

```go
// @Summary      创建 Issue
// @Description  在指定项目中创建新的 Issue
// @Tags         Issues
// @Accept       json
// @Produce      json
// @Param        pid   path      string              true  "项目 ID"
// @Param        body  body      CreateIssueRequest  true  "Issue 内容"
// @Success      201   {object}  Response{data=Issue}
// @Failure      400   {object}  Response
// @Failure      403   {object}  Response
// @Security     BearerAuth
// @Router       /api/v1/projects/{pid}/issues [post]
func (h *IssueHandler) Create(c *gin.Context) { ... }
```

### 访问方式

```
开发环境: http://localhost:3000/swagger/index.html
生产环境: https://flowcode.example.com/swagger/index.html
```

> Swagger UI 仅在 `server.mode=debug` 时启用，生产环境关闭。
