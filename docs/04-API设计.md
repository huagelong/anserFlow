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
| GET | `/api/v1/projects/:pid/requirements` | 需求列表 |
| POST | `/api/v1/projects/:pid/requirements` | 创建需求 |
| GET | `/api/v1/requirements/:id` | 需求详情 |
| PUT | `/api/v1/requirements/:id` | 更新需求 |
| DELETE | `/api/v1/requirements/:id` | 删除需求 |
| POST | `/api/v1/requirements/:id/convert-to-issues` | **拆解为 Issue(s)** |
| PUT | `/api/v1/requirements/:id/status` | 变更需求状态 |

**拆解为 Issue 请求体**：

```json
{
  "issues": [
    {
      "title": "实现用户登录接口 POST /api/v1/auth/login",
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
| GET | `/api/v1/projects/:pid/issues` | Issue 列表（支持筛选/排序） |
| POST | `/api/v1/projects/:pid/issues` | 创建 Issue |
| GET | `/api/v1/issues/:id` | Issue 详情（含执行日志摘要） |
| PUT | `/api/v1/issues/:id` | 更新 Issue |
| DELETE | `/api/v1/issues/:id` | 删除 Issue |
| POST | `/api/v1/issues/:id/approve` | **管理员审核通过** |
| POST | `/api/v1/issues/:id/reject` | **管理员驳回（附原因）** |
| POST | `/api/v1/issues/:id/execute` | **触发 AI 执行** |
| POST | `/api/v1/issues/:id/continue` | **继续 vibe coding** |
| POST | `/api/v1/issues/:id/cancel` | 取消执行 |
| GET | `/api/v1/issues/:id/logs` | 获取执行日志列表 |
| PUT | `/api/v1/issues/:id/priority` | 调整优先级 |
| PUT | `/api/v1/issues/:id/category` | 调整分类 |

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
| GET | `/api/v1/executions/:logId` | 获取执行日志详情 |
| GET | `/api/v1/executions/:logId/stdout` | 获取完整标准输出 |
| WS | `/ws/executions/:issueId` | **WebSocket 实时日志流** |

### 4.3.6 Git 管理 `/api/v1/git`

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/git/validate-url` | **校验 Git 地址可达性**（创建项目前置） |
| POST | `/api/v1/projects/:pid/git/connect` | 配置 Git 凭证（PAT 或 SSH 密钥） |
| GET | `/api/v1/projects/:pid/git/status` | Git 连接状态 |
| GET | `/api/v1/projects/:pid/git/branches` | 仓库分支列表 |
| POST | `/api/v1/projects/:pid/git/pr` | 手动创建 PR |
| GET | `/api/v1/issues/:id/pr-status` | 查看 Issue 关联 PR 状态 |

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
| POST | `/api/v1/monitor/:projectKey/events` | **接收监控事件**（公开接口） |
| GET | `/api/v1/projects/:pid/monitor/events` | 查看监控事件列表 |
| GET | `/api/v1/projects/:pid/monitor/stats` | 监控统计（错误趋势等） |
| POST | `/api/v1/monitor/events/:id/convert` | 事件转 Issue |

### 4.3.8 AI 工具配置 `/api/v1/ai-tools`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/projects/:pid/ai-tools` | AI 工具配置列表 |
| POST | `/api/v1/projects/:pid/ai-tools` | 添加 AI 工具配置 |
| PUT | `/api/v1/ai-tools/:id` | 更新配置 |
| DELETE | `/api/v1/ai-tools/:id` | 删除配置 |
| POST | `/api/v1/ai-tools/:id/test` | **测试连接** |

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

### 4.3.10a 项目配置 `.flowcode/config.yaml`

AI 工具读取 Skill 文件前，先加载项目根目录的配置文件，获取 API 地址和认证信息：

```yaml
# .flowcode/config.yaml — 项目级 flowcode 配置
apiUrl: "https://flowcode.mycompany.com"   # flowcode 部署地址
apiKey: "fk_proj_abc123xyz"                # 项目 API Key (只读)
projectId: "proj-abc123"                   # 当前项目 ID
orgId: "org-xyz789"                        # 所属组织 ID
```

**生成时机**：创建项目时自动生成并 commit 到仓库（`apiKey` 为只读密钥，可随时轮换）。

### 4.3.10b Skill 文件格式

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
    ├── requirement-update.md
    ├── test-result.md
    ├── workflow-next.md
    └── git-pr-create.md
```

**Skill 内可用变量**：

| 变量 | 来源 | 示例 |
|------|------|------|
| `{API_URL}` | config.yaml apiUrl | `https://flowcode.mycompany.com` |
| `{API_KEY}` | config.yaml apiKey | `fk_proj_abc123xyz` |
| `{PROJECT_ID}` | config.yaml projectId | `proj-abc123` |
| `{ORG_ID}` | config.yaml orgId | `org-xyz789` |

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
Authorization: Bearer {API_KEY}
Content-Type: application/json

{
  "title": "{用户描述的需求标题}",
  "description": "{用户描述的需求详情}",
  "priority": "P2"
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

### 4.3.10c 外部 AI 工具集成流程

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
- 变量 `{API_KEY}`, `{PROJECT_ID}`, `{ORG_ID}` 同理从 `config.yaml` 获取
- flowcode 通过 `POST /api/v1/projects/:pid/skills/sync` 将 `config.yaml` + published Skills 写入项目 Git 仓库

### 4.3.10d sync 实现

```go
// POST /api/v1/projects/:pid/skills/sync 时执行
func (s *SkillService) SyncToRepo(ctx context.Context, projectID string) error {
    project := s.getProject(projectID)

    // 1. 写入 config.yaml
    configYaml := fmt.Sprintf(`apiUrl: "%s"
apiKey: "%s"
projectId: "%s"
orgId: "%s"
`, s.cfg.BaseURL, project.APIKey, project.ID, project.OrgID)
    s.git.WriteFile(projectID, ".flowcode/config.yaml", []byte(configYaml), "sync config")

    // 2. 写入所有 published Skills
    skills := s.getPublishedSkills(project.OrgID)
    for _, sk := range skills {
        filename := fmt.Sprintf(".flowcode/skills/%s.md", sk.Slug)
        s.git.WriteFile(projectID, filename, []byte(sk.Content), "sync skills")
    }

    return s.git.CommitAndPush(projectID, "chore: sync flowcode config and skills")
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

### 4.3.12 PR 列表 `/api/v1/prs`

整合该项目所有 Issue 关联的 PR，通过 `PlatformProvider` 实时获取最新状态：

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

> status 不依赖 Webhook 延迟：调用 `PlatformProvider.GetPR()` 实时拉取最新状态。

### 4.3.13 审计日志 `/api/v1/audit-logs`

**操作审计**（业务敏感操作）：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/projects/:pid/audit-logs` | 项目操作审计日志 |
| GET | `/api/v1/orgs/:slug/audit-logs` | 组织级操作审计日志 |
| GET | `/api/v1/users/me/audit-logs` | 当前用户个人操作历史 |
| GET | `/api/v1/issues/:id/audit-logs` | 某个 Issue 的全部操作记录 |
| GET | `/api/v1/test-cases/:id/audit-logs` | 某个测试用例的全部操作记录 |

**查询参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| action | string | 操作类型: issue.create / issue.execute / test_case.run ... |
| source | string | 来源: cli / web / api / webhook |
| targetType | string | 目标类型: issue / test_case / project / skill |
| userId | UUID | 按操作人过滤 (组织级可用) |
| since | string | 起始时间 (如 7d / 30d / 2026-05-01) |
| until | string | 结束时间 |
| cursor | string | 分页游标 |
| limit | int | 每页数量 (默认 50) |

**响应示例**：

```json
{
  "code": 0,
  "data": [
    {
      "id": "audit_001",
      "action": "issue.execute",
      "source": "cli",
      "targetType": "issue",
      "targetId": "iss_abc123",
      "detail": "触发 AI 执行: 用户登录页",
      "changes": {"status": {"old": "approved", "new": "queued"}},
      "userId": "user_001",
      "userName": "张三",
      "deviceId": "dev_macbook_01",
      "ip": "192.168.1.100",
      "createdAt": "2026-05-06T10:20:00Z"
    }
  ],
  "cursor": "eyJjcmVhdGVkQXQiOiAiMjAyNi0wNS0wNlQ...",
  "hasMore": true
}
```

### 4.3.13b 登录日志 `/api/v1/login-logs`

**登录审计**（所有登录尝试，含失败记录）：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/users/me/login-logs` | 当前用户登录历史 |
| GET | `/api/v1/orgs/:slug/login-logs` | 组织成员登录记录（管理员） |

**查询参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| status | string | success / failed |
| source | string | cli / web |
| userId | UUID | 按用户过滤 (组织级可用) |
| since | string | 起始时间 |
| cursor | string | 分页游标 |
| limit | int | 每页数量 (默认 20) |

**响应示例**：

```json
{
  "code": 0,
  "data": [
    {
      "id": "login_001",
      "userId": "user_001",
      "status": "success",
      "ip": "192.168.1.100",
      "userAgent": "Mozilla/5.0 ...",
      "deviceId": "dev_macbook_01",
      "deviceName": "MacBook Pro M3",
      "source": "web",
      "createdAt": "2026-05-06T09:00:00Z"
    },
    {
      "id": "login_002",
      "userId": "user_001",
      "status": "failed",
      "failReason": "invalid_password",
      "ip": "192.168.1.100",
      "userAgent": "Mozilla/5.0 ...",
      "source": "web",
      "createdAt": "2026-05-06T09:01:00Z"
    }
  ]
}
```

**安全告警查询**（组织管理员）：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/orgs/:slug/login-logs/anomalies` | 登录异常检测 |

返回近 24 小时内同一 IP 失败 ≥ 5 次 / 异地登录 / 新设备登录等异常事件：

```json
{
  "code": 0,
  "data": [
    {
      "type": "brute_force",
      "severity": "high",
      "ip": "203.0.113.42",
      "failedCount": 12,
      "userIds": ["user_001", "user_003"],
      "firstSeen": "2026-05-06T08:00:00Z",
      "lastSeen": "2026-05-06T08:15:00Z"
    },
    {
      "type": "new_device",
      "severity": "low",
      "userId": "user_005",
      "deviceName": "Unknown Windows PC",
      "ip": "198.51.100.10",
      "createdAt": "2026-05-06T07:30:00Z"
    }
  ]
}
```

### 4.3.14 设备管理 `/api/v1/devices`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/users/me/devices` | 当前用户已登录设备列表 |
| DELETE | `/api/v1/devices/:id` | **吊销设备**（远程登出） |
| DELETE | `/api/v1/users/me/devices` | 吊销当前用户所有其他设备 (仅保留当前设备) |

**响应示例**：

```json
{
  "code": 0,
  "data": [
    {
      "id": "dev_macbook_01",
      "deviceName": "MacBook Pro M3",
      "os": "darwin",
      "arch": "arm64",
      "current": true,
      "lastSeenAt": "2026-05-06T11:00:00Z",
      "createdAt": "2026-04-15T09:00:00Z"
    },
    {
      "id": "dev_ci_runner_02",
      "deviceName": "CI Runner #2",
      "os": "linux",
      "arch": "amd64",
      "current": false,
      "lastSeenAt": "2026-05-05T18:30:00Z",
      "createdAt": "2026-04-20T14:00:00Z"
    }
  ]
}
```

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
