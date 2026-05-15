# 项目概览与技术选型

# AnserFlow — 多智能体协作项目管理系统 架构分析文档

## 一、系统愿景

构建一个 **AI Agent + 自然人混合协作** 的项目管理平台。核心场景：

> 自然人创建项目 → 拉群（CEO/CTO/前端/后端等 Agent 角色入群）→ 发布需求 → Agent 根据角色设定自动讨论生成落地方案 → 自动拆解为 Issue 并关联项目 → Agent 认领执行 → 自然人审核验收。

---

## 当前阶段闭环约束

为避免规划项长期悬空，本文档对当前阶段范围做如下收口：

1. 当前交付只以 **L1-L4 路线图** 为验收范围；第十五章统一视为远期 backlog，不计入本轮完成标准。
2. 当前 Git 平台只验收 **GitHub**；`git_platform` 仅保留数据模型兼容位，不要求本轮实现 Gitea / GitLab / Gitee Provider。
3. 当前前端交付只闭环 **admin SPA 嵌入 Go** 与 **Tauri 桌面端独立打包**；不在本轮实现双 SPA 同进程路由分发。
4. 当前客户端只闭环 **桌面端**；Android / iOS、Crowdin / Lokalise、Pact 合约测试、`golang-migrate`、文档自动生成与 wiki 拆分均归入 Phase 2 或单独立项，不阻塞本轮验收。
5. 文中的目录树、接口、伪代码和工作流若未在仓库中落地，默认按 **目标架构说明** 理解，不视为“仓库现状已实现”。

---

## 二、技术栈总览

```
┌──────────────────────────────────────────┐
│ 前端       Next.js 14 SPA (static export)│
│           shadcn/ui + Tailwind CSS        │
│           TanStack Query (数据请求)       │
│           Zustand (客户端状态)            │
│           React Hook Form + Zod (表单)    │
│           next-intl (国际化)              │
│           Tauri 2.x (桌面+移动端)         │
├──────────────────────────────────────────┤
│ 后端框架   Gin                           │
│ ORM        GORM                          │
│ 数据库     MySQL 8.0+                    │
│ CLI        Cobra                         │
│ 静态嵌入   embed (Go 1.16+)              │
│ 配置       Viper                         │
│ 日志       Zap                           │
│ 校验       go-playground/validator       │
│ 权限       Casbin (RBAC)                 │
├──────────────────────────────────────────┤
│ 缓存/广播  Redis                         │
│ 实时通信   Gorilla WebSocket             │
│           + Redis Pub/Sub (分布式)        │
│ 任务队列   Asynq (基于 Redis)            │
├──────────────────────────────────────────┤
│ AI Agent   Eino (字节跳动 CloudWeGo)     │
│            + 自研业务封装层               │
│ 沙箱       Docker SDK for Go            │
├──────────────────────────────────────────┤
│ Skills     手动编写 + ZIP 导入           │
│ 认证       JWT + OAuth2 (GitHub)         │
│ 邮件       gomail (SMTP)                 │
│ 邀请       分享链接 + 邮箱               │
│ 国际化     next-intl (前端)               │
│           go-i18n (后端)                  │
│ API文档    Swagger (swaggo/swag)         │
│ 跨域       gin-contrib/cors              │
└──────────────────────────────────────────┘
```

### 选型理由

| 技术 | 理由 |
|------|------|
| **Gin** | 高性能、生态成熟、中文社区活跃 |
| **GORM** | Go 最流行的 ORM，支持 MySQL 全面 |
| **MySQL 8.0+** | 关系型数据、事务支持、稳定可靠 |
| **Cobra** | Go CLI 标准库，`anserflow server/init` 命令 |
| **embed** | Go 1.16+ 原生静态文件嵌入，编译为单一可执行文件 |
| **Viper** | Go 配置管理标准库，支持 YAML/ENV 多源加载 |
| **Zap** | Uber 开源高性能结构化日志库 |
| **validator** | Go 结构体校验标准库，API 请求参数校验 |
| **Casbin** | 灵活的 RBAC/ABAC 权限模型，满足组织角色管理 |
| **Next.js SPA** | `output: "export"` 模式，产物可直接嵌入 Go 二进制 |
| **Redis** | 缓存 + WebSocket 分布式 Pub/Sub + Asynq 任务队列，一个组件覆盖三个场景 |
| **Gorilla WebSocket** | Go 社区最成熟的 WebSocket 库 |
| **Asynq** | Go 原生、基于 Redis、支持重试/超时/优先级/死信队列，零额外运维 |
| **Eino** | 字节跳动开源、12k+ Stars、Graph/Workflow 多 Agent 编排、流式原生支持、中文社区强 |
| **Docker SDK** | Agent 编码沙箱隔离，资源限制、自动清理 |
| **TanStack Query** | 服务端状态管理，自动缓存/重取/去重，SPA 模式完美兼容 |
| **Zustand** | 极轻量客户端状态管理（侧栏、弹窗、Tab 切换状态） |
| **React Hook Form + Zod** | 高性能表单 + 声明式校验，与 shadcn/ui 深度集成 |
| **TanStack Table** | 无头表格库，Issue 列表/Agent 列表/成员表格 |
| **Recharts** | 图表库，Dashboard 数据可视化 |
| **next-themes** | 暗色/亮色主题切换，与 Tailwind CSS 原生配合 |
| **next-intl** | Next.js App Router 原生 i18n、静态导出兼容、TypeScript 类型安全、ICU 消息格式 |
| **Framer Motion** | 动效库，页面过渡、Issue Tab 展开/折叠动画 |
| **Sonner** | 轻量 Toast 通知，操作反馈 |
| **date-fns** | 日期处理，轻量 tree-shakable |
| **lucide-react** | 图标库，与 shadcn/ui 配套 |
| **shadcn/ui** | 无捆绑、可定制、基于 Radix 的可访问组件 |
| **Tauri 2.x** | 比 Electron 轻量、Rust 内核、支持移动端 |
| **gomail** | Go 邮件发送库，支持 SMTP/SSL，用于邮箱邀请和通知 |
| **swaggo/swag** | Swagger/OpenAPI 文档自动生成，便于前后端联调 |
| **gin-contrib/cors** | Gin 官方 CORS 中间件，SPA 跨域支持 |
| **go-i18n** | Go 标准 i18n 库，CLI 管理翻译文件，复数规则支持，用于邮件模板和 API 错误消息 |

### 框架补充说明

> 以下为生产级 Gin 项目的标准配套设施，确保系统可维护、可观测、可扩展。

#### Viper — 配置管理

`github.com/spf13/viper` 统一管理 `config.yaml`，支持环境变量覆盖（如 `DB_HOST`、`REDIS_ADDR` 覆盖配置文件值），生产环境敏感信息通过环境变量注入。

```go
viper.SetConfigName("config")
viper.AddConfigPath(".")
viper.AutomaticEnv() // ENV 自动覆盖
viper.ReadInConfig()
```

#### Zap — 结构化日志

`go.uber.org/zap` 替代标准库 `log`，支持 JSON 格式输出、日志分级（Debug/Info/Warn/Error）、按时间/大小自动切割。GORM 可直接接入 Zap 作为日志后端：

```go
logger, _ := zap.NewProduction()
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: logger.New(gormLogger.Info, &gormLogger.Config{LogLevel: gormLogger.Info}),
})
```

#### go-playground/validator — 请求校验

Gin 原生集成了 `github.com/go-playground/validator`，通过 struct tag 声明校验规则：

```go
type CreateIssueReq struct {
    Title       string `json:"title" binding:"required,min=1,max=256"`
    Priority    string `json:"priority" binding:"required,oneof=p0 p1 p2 p3 p4"`
    ProjectID   uint   `json:"project_id" binding:"required"`
}
```

#### Casbin — 权限控制

`github.com/casbin/casbin` 实现 RBAC，支持组织级角色（owner/admin/member）和资源级权限（项目/Issue/Agent）。策略模型存储在 MySQL 中，运行时动态加载：

```ini
[request_definition]
r = sub, obj, act
[policy_definition]
p = sub, obj, act
[role_definition]
g = _, _
[policy_effect]
e = some(where (p.eft == allow))
[matchers]
m = g(r.sub, p.sub) && keyMatch(r.obj, p.obj) && regexMatch(r.act, p.act)
```

#### Swagger — API 文档

`github.com/swaggo/swag` + `github.com/swaggo/gin-swagger`，通过代码注解自动生成 OpenAPI 3.0 文档，开发环境访问 `/swagger/index.html`：

```go
// @title           AnserFlow API
// @version         1.0
// @host            localhost:8080
// @BasePath        /api
func main() { /* ... */ }
```

#### CORS — 跨域支持

`github.com/gin-contrib/cors` 允许 SPA 前端和 Tauri WebView 跨域访问 API：

```go
r.Use(cors.New(cors.Config{
    AllowOrigins: []string{"http://localhost:3000", "tauri://localhost"},
    AllowMethods: []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
    AllowHeaders: []string{"Origin", "Content-Type", "Authorization"},
}))
```

#### 优雅关闭

Go 标准库 `signal` + `http.Server.Shutdown`，确保收到 SIGINT/SIGTERM 时完成进行中的请求再退出：

```go
srv := &http.Server{Addr: ":8080", Handler: r}
go srv.ListenAndServe()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

#### 健康检查

`/api/health` 端点返回数据库、Redis 连通性，供 K8s/Docker 探活：

```go
r.GET("/api/health", func(c *gin.Context) {
    c.JSON(200, gin.H{
        "status": "ok",
        "db":     checkDB(),
        "redis":  checkRedis(),
    })
})
```

#### OAuth2 — 第三方登录（GitHub）

AnserFlow 支持 GitHub OAuth 登录，降低注册门槛。前端 `login` 页面提供「GitHub 登录」按钮，跳转到 GitHub 授权页；回调后后端完成账号创建/绑定并返回 JWT。

```
用户点击 GitHub 登录
        │
        ▼
GET /api/auth/github/login  → 302 跳转 GitHub
        │
        ▼
用户授权 → GitHub 回调 GET /api/auth/github/callback?code=xxx
        │
        ▼
后端：code → access_token → 获取 GitHub 用户信息
        │
        ▼
┌─────────────────────────────────────────┐
│ github_id 已存在?                        │
│  ├── 是 → 直接生成 JWT，登录成功         │
│  └── 否 → 已登录用户？（绑定）            │
│           ├── 是 → 绑定 github_id 到账户 │
│           └── 否 → 自动注册新用户         │
└─────────────────────────────────────────┘
        │
        ▼
返回 JWT → 前端存储 → 跳转到 /admin/dashboard
```

**非 GitHub 用户的注册兼容**：支持传统邮箱+密码注册（通过 `/api/auth/register` / `/api/auth/login`），与 OAuth 用户共用 `users` 表，`password_hash` 为 NULL 表示仅 OAuth 登录。

```go
// internal/handler/auth.go
func (h *AuthHandler) GitHubCallback(c *gin.Context) {
    code := c.Query("code")
    // 1. code → access_token
    token, _ := h.oauth.Exchange(ctx, code)
    // 2. access_token → GitHub user info
    ghUser, _ := h.oauth.GetUser(ctx, token)
    // 3. 查找或创建用户
    user := h.userRepo.FindOrCreateByGitHub(ghUser)
    // 4. 生成 JWT
    jwtToken, _ := h.jwtService.Generate(user.ID)
    // 5. 重定向到前端（URL 参数携带 JWT）
    c.Redirect(http.StatusFound,
        fmt.Sprintf("/admin/dashboard?token=%s", jwtToken))
}
```

### 前端框架补充说明

> 以下为 Next.js SPA 生产级项目标准配套设施，覆盖状态管理、数据请求、表单、动效、主题等核心能力。

#### TanStack Query — 服务端状态管理

`@tanstack/react-query` 负责所有 API 数据请求，提供自动缓存、后台刷新、请求去重、乐观更新。SPA 模式下完全运行在客户端：

```tsx
// hooks/use-issues.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

export function useIssues(projectId: number) {
  return useQuery({
    queryKey: ['issues', projectId],
    queryFn: () => fetch(`/api/projects/${projectId}/issues`).then(r => r.json()),
    staleTime: 30 * 1000,  // 30s 内不重新请求
  })
}

export function useCreateIssue() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (data: CreateIssueReq) =>
      fetch('/api/issues', { method: 'POST', body: JSON.stringify(data) }).then(r => r.json()),
    onSuccess: (_, vars) => {
      qc.invalidateQueries({ queryKey: ['issues', vars.project_id] })
    },
  })
}
```

#### Zustand — 客户端状态管理

`zustand` 管理纯客户端状态：侧栏展开/收起、弹窗开关、Tab 切换状态、WebSocket 连接状态等。极简 API，无 Provider 包裹：

```tsx
// stores/sidebar.ts
import { create } from 'zustand'

export const useSidebar = create<SidebarState>((set) => ({
  isOpen: true,
  toggle: () => set((s) => ({ isOpen: !s.isOpen })),
}))
```

#### React Hook Form + Zod — 表单与校验

`react-hook-form` 提供高性能非受控表单，`zod` 声明式 schema 校验，与 shadcn/ui 的 `<Form />` 组件深度集成：

```tsx
// components/agent-form.tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const agentSchema = z.object({
  name: z.string().min(1, '名称不能为空').max(64),
  role_label: z.string().max(64),               // 自定义角色标签（如 PM / 前端 / 后端）
  system_prompt: z.string().max(200, '人设 1-2 句话即可，调度行为由 Eino Skill 定义'),
  runtime_id: z.number().min(1, '请选择运行时'),
  // runtime_config 由前端根据 runtimes.config_schema 动态生成表单字段
})

type AgentFormData = z.infer<typeof agentSchema>

export function AgentForm() {
  const form = useForm<AgentFormData>({ resolver: zodResolver(agentSchema) })
  // ...
}
```

#### TanStack Table — 数据表格

`@tanstack/react-table` 无头表格库，配合 shadcn/ui 的 `<DataTable />` 组件实现排序、筛选、分页、行选择：

```tsx
// features/issues/components/issue-table.tsx
const columns: ColumnDef<Issue>[] = [
  { accessorKey: 'title', header: '标题' },
  { accessorKey: 'status', header: '状态' },
  { accessorKey: 'priority', header: '优先级' },
]

<DataTable columns={columns} data={issues} />
```

#### Recharts — 数据可视化

`recharts` 用于 Dashboard 仪表盘图表，支持折线图、柱状图、饼图等常用图表类型：

```tsx
// features/dashboard/components/issue-stats-chart.tsx
import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts'

const data = [
  { status: 'backlog', count: 12 },
  { status: 'todo', count: 8 },
  { status: 'in_progress', count: 5 },
  { status: 'in_review', count: 3 },
  { status: 'done', count: 20 },
]

<ResponsiveContainer width="100%" height={300}>
  <BarChart data={data}>
    <CartesianGrid strokeDasharray="3 3" />
    <XAxis dataKey="status" />
    <YAxis />
    <Tooltip />
    <Bar dataKey="count" fill="var(--primary)" radius={[4, 4, 0, 0]} />
  </BarChart>
</ResponsiveContainer>
```

#### lucide-react — 图标库

`lucide-react` 提供 1000+ 开源 SVG 图标，按需导入，与 shadcn/ui 组件配套使用：

```tsx
import { Plus, Trash2, Settings, Users, FolderKanban, Bot, MessageSquare } from 'lucide-react'

<Button><Plus className="mr-2 h-4 w-4" />创建</Button>
<Button variant="destructive"><Trash2 className="h-4 w-4" /></Button>
```

> 图标命名语义化：`FolderKanban`（看板）、`Bot`（Agent）、`MessageSquare`（群聊）。

#### next-themes — 主题切换

`next-themes` 提供暗色/亮色主题切换，与 Tailwind CSS `dark:` 前缀原生配合，支持系统偏好跟随：

```tsx
import { useTheme } from 'next-themes'

<Button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
  <Sun className="dark:hidden" />
  <Moon className="hidden dark:block" />
</Button>
```

#### Framer Motion — 动效

`framer-motion` 提供页面过渡动画、Issue 行展开/折叠、模态框弹出动效：

```tsx
import { motion, AnimatePresence } from 'framer-motion'

<AnimatePresence>
  {isOpen && (
    <motion.div initial={{ opacity: 0, y: 20 }} animate={{ opacity: 1, y: 0 }} exit={{ opacity: 0 }}>
      <Modal />
    </motion.div>
  )}
</AnimatePresence>
```

#### Sonner — Toast 通知

`sonner` 替代传统 toast 库，支持 Promise 状态、富文本、自定义样式：

```tsx
import { toast } from 'sonner'

toast.promise(createAgent(data), {
  loading: '创建 Agent 中...',
  success: 'Agent 创建成功',
  error: '创建失败',
})
```

#### 日期处理

`date-fns` 纯函数日期工具，tree-shakable，按需导入：

```tsx
import { format, formatDistanceToNow } from 'date-fns'
import { zhCN } from 'date-fns/locale'

format(new Date(), 'yyyy-MM-dd HH:mm')
formatDistanceToNow(issue.created_at, { locale: zhCN, addSuffix: true }) // "3 小时前"
```

#### 环境变量管理

Next.js SPA 模式下，`NEXT_PUBLIC_` 前缀变量在构建时内联，敏感信息必须保留在后端：

```bash
# .env.local
NEXT_PUBLIC_API_BASE=http://localhost:8080/api
NEXT_PUBLIC_WS_URL=ws://localhost:8080/ws
```

#### 前端目录结构

采用 features 模块化架构，按业务功能组织代码。以下以 `admin/` 包为例，后台运行时通过 `basePath: '/admin'` 暴露为 `/admin/*`：

```
src/
├── app/
│       ├── layout.tsx          # 根布局（Provider 包裹）
│       ├── page.tsx            # /admin → 重定向到 /admin/dashboard
│       ├── login/              # /admin/login
│       ├── dashboard/          # /admin/dashboard
│       ├── agents/             # Agent 管理页
│       ├── projects/           # 项目 & Issue 页
│       └── groups/             # 群组 & 群聊页
├── components/                 # 共享 UI 组件
│   ├── ui/                     # shadcn/ui 基础组件
│   └── layout/                 # 布局组件（Sidebar/Navbar）
├── features/                   # 业务功能模块
│   ├── agents/
│   │   ├── api/               # API 请求 & mutation
│   │   ├── components/        # Agent 卡片/表单/列表
│   │   └── schemas/           # Zod 校验 schema
│   ├── issues/
│   ├── projects/
│   ├── organizations/         # 组织管理 + 成员邀请
│   ├── groups/                # 群组 & 群聊 WebSocket 通信
│   ├── skills/                # Skills 管理
│   ├── notifications/         # 通知中心
│   ├── dashboard/             # 仪表盘图表
│   └── settings/              # 组织/全局设置
├── hooks/                      # 自定义 Hook
├── stores/                     # Zustand Store
├── lib/                        # 工具函数 & API client
│   ├── api.ts                 # 统一 fetch 封装
│   └── ws.ts                  # WebSocket 客户端
└── types/                      # TypeScript 类型定义
```

#### 代码质量工具

```json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx",
    "format": "prettier --write .",
    "type-check": "tsc --noEmit"
  }
}
```

| 工具 | 用途 |
|------|------|
| **ESLint** + `@next/eslint-plugin` | 代码规范检查 |
| **Prettier** + `prettier-plugin-tailwindcss` | 代码格式化 + Tailwind 类名排序 |
| **TypeScript strict** | `tsconfig.json` 启用 `strict: true` |

### 国际化（i18n）

> AnserFlow 面向全球用户，前端 UI + 后端邮件模板 + API 错误消息均需多语言支持。首期支持中文（zh-CN）和英文（en-US），架构预留扩展。

#### 整体架构

```
┌──────────────────────────────────────────────┐
│ 前端 (next-intl)                              │
│ ├── admin/messages/   ← 后台管理翻译           │
│ ├── desktop/messages/ ← 桌面端翻译             │
│ └── packages/shared-ui/messages/ ← 公共翻译    │
├──────────────────────────────────────────────┤
│ 后端 (go-i18n)                                │
│ ├── 邮件模板 i18n   → 邀请/通知邮件双语         │
│ └── API 错误码映射  → 前端根据 locale 展示      │
└──────────────────────────────────────────────┘
```

**设计原则**：

| 原则 | 说明 |
|------|------|
| 前端驱动 | UI 文案由前端 `next-intl` 管理，locale 存储在 localStorage + 已登录用户同步到 `users.locale`，URL 不含 locale 段 |
| 后端错误码 | API 返回国际化错误码（如 `ERR_ISSUE_NOT_FOUND`），前端映射为当前语言文案 |
| 邮件双语 | 邀请邮件根据用户语言偏好发送中/英文版本 |
| 翻译共享 | 公共 UI（按钮、表单校验提示）抽取到 `packages/shared-ui/messages/` 复用 |

#### 后端：go-i18n 错误码映射实现

后端 API 返回统一的国际化错误码，前端根据当前 locale 映射为对应语言文案。

**错误码枚举**（完整清单）：

| 错误码 | HTTP 状态码 | 说明 |
|--------|-----------|------|
| `ERR_ISSUE_NOT_FOUND` | 404 | Issue 不存在 |
| `ERR_PROJECT_NOT_FOUND` | 404 | 项目不存在 |
| `ERR_AGENT_NOT_FOUND` | 404 | Agent 不存在 |
| `ERR_ORG_NOT_FOUND` | 404 | 组织不存在 |
| `ERR_SKILL_NOT_FOUND` | 404 | Skill 不存在 |
| `ERR_USER_NOT_FOUND` | 404 | 用户不存在 |
| `ERR_VALIDATION_FAILED` | 400 | 请求参数校验失败 |
| `ERR_UNAUTHORIZED` | 401 | 未认证（JWT 过期/无效） |
| `ERR_PERMISSION_DENIED` | 403 | 权限不足 |
| `ERR_ORG_LIMIT_EXCEEDED` | 403 | 组织并发 Agent 数已达上限 |
| `ERR_SANDBOX_TIMEOUT` | 500 | Docker 沙箱执行超时 |
| `ERR_SANDBOX_CREATE_FAILED` | 500 | Docker 沙箱创建失败 |
| `ERR_GIT_CLONE_FAILED` | 500 | Git 仓库克隆失败 |
| `ERR_GIT_PUSH_FAILED` | 500 | Git 推送失败 |
| `ERR_LLM_API_ERROR` | 502 | LLM API 调用失败 |
| `ERR_INVITE_EXPIRED` | 400 | 邀请链接已过期 |
| `ERR_INVITE_MAX_USES` | 400 | 邀请链接已达使用上限 |
| `ERR_RUNTIME_NOT_FOUND` | 404 | 运行时不存在 |
| `ERR_INTERNAL` | 500 | 服务器内部错误 |

**Gin 错误响应中间件**：

```go
// internal/middleware/i18n_error.go
package middleware

import (
    "net/http"
    "github.com/BurntSushi/toml"
    "github.com/nicksnyder/go-i18n/v2/i18n"
    "golang.org/x/text/language"
    "github.com/gin-gonic/gin"
)

var bundle *i18n.Bundle

func InitI18n() {
    bundle = i18n.NewBundle(language.English)
    bundle.RegisterUnmarshalFunc("toml", toml.Unmarshal)
    bundle.MustLoadMessageFile("locales/active.en.toml")
    bundle.MustLoadMessageFile("locales/active.zh-CN.toml")
}

// APIError 统一错误响应结构
type APIError struct {
    Code    string `json:"code"`    // 错误码（如 ERR_ISSUE_NOT_FOUND）
    Message string `json:"message"` // 当前 locale 的文案
}

// RespondError 返回国际化错误响应
func RespondError(c *gin.Context, statusCode int, errorCode string, templateData map[string]interface{}) {
    // 确定 locale：优先用户设置 → Accept-Language 头 → 默认 en
    locale := getUserLocale(c)
    localizer := i18n.NewLocalizer(bundle, locale)

    msg, err := localizer.Localize(&i18n.LocalizeConfig{
        MessageID:    errorCode,
        TemplateData: templateData,
    })
    if err != nil {
        msg = errorCode // fallback 到错误码本身
    }

    c.JSON(statusCode, APIError{Code: errorCode, Message: msg})
    c.Abort()
}

func getUserLocale(c *gin.Context) string {
    // 优先级：已登录用户 users.locale > Accept-Language > 默认 en
    if userLocale, exists := c.Get("user_locale"); exists {
        return userLocale.(string)
    }
    acceptLang := c.GetHeader("Accept-Language")
    if strings.HasPrefix(acceptLang, "zh") {
        return "zh-CN"
    }
    return "en"
}
```

**翻译文件示例**：

```toml
# locales/active.zh-CN.toml
[ERR_ISSUE_NOT_FOUND]
other = "Issue #{{.IssueID}} 不存在"

[ERR_VALIDATION_FAILED]
other = "请求参数校验失败: {{.Detail}}"

[ERR_PERMISSION_DENIED]
other = "权限不足，无法执行此操作"

[ERR_ORG_LIMIT_EXCEEDED]
other = "组织 {{.OrgName}} 并发 Agent 数已达上限 ({{.Max}})，请等待或提升限额"

[ERR_SANDBOX_TIMEOUT]
other = "Docker 沙箱执行超时（超过 {{.Timeout}} 秒）"
```

```toml
# locales/active.en.toml
[ERR_ISSUE_NOT_FOUND]
other = "Issue #{{.IssueID}} not found"

[ERR_VALIDATION_FAILED]
other = "Validation failed: {{.Detail}}"

[ERR_PERMISSION_DENIED]
other = "Permission denied"

[ERR_ORG_LIMIT_EXCEEDED]
other = "Organization {{.OrgName}} has reached the max concurrent agent limit ({{.Max}}). Please wait or upgrade."

[ERR_SANDBOX_TIMEOUT]
other = "Docker sandbox execution timed out (exceeded {{.Timeout}} seconds)"
```

> **前端消费**：前端 `apiFetch` 封装层拦截 APIError，根据 `code` 从 `messages/{locale}.json` 的 `Errors` 段读取对应文案作为 fallback。后端错误码优先显示，前端翻译仅为降级兜底。

#### 前端：next-intl

> `next-intl` 是 Next.js App Router 最主流的 i18n 库，原生支持 `output: "export"` 静态导出模式。

##### 安装与配置

```bash
npm install next-intl
```

```ts
// admin/next.config.ts
import createNextIntlPlugin from 'next-intl/plugin'

const withNextIntl = createNextIntlPlugin('./src/i18n/request.ts')

const nextConfig = {
  basePath: '/admin',
  output: 'export',
  distDir: 'dist',
}

export default withNextIntl(nextConfig)
```

```ts
// admin/src/i18n/request.ts
import { getRequestConfig } from 'next-intl/server'

// 静态导出模式：locale 从客户端 localStorage / navigator.language / 用户设置获取
// URL 不含 locale 段，所有页面统一走 /admin/* 路径
export default getRequestConfig(async () => {
  // 静态导出时使用默认 locale，实际切换在客户端完成
  return {
    locale: 'zh-CN',
    messages: (await import(`../../messages/zh-CN.json`)).default,
  }
})
```

**Locale 检测与切换**（纯客户端，不经过 URL）：

```ts
// admin/src/lib/locale.ts
export function detectLocale(): string {
  // 优先级：已登录用户设置 > localStorage > 浏览器语言 > 默认 zh-CN
  const stored = localStorage.getItem('anserflow-locale')
  if (stored) return stored

  const browserLang = navigator.language
  if (browserLang.startsWith('zh')) return 'zh-CN'
  if (browserLang.startsWith('en')) return 'en-US'

  return 'zh-CN'
}

export function setLocale(locale: string) {
  localStorage.setItem('anserflow-locale', locale)
  // 如果已登录，同步到后端 users.locale
  // window.location.reload() 或触发 next-intl 的 setLocale
}
```

```tsx
// 语言切换组件（无 URL 变化，纯客户端切换）
import { useRouter } from 'next/navigation'

export function LanguageSwitcher() {
  const switchTo = (locale: string) => {
    setLocale(locale)
    window.location.reload() // 重新加载页面以切换翻译包
  }

  return (
    <select onChange={(e) => switchTo(e.target.value)} value={detectLocale()}>
      <option value="zh-CN">中文</option>
      <option value="en-US">English</option>
    </select>
  )
}
```

##### 目录结构

```
admin/
├── messages/
│   ├── zh-CN.json          # 后台管理 - 中文
│   └── en-US.json          # 后台管理 - 英文
├── src/
│   ├── i18n/
│   │   └── request.ts      # next-intl 配置
│   └── app/                 # 统一 /admin/* 路由（不含 locale 段）
│       ├── layout.tsx       # NextIntlClientProvider
│       ├── page.tsx         # /admin → 重定向到 dashboard
│       ├── dashboard/
│       ├── agents/
│       └── projects/

desktop/
├── messages/               # 桌面端翻译（同上结构）

packages/shared-ui/
├── messages/               # 公共翻译（按钮、校验提示等）
│   ├── zh-CN.json
│   └── en-US.json
```

##### 翻译文件示例

```json
// admin/messages/zh-CN.json
{
  "Nav": {
    "dashboard": "仪表盘",
    "agents": "智能体",
    "projects": "项目",
    "skills": "技能",
    "settings": "设置"
  },
  "Issue": {
    "title": "Issue 标题",
    "status": "状态",
    "priority": "优先级",
    "assignee": "负责人",
    "create": "创建 Issue",
    "noResults": "暂无 Issue"
  },
  "Common": {
    "save": "保存",
    "cancel": "取消",
    "delete": "删除",
    "confirm": "确认",
    "loading": "加载中...",
    "error": "出错了"
  }
}
```

```json
// admin/messages/en-US.json
{
  "Nav": {
    "dashboard": "Dashboard",
    "agents": "Agents",
    "projects": "Projects",
    "skills": "Skills",
    "settings": "Settings"
  },
  "Issue": {
    "title": "Issue Title",
    "status": "Status",
    "priority": "Priority",
    "assignee": "Assignee",
    "create": "Create Issue",
    "noResults": "No Issues"
  },
  "Common": {
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "confirm": "Confirm",
    "loading": "Loading...",
    "error": "Something went wrong"
  }
}
```

##### 组件使用

```tsx
// admin/src/app/dashboard/page.tsx
import { useTranslations } from 'next-intl'

export default function DashboardPage() {
  const t = useTranslations('Nav')
  const tIssue = useTranslations('Issue')

  return (
    <div>
      <h1>{t('dashboard')}</h1>           {/* "仪表盘" 或 "Dashboard" */}
      <span>{tIssue('noResults')}</span>  {/* "暂无 Issue" 或 "No Issues" */}
    </div>
  )
}
```

##### 日期/数字本地化

```tsx
import { useFormatter } from 'next-intl'

const format = useFormatter()

// 日期
format.dateTime(issue.createdAt, {
  year: 'numeric', month: 'long', day: 'numeric'
})
// zh-CN → "2026年5月13日"
// en-US → "May 13, 2026"

// 相对时间
format.relativeTime(issue.createdAt)
// zh-CN → "3小时前"
// en-US → "3 hours ago"
```

#### 翻译管理

| 阶段 | 方式 |
|------|------|
| 当前 L1-L4 | 手动编辑 JSON 文件 + `goi18n merge` 合并新增翻译 key |
| Phase 2 | 如翻译量明显增加，再单独立项接入 Crowdin / Lokalise |

```bash
# Go 后端翻译管理
goi18n extract           # 从 Go 源码提取待翻译消息 → translate.zh-CN.json
goi18n merge active.*.json translate.*.json  # 合并新增 key
```
