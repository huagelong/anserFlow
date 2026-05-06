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

### 4.3.1 认证模块 `/api/v1/auth`

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/auth/register` | 注册 |
| POST | `/api/v1/auth/login` | 登录，返回 JWT |
| POST | `/api/v1/auth/refresh` | 刷新 token |
| GET | `/api/v1/auth/me` | 获取当前用户信息 |

### 4.3.1b 组织管理 `/api/v1/orgs` (SaaS)

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/orgs` | 创建组织 |
| GET | `/api/v1/orgs` | 用户所属组织列表 |
| GET | `/api/v1/orgs/:slug` | 组织详情 |
| PUT | `/api/v1/orgs/:slug` | 更新组织 |
| POST | `/api/v1/orgs/:slug/members` | 邀请成员 |
| DELETE | `/api/v1/orgs/:slug/members/:uid` | 移除成员 |
| PUT | `/api/v1/orgs/:slug/members/:uid/role` | 修改成员角色 |

### 4.3.1c API 密钥管理 `/api/v1/api-keys`

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/orgs/:slug/api-keys` | 创建 API Key |
| GET | `/api/v1/orgs/:slug/api-keys` | API Key 列表 |
| DELETE | `/api/v1/api-keys/:id` | 吊销 API Key |

### 4.3.2 项目管理 `/api/v1/projects`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/orgs/:slug/projects` | 组织下的项目列表（分页） |
| POST | `/api/v1/orgs/:slug/projects` | 创建项目 |
| GET | `/api/v1/projects/:id` | 项目详情 |
| PUT | `/api/v1/projects/:id` | 更新项目（名称/描述/Git配置等） |
| DELETE | `/api/v1/projects/:id` | 删除项目（软删除） |
| GET | `/api/v1/projects/:id/stats` | 项目统计（Issue数/完成率/测试覆盖率等） |
| GET | `/api/v1/projects/:id/history` | **项目变更历史** |

**创建/更新项目请求体**：

```json
{
  "name": "MySaaS",
  "slug": "my-saas",
  "description": "全栈 SaaS 应用，包含用户认证、支付、数据分析",
  "gitProtocol": "https",
  "gitProvider": "github",
  "gitRepoUrl": "https://github.com/user/my-saas",
  "gitRepoId": "user/my-saas",
  "defaultBranch": "main",
  "settings": {
    "autoCreatePR": true,
    "defaultAITool": "claude-code",
    "branchNaming": "flowcode/{{issueId}}-{{slug}}",
    "requireApproval": true,
    "testRequireApproval": true,
    "testFramework": "vitest",
    "testCoverageTarget": 80
  }
}
```

> **创建流程**：前端先调用 `POST /api/v1/git/validate-url` 校验 Git 地址可达性 → 获得 `defaultBranch` / `repoId` → 填充完整请求体 → 调用创建项目接口。校验不通过则阻断创建。

### 4.3.3 需求管理 `/api/v1/requirements`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/projects/:pid/requirements` | 需求列表 |
| POST | `/api/projects/:pid/requirements` | 创建需求 |
| GET | `/api/requirements/:id` | 需求详情 |
| PUT | `/api/requirements/:id` | 更新需求 |
| DELETE | `/api/requirements/:id` | 删除需求 |
| POST | `/api/requirements/:id/convert-to-issues` | **拆解为 Issue(s)** |
| PUT | `/api/requirements/:id/status` | 变更需求状态 |

**拆解为 Issue 请求体**：

```json
{
  "issues": [
    {
      "title": "实现用户登录接口 POST /api/auth/login",
      "description": "使用 JWT 方案，支持邮箱+密码登录...",
      "priority": "p0",
      "category": "feature",
      "aiTool": "claude-code"
    },
    {
      "title": "编写登录接口单元测试",
      "description": "覆盖正常登录、密码错误、用户不存在等场景",
      "priority": "p1",
      "category": "feature",
      "aiTool": "claude-code"
    }
  ]
}
```

### 4.3.4 Issue 管理 `/api/v1/issues`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/projects/:pid/issues` | Issue 列表（支持筛选/排序） |
| POST | `/api/projects/:pid/issues` | 创建 Issue |
| GET | `/api/issues/:id` | Issue 详情（含执行日志摘要） |
| PUT | `/api/issues/:id` | 更新 Issue |
| DELETE | `/api/issues/:id` | 删除 Issue |
| POST | `/api/issues/:id/approve` | **管理员审核通过** |
| POST | `/api/issues/:id/reject` | **管理员驳回（附原因）** |
| POST | `/api/issues/:id/execute` | **触发 AI 执行** |
| POST | `/api/issues/:id/continue` | **继续 vibe coding** |
| POST | `/api/issues/:id/cancel` | 取消执行 |
| GET | `/api/issues/:id/logs` | 获取执行日志列表 |
| PUT | `/api/issues/:id/priority` | 调整优先级 |
| PUT | `/api/issues/:id/category` | 调整分类 |

**Issue 列表查询参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| status | string | 状态过滤，逗号分隔 |
| priority | string | 优先级过滤 |
| category | string | 分类过滤 |
| search | string | 标题/描述全文搜索 |
| requirementId | UUID | 按需求筛选 |
| sort | string | 排序字段 (createdAt/priority/updatedAt) |
| order | string | asc / desc |
| cursor | string | 分页游标 |
| limit | int | 每页数量 (默认20) |

### 4.3.5 执行管理 `/api/v1/executions`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/executions/:logId` | 获取执行日志详情 |
| GET | `/api/executions/:logId/stdout` | 获取完整标准输出 |
| WS | `/ws/executions/:issueId` | **WebSocket 实时日志流** |

### 4.3.6 Git 管理 `/api/v1/git`

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/git/validate-url` | **校验 Git 地址可达性**（创建项目前置） |
| POST | `/api/projects/:pid/git/connect` | 配置 Git 凭证（PAT 或 SSH 密钥） |
| GET | `/api/projects/:pid/git/status` | Git 连接状态 |
| GET | `/api/projects/:pid/git/branches` | 仓库分支列表 |
| POST | `/api/projects/:pid/git/pr` | 手动创建 PR |
| GET | `/api/issues/:id/pr-status` | 查看 Issue 关联 PR 状态 |

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

**配置 Git 凭证** `POST /api/projects/:pid/git/connect`：

```json
// HTTP 方式
{
  "authMethod": "http",
  "accessToken": "ghp_xxxxxxxxxxxxxxxxxxxx",
  "tokenType": "pat",
  "provider": "github"
}

// SSH 方式
{
  "authMethod": "ssh",
  "sshPrivateKey": "-----BEGIN OPENSSH PRIVATE KEY-----\nb3BlbnNzaC1rZXktdjEAAAAA...\n-----END OPENSSH PRIVATE KEY-----",
  "sshPassphrase": "",
  "provider": "github"
}
```

> SSH 私钥传入后，服务端验证格式与指纹，加密后写入 `GitCredential.sshPrivateKey`。指纹通过响应返回供确认。

### 4.3.7 监控管理 `/api/v1/monitor`

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/monitor/:projectKey/events` | **接收监控事件**（公开接口） |
| GET | `/api/projects/:pid/monitor/events` | 查看监控事件列表 |
| GET | `/api/projects/:pid/monitor/stats` | 监控统计（错误趋势等） |
| POST | `/api/monitor/events/:id/convert` | 事件转 Issue |

### 4.3.8 AI 工具配置 `/api/v1/ai-tools`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/projects/:pid/ai-tools` | AI 工具配置列表 |
| POST | `/api/projects/:pid/ai-tools` | 添加 AI 工具配置 |
| PUT | `/api/ai-tools/:id` | 更新配置 |
| DELETE | `/api/ai-tools/:id` | 删除配置 |
| POST | `/api/ai-tools/:id/test` | **测试连接** |

### 4.3.9 标签管理 `/api/v1/labels`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/projects/:pid/labels` | 标签列表 |
| POST | `/api/v1/projects/:pid/labels` | 创建标签 |
| PUT | `/api/v1/labels/:id` | 更新标签 |
| DELETE | `/api/v1/labels/:id` | 删除标签 |

### 4.3.10 Skills 管理 `/api/v1/skills`

**组织内部 Skills CRUD**（需 org 权限）：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/skills` | Skills 列表 (支持搜索/分类筛选) |
| GET | `/api/v1/skills/:slug` | Skill 详情 (含 Markdown 内容) |
| POST | `/api/v1/skills` | 创建自定义 Skill |
| PUT | `/api/v1/skills/:slug` | 更新 Skill |
| DELETE | `/api/v1/skills/:slug` | 删除 Skill |
| POST | `/api/v1/skills/:slug/publish` | 发布 (status=published) |
| POST | `/api/v1/skills/:slug/deprecate` | 标记为废弃 |
| POST | `/api/v1/projects/:pid/skills/sync` | 将 org 的 published Skills 写入项目 Git 仓库 `.flowcode/skills/` |
| GET | `/api/v1/orgs/:slug/skills` | 组织已安装的 Skills |

**组织内部 Skills CRUD**（需 org 权限）：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/skills` | Skills 列表 (支持搜索/分类筛选) |
| GET | `/api/v1/skills/:slug` | Skill 详情 (含 Markdown 内容) |
| POST | `/api/v1/skills` | 创建自定义 Skill |
| PUT | `/api/v1/skills/:slug` | 更新 Skill |
| DELETE | `/api/v1/skills/:slug` | 删除 Skill |
| POST | `/api/v1/skills/:slug/publish` | 发布 (status=published) |
| POST | `/api/v1/skills/:slug/deprecate` | 标记为废弃 |
| POST | `/api/v1/projects/:pid/skills/sync` | 将 org 的 published Skills 写入项目 Git 仓库 `.flowcode/skills/` |
| GET | `/api/v1/orgs/:slug/skills` | 组织已安装的 Skills |

### 4.3.10a Skill 文件格式

Skill 存于 DB 用于管理，**导出到项目目录** 供外部 AI 工具直接读取。文件格式：

```
.flowcode/skills/
├── issue-create.md          ← 创建需求
├── issue-search.md          ← 搜索需求
├── issue-status.md          ← 查看状态
├── requirement-update.md    ← 更新需求
├── test-result.md           ← 查询测试结果
├── workflow-next.md         ← 推进工作流
└── git-pr-create.md         ← 创建 PR
```

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

POST https://flowcode.example.com/api/v1/issues
Authorization: Bearer {API_KEY}
Content-Type: application/json

{
  "title": "{用户描述的需求标题}",
  "description": "{用户描述的需求详情}",
  "projectId": "{当前项目 ID}",
  "priority": "P2"
}

## 响应
成功返回 201，响应体包含 issue ID，后续操作使用此 ID。

## 备注
- title 必填，不超过 200 字符
- description 支持 Markdown
- priority 可选 P0/P1/P2/P3，默认 P2
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

### 4.3.10b 外部 AI 工具集成流程

```
┌─ 项目仓库 ─────────────────────────────────────────────┐
│                                                          │
│  .flowcode/skills/                                       │
│  ├── issue-create.md        ← flowcode sync 写入        │
│  ├── issue-search.md                                    │
│  ├── test-result.md                                     │
│  └── ...                                                │
│                                                          │
└──────────────────────┬───────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   opencode         codex       claude-code
        │              │              │
        │  读取 .flowcode/skills/*.md，理解调用方式
        │
        ▼
  按 Skill 指令调用 flowcode API
  (POST /api/v1/issues, GET /api/v1/issues/:id, ...)
        │
        ▼
  flowcode 执行操作 → 返回结果 → AI 工具拿到结果继续编码
```

**关键点**：
- Skill 文件是纯 Markdown，**零依赖**，任何 AI 工具都能读
- 变量占位 `{API_KEY}`, `{项目 ID}` 由 AI 工具从上下文自动填充
- flowcode 通过 `POST /api/v1/projects/:pid/skills/sync` 将 org 的 published Skills 写入项目 Git 仓库

### 4.3.10c sync 实现

```go
// POST /api/v1/projects/:pid/skills/sync 时执行
func (s *SkillService) SyncToRepo(ctx context.Context, projectID string) error {
    skills := s.getPublishedSkills(projectID)
    dir := ".flowcode/skills"
    for _, sk := range skills {
        filename := fmt.Sprintf("%s/%s.md", dir, sk.Slug)
        s.git.WriteFile(projectID, filename, []byte(sk.Content), "sync skills")
    }
    return s.git.CommitAndPush(projectID, "chore: sync flowcode skills")
}
```

### 4.3.11 测试用例管理 `/api/v1/test-cases`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/issues/:iid/test-cases` | Issue 关联的测试用例列表 |
| POST | `/api/v1/issues/:iid/test-cases` | 创建测试用例（用户定义测试描述） |
| GET | `/api/v1/test-cases/:id` | 测试用例详情 |
| PUT | `/api/v1/test-cases/:id` | 更新测试描述 |
| DELETE | `/api/v1/test-cases/:id` | 删除测试用例 |
| POST | `/api/v1/test-cases/:id/submit-review` | **提交审核** (draft → pending_review) |
| POST | `/api/v1/test-cases/:id/approve` | **管理员审核通过** → AI 自动生成测试代码 |
| POST | `/api/v1/test-cases/:id/reject` | **管理员驳回** (附驳回原因) |
| POST | `/api/v1/test-cases/:id/regenerate` | 重新生成测试代码（审核通过后） |
| POST | `/api/v1/test-cases/:id/run` | 运行测试用例 |
| GET | `/api/v1/test-cases/:id/history` | **测试用例变更历史** |

**创建测试用例请求体**：

```json
{
  "name": "用户登录接口测试",
  "description": "测试 POST /api/v1/auth/login 接口的各种场景：\n1. 正确的用户名和密码返回 JWT token\n2. 错误的密码返回 401\n3. 不存在的用户返回 404\n4. 缺少必填字段返回 400\n5. 并发登录限制",
  "language": "go",
  "framework": "testing",
  "testScript": {
    "install": "cd {workspace} && go mod download",
    "command": "cd {workspace} && go test ./pkg/auth/... -v -race -cover 2>&1",
    "timeout": 120
  }
}
```