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

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/skills` | Skills 列表 (支持搜索/分类筛选) |
| GET | `/api/v1/skills/:id` | Skill 详情 (含 markdown 内容) |
| POST | `/api/v1/skills` | 创建自定义 Skill |
| PUT | `/api/v1/skills/:id` | 更新 Skill |
| DELETE | `/api/v1/skills/:id` | 删除 Skill |
| POST | `/api/v1/skills/:id/install` | 安装到组织 (从市场) |
| GET | `/api/v1/orgs/:slug/skills` | 组织已安装的 Skills |

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
  "framework": "testing"
}
```

**审核通过后的自动流程**：

```
TestCase approved → AI 根据 description 生成 testCode → status=generated → 自动运行测试 → 记录结果
```

### 4.3.12 CLI 辅助接口 `/api/v1/cli`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/cli/version` | 最新 CLI 版本信息 |
| GET | `/api/v1/cli/download/:os/:arch` | CLI 下载 (自动重定向) |
| GET | `/api/v1/cli/status` | CLI 服务状态

### 4.3.13 历史记录 `/api/v1/history`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/projects/:id/history` | 项目变更历史时间线 |
| GET | `/api/v1/requirements/:id/history` | 需求变更历史 |
| GET | `/api/v1/issues/:id/history` | Issue 变更历史 |
| GET | `/api/v1/test-cases/:id/history` | 测试用例变更历史 |
| GET | `/api/v1/history/compare/:id` | **对比两个历史版本** (v1 vs v2) |
| GET | `/api/v1/history/:recordId` | 单条历史记录详情 |

## 4.4 WebSocket 协议

### 连接地址

```
ws://localhost:3000/ws/executions/:issueId?token=<JWT>
```

### 消息格式

**服务端 → 客户端**：

```json
{
  "type": "log",
  "data": {
    "timestamp": 1715000000000,
    "level": "info",
    "content": "Installing dependencies: express, zod, jsonwebtoken...",
    "step": "install"
  }
}
```

```json
{
  "type": "status",
  "data": {
    "status": "completed",
    "filesChanged": ["src/routes/auth.ts", "src/middleware/auth.ts"],
    "exitCode": 0,
    "duration": 45200
  }
}
```

```json
{
  "type": "progress",
  "data": {
    "percent": 75,
    "currentStep": "Writing tests",
    "stepsTotal": 4,
    "stepsDone": 3
  }
}
```

### 连接生命周期

```
[连接] → [认证JWT] → [订阅Issue] → [接收实时日志]
   │                                      │
   │          ┌───────────────────────────┘
   │          ▼
   │   [执行完成/失败] → [服务端发送 status 消息] → [关闭连接]
   │
   └── [客户端主动断开]
```

## 4.5 鉴权方案

- 使用 JWT (Access Token + Refresh Token)
- Access Token 有效期 2 小时，Refresh Token 有效期 30 天
- 除 `/api/v1/auth/*` 和 `/api/v1/monitor/:projectKey/events` 外，所有接口需携带 `Authorization: Bearer <token>`
- **API Key 鉴权**：可在 HTTP Header 中传递 `X-API-Key: fc_sk_xxx` 用于 CLI 和 CI/CD 场景
- 监控事件接口使用 `projectKey`（项目级 Key）鉴权

---

## 4.5b 历史记录与版本对比

### 查询参数

| 参数 | 类型 | 说明 |
|------|------|------|
| limit | int | 每页数量 (默认 50) |
| cursor | string | 分页游标 |
| action | string | 按操作类型过滤 (created/updated/status_changed) |
| fieldName | string | 按字段名过滤 (title/description/status/priority) |

### 响应格式

```json
{
  "code": 0,
  "data": [
    {
      "id": "h_rec_xxx",
      "targetType": "issue",
      "targetId": "abc123",
      "action": "status_changed",
      "fieldName": "status",
      "oldValue": {"status": "draft"},
      "newValue": {"status": "approved"},
      "changedBy": {
        "id": "u1",
        "username": "admin"
      },
      "createdAt": "2026-05-06T14:30:00Z"
    }
  ],
  "message": "ok",
  "timestamp": 1715000000000
}
```

### 版本对比

`GET /api/v1/history/compare/:recordId`

```json
{
  "code": 0,
  "data": {
    "recordId": "h_rec_yyy",
    "fieldName": "title",
    "diff": {
      "type": "unified",
      "lines": [
        {"type": "remove", "content": "修复登录页面"},
        {"type": "add",   "content": "修复登录页面样式错乱问题"}
      ]
    },
    "oldValue": "修复登录页面",
    "newValue": "修复登录页面样式错乱问题",
    "changedAt": "2026-05-06T14:30:00Z",
    "changedBy": "admin"
  }
}
```

## 4.6 API 安全机制

### 4.6.1 请求签名 (HMAC-SHA256)

CLI 和第三方集成在调用敏感接口时，除 API Key 外还需附带请求签名，防止中间人篡改：

```
请求头:
  X-API-Key: fc_sk_xxx
  X-Request-Timestamp: 1715000000
  X-Request-Nonce: random-uuid-v4
  X-Request-Signature: sha256=<HMAC(method+path+body+timestamp+nonce, api_secret)>
```

```go
// internal/middleware/request_sign.go

func RequestSignatureMiddleware(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        timestamp := c.GetHeader("X-Request-Timestamp")
        nonce := c.GetHeader("X-Request-Nonce")
        signature := c.GetHeader("X-Request-Signature")

        // 1. 时间戳防重放 (5 分钟窗口)
        ts, _ := strconv.ParseInt(timestamp, 10, 64)
        if time.Now().Unix()-ts > 300 || ts-time.Now().Unix() > 300 {
            c.AbortWithStatusJSON(401, gin.H{"code": 40101, "message": "Request expired"})
            return
        }

        // 2. Nonce 防重放 (Redis SETNX, 5 分钟 TTL)
        if !redis.SetNX(c.Request.Context(), "nonce:"+nonce, "1", 5*time.Minute).Val() {
            c.AbortWithStatusJSON(401, gin.H{"code": 40102, "message": "Nonce already used"})
            return
        }

        // 3. 签名验证
        bodyBytes, _ := io.ReadAll(c.Request.Body)
        c.Request.Body = io.NopCloser(bytes.NewBuffer(bodyBytes))

        payload := fmt.Sprintf("%s%s%s%s%s",
            c.Request.Method,
            c.Request.URL.Path,
            string(bodyBytes),
            timestamp,
            nonce,
        )
        mac := hmac.New(sha256.New, []byte(secret))
        mac.Write([]byte(payload))
        expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))

        if !hmac.Equal([]byte(signature), []byte(expected)) {
            c.AbortWithStatusJSON(401, gin.H{"code": 40103, "message": "Invalid signature"})
            return
        }

        c.Next()
    }
}
```

### 4.6.2 频率限制 (Rate Limiting)

多层级限流策略，防止滥用和 DDoS：

| 层级 | 限流键 | 默认限制 | 适用对象 |
|------|--------|----------|---------|
| 全局限流 | IP | 1000 req/min | 所有请求 |
| API Key 限流 | API Key ID | 300 req/min | CLI/API 调用 |
| 用户限流 | User ID | 500 req/min | Web 用户 |
| 敏感接口限流 | 接口路径 | 10 req/min | login/register/execute |

```go
// internal/middleware/rate_limiter.go

func RateLimiter(store *redis.Client) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 优先级: API Key > User ID > IP
        key := c.ClientIP()
        limit := 1000

        if apiKey := c.GetHeader("X-API-Key"); apiKey != "" {
            key = "apikey:" + sha256Prefix(apiKey)
            limit = 300
        } else if userID := c.GetString("userID"); userID != "" {
            key = "user:" + userID
            limit = 500
        }

        // 敏感接口更严限制
        if isSensitivePath(c.Request.URL.Path) {
            limit = 10
        }

        count, _ := store.Incr(c.Request.Context(), "ratelimit:"+key+":"+windowKey()).Result()
        if count == 1 {
            store.Expire(c.Request.Context(), "ratelimit:"+key+":"+windowKey(), time.Minute)
        }
        if count > int64(limit) {
            c.AbortWithStatusJSON(429, gin.H{
                "code":      42900,
                "message":   "Too many requests, please retry later",
                "retryAfter": 60,
            })
            return
        }

        c.Header("X-RateLimit-Limit", strconv.Itoa(limit))
        c.Header("X-RateLimit-Remaining", strconv.Itoa(limit-int(count)))
        c.Next()
    }
}
```

### 4.6.3 API Key 权限范围 (Scopes)

API Key 创建时必须声明权限范围，最小权限原则：

```json
{
  "name": "CI/CD Pipeline",
  "scopes": ["issue:read", "issue:write", "execution:trigger"],
  "allowedIPs": ["10.0.1.0/24", "203.0.113.5"],
  "expiresIn": "90d"
}
```

| Scope | 允许操作 |
|-------|---------|
| `issue:read` | 查看 Issue 列表/详情 |
| `issue:write` | 创建/更新 Issue |
| `execution:trigger` | 触发 AI 执行 |
| `requirement:read` | 查看需求 |
| `requirement:write` | 创建/更新需求 |
| `skill:install` | 安装 Skill |
| `project:admin` | 项目管理（配置/成员） |
| `org:admin` | 组织管理（成员/计费） |

```go
// internal/middleware/api_key.go

func APIKeyAuth(db *gorm.DB) gin.HandlerFunc {
    return func(c *gin.Context) {
        key := c.GetHeader("X-API-Key")
        if key == "" {
            c.Next()
            return
        }

        hashed := sha256Hex(key)
        var token model.APIToken
        if err := db.Where("token = ?", hashed).First(&token).Error; err != nil {
            c.AbortWithStatusJSON(401, gin.H{"code": 40100, "message": "Invalid API Key"})
            return
        }

        // 检查过期
        if token.ExpiresAt != nil && token.ExpiresAt.Before(time.Now()) {
            c.AbortWithStatusJSON(401, gin.H{"code": 40104, "message": "API Key expired"})
            return
        }

        // 检查 IP 白名单
        if len(token.AllowedIPs) > 0 && !isIPAllowed(c.ClientIP(), token.AllowedIPs) {
            c.AbortWithStatusJSON(403, gin.H{"code": 40301, "message": "IP not allowed"})
            return
        }

        // 检查 scope 权限
        var scopes []string
        json.Unmarshal([]byte(token.Scopes), &scopes)
        if !hasScope(scopes, requiredScope(c.Request)) {
            c.AbortWithStatusJSON(403, gin.H{"code": 40302, "message": "Insufficient scope"})
            return
        }

        // 更新最后使用时间
        db.Model(&token).Update("last_used_at", time.Now())

        c.Set("orgID", token.OrgID)
        c.Set("userID", token.UserID)
        c.Set("authMethod", "apikey")
        c.Next()
    }
}
```

### 4.6.4 输入校验与防注入

所有外部输入必须经过严格校验，使用 Go validator 库：

```go
// internal/middleware/validator.go

type CreateIssueInput struct {
    Title       string `json:"title"       binding:"required,min=3,max=200,safetext"`
    Description string `json:"description" binding:"required,max=10000,safetext"`
    Priority    string `json:"priority"    binding:"required,oneof=p0 p1 p2 p3"`
    Category    string `json:"category"    binding:"required,oneof=feature bug refactor docs infra"`
    ProjectID   string `json:"project_id"  binding:"required,uuid"`
    AITool      string `json:"ai_tool"     binding:"required,alphanum,min=2,max=30"`
    SkillIDs    []string `json:"skill_ids" binding:"max=10,dive,uuid"`
}
```

关键防护措施：

| 措施 | 说明 |
|------|------|
| **类型约束** | 所有字段声明 Go 类型，JSON 反序列化自动类型检查 |
| **长度限制** | 标题 max=200，描述 max=10000，数组 max=10 |
| **枚举约束** | priority/category 仅允许预定义值 |
| **UUID 校验** | ID 类字段强制 UUID 格式 |
| **XSS 防护** | 输出时 HTML 转义 (前端 + 服务端双保险) |
| **SQL 注入防护** | Gorm 参数化查询，不拼接 SQL |
| **Prompt 注入防护** | Skills 内容注入前做 Markdown 解析器沙箱化处理 |

### 4.6.5 CORS 与安全头

```go
// internal/middleware/security_headers.go

func SecurityHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Referrer-Policy", "strict-origin-when-cross-origin")
        c.Header("Content-Security-Policy", "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'")
        c.Header("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        c.Next()
    }
}
```

CORS 白名单仅允许已注册的域名：

```go
func CORS(allowedOrigins []string) gin.HandlerFunc {
    return func(c *gin.Context) {
        origin := c.GetHeader("Origin")
        for _, allowed := range allowedOrigins {
            if origin == allowed {
                c.Header("Access-Control-Allow-Origin", origin)
                c.Header("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE")
                c.Header("Access-Control-Allow-Headers", "Authorization,Content-Type,X-API-Key,X-Request-Timestamp,X-Request-Nonce,X-Request-Signature")
                c.Header("Access-Control-Max-Age", "86400")
                break
            }
        }
        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }
        c.Next()
    }
}
```

### 4.6.6 审计日志

所有敏感操作记录审计日志，不可篡改：

```go
// internal/middleware/audit.go

type AuditLog struct {
    ID         string    `gorm:"primaryKey"`
    UserID     string
    OrgID      string
    Action     string    // "issue.execute" / "org.member.add" / "api_key.create"
    Resource   string    // "issue:abc123" / "org:my-org"
    IP         string
    UserAgent  string
    AuthMethod string    // "jwt" / "apikey" / "projectkey"
    Detail     string    // JSON 详情
    CreatedAt  time.Time
}

func AuditMiddleware(db *gorm.DB, sensitiveActions []string) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next() // 先执行业务逻辑

        action := c.GetString("auditAction")
        if !contains(sensitiveActions, action) {
            return
        }

        db.Create(&AuditLog{
            UserID:     c.GetString("userID"),
            OrgID:      c.GetString("orgID"),
            Action:     action,
            Resource:   c.GetString("auditResource"),
            IP:         c.ClientIP(),
            UserAgent:  c.GetHeader("User-Agent"),
            AuthMethod: c.GetString("authMethod"),
            Detail:     c.GetString("auditDetail"),
        })
    }
}
```

### 4.6.7 安全中间件执行顺序

```
请求 → CORS → SecurityHeaders → RateLimiter → [Auth选择]
                                                      │
                                    ┌─────────────────┼─────────────────┐
                                    ▼                 ▼                 ▼
                              JWT Auth          API Key Auth      ProjectKey Auth
                                │                   │                   │
                                └───────────────────┼───────────────────┘
                                                    ▼
                                            RequestSignature (可选)
                                                    │
                                                    ▼
                                              InputValidator
                                                    │
                                                    ▼
                                              业务 Handler
                                                    │
                                                    ▼
                                              AuditLogger
```
