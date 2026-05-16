# AnserFlow - Client / Frontend

---

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


---
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
│   ├── groups/                # 群组 & 群聊 WebSocket 通信（群管理 + 聊天复用 conversations）
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

---

### 国际化（i18n）

> AnserFlow 面向全球用户，前端 UI + 后端邮件模板 + API 错误消息均需多语言支持。首期支持中文（zh-CN）和英文（en-US），架构预留扩展。

#### 整体架构

```
┌──────────────────────────────────────────────┐
│ 前端 (next-intl)                              │
│ ├── admin/messages/   ← 后台管理翻译           │
│ ├── client/messages/  ← 客户端翻译             │
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


---

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

client/
├── messages/               # 客户端翻译（同上结构）

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

---

### 客户端（Web SPA）

客户端统一使用 Next.js 14 SPA（static export），浏览器直接访问，界面以 IM 聊天为核心交互模式。

- **技术栈**：与 admin 一致（Next.js SPA + shadcn/ui + Tailwind CSS + TanStack Query + Zustand）
- **核心路由**：`/dashboard`、`/projects/:id`、`/chat`、`/invite/:token`
- **部署方式**：Go embed 嵌入或独立部署，浏览器访问
- **通知方式**：浏览器 Notification API + WebSocket 实时推送

**`/chat` IM 四栏布局**：

```
/chat                          ← 主聊天页面（四栏布局）
  ① 第一栏：导航栏（窄栏，图标 + 用户头像）
    ├── 用户头像（点击进入个人中心）
    ├── 💬 聊天图标（激活第二栏为聊天视图）
    ├── 📁 项目图标（激活第二栏为项目视图）
    └── ⚙️ 个人中心（设置、通知偏好、退出）

  ② 第二栏：会话列表 / 项目列表
    [聊天视图]
    ├── 搜索/新建双人聊（支持搜索用户和 Agent）
    ├── 双人聊列表项
    │     - 人+人：对方用户头像 + 昵称 + 最后消息 + 未读数
    │     - 人+Agent：Agent 头像 + 名称 + Agent 标识 + 最后消息 + 未读数
    ├── 群聊列表项（群名 + 最后消息 + 未读数）
    │     - 按最后消息时间统一排序（双人聊和群聊混合排列）
    [项目视图]
    ├── 我的项目列表（项目名 + 进度概要）

  ③ 第三栏：聊天窗口
    /chat/:group_id            ← 选中会话后展示聊天内容
      - 顶部：会话标题（direct: 对方昵称/Agent名称，从成员信息派生；group: 群名）
      - 中部：消息列表（复用现有 MessageList 组件）
      - 底部：输入框（条件渲染，见下方）

  ④ 第四栏：项目上下文面板
    /chat/:group_id/context    ← 当前会话关联的项目信息
      - 项目基本情况（名称、仓库、成员）
      - 关联 Issue 列表（状态 Tab：backlog/todo/in_progress/in_review/done）
      - 参与的 Agent（角色、状态、绑定 Skill）
      - 进度概览（Issue 完成率、PR 状态、最近活动）
```

**Issue 详情页（展开 Issue 行后显示）**：

第四栏的 Issue 列表中，点击某条 Issue 后展开为详情视图，核心是**时间线面板** + **执行控制**：

```
┌─ Issue #42: 实现登录表单组件 ────────────────────────────────────┐
│  状态: in_progress   优先级: P1   负责人: Agent-前端              │
│                                                                  │
│  ┌─ 工具栏 ────────────────────────────────────────────────────┐  │
│  │  [编辑]  [⏸ 暂停]  [⏹ 停止]              筛选: [全部 ▾]    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ 时间线 ────────────────────────────────────────────────────┐  │
│  │                                                              │  │
│  │  12:00  system   状态变更: backlog → todo                    │  │
│  │  12:01  system   状态变更: todo → in_progress                │  │
│  │  12:02  agent    📝 开始编码: 正在读取 Issue 描述...          │  │
│  │  12:05  agent    📄 生成文件: src/login.tsx                   │  │
│  │  12:05  agent    📄 生成文件: src/login.test.tsx              │  │
│  │  12:08  agent    ✅ 运行测试: 3 passed, 1 failed              │  │
│  │  12:08  agent    ❌ FAIL: login.test.tsx > 密码验证           │  │
│  │  12:09  agent    🔧 正在修复 login.test.tsx                   │  │
│  │  12:12  agent    ✅ 运行测试: 4 passed, 0 failed              │  │
│  │  12:12  agent    📦 commit + push → PR #15                   │  │
│  │  12:12  system   状态变更: in_progress → in_review           │  │
│  │  ───────────────── 自动滚动到底部 ─────────────────          │  │
│  │                                                              │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ 追加提示词 ────────────────────────────────────────────────┐  │
│  │  [                                ] [发送并重新执行]          │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ Token 消耗 ────────────────────────────────────────────────┐  │
│  │  输入: 12,450 tokens  输出: 3,200 tokens  合计: 15,650       │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

**时间线事件类型与样式**：

| event_type | source | 图标 | 样式 | 说明 |
|------------|--------|------|------|------|
| `status_change` | system | 无 | 灰色斜体 | 状态流转记录，显示 `旧状态 → 新状态` + 触发原因 |
| `agent_log` | agent | 按动作分类 | 默认文本 | 沙箱执行日志，action 决定图标（见下表） |
| `human_prompt` | user | ✏️ | 蓝色高亮 | 人工追加提示词，显示用户名 + 内容 |
| `system_note` | system | 📋 | 灰色 | 编辑/备注等管理操作 |

**agent_log 的 action 分类与图标**：

| action | 图标 | 说明 |
|--------|------|------|
| `generate` | 📄 | 生成文件 |
| `test` / `test_pass` | ✅ | 测试通过 |
| `test_fail` / `fix` | ❌ / 🔧 | 测试失败 / 修复中 |
| `commit` | 📦 | 提交代码 |
| `create_pr` | 🔀 | 创建 PR |
| `error` | ⚠️ | 执行异常 |
| `paused` | ⏸ | 执行暂停 |
| 其他 | 📝 | 通用日志 |

**时间线数据流**：

```
沙箱 opencode stdout
    │
    ▼
Worker streamLogs() 捕获
    ├── 写入 agent_logs 表（结构化：action / status / output）
    ├── 写入 issue_timeline 表（展示用：source / event_type / content）
    └── WebSocket 推送 {type: "agent_log", text, ts}
            │
            ▼
前端 IssueTimeline 组件
    ├── 历史加载: GET /api/issues/:id/timeline?page=1&size=50
    ├── 实时追加: WebSocket issue:{id} 频道订阅
    └── 筛选过滤: 前端内存过滤（数据量可控，无需服务端筛选）
```

**前端组件结构**：

```tsx
// features/issues/components/issue-detail.tsx

export function IssueDetail({ issueId }: { issueId: number }) {
  // 加载 Issue 元信息
  const { data: issue } = useQuery({
    queryKey: ['issue', issueId],
    queryFn: () => fetch(`/api/issues/${issueId}`).then(r => r.json()),
  })

  return (
    <div className="flex flex-col h-full">
      {/* 头部：标题 + 状态 + 优先级 + 负责人 */}
      <IssueHeader issue={issue} />

      {/* 工具栏：编辑/暂停/停止 + 日志筛选 */}
      <IssueToolbar issueId={issueId} status={issue?.status} />

      {/* 时间线：核心区域，占满剩余空间 */}
      <IssueTimeline issueId={issueId} />

      {/* 底部：追加提示词输入框（in_progress / paused / todo 状态显示） */}
      <PromptInput issueId={issueId} status={issue?.status} />

      {/* Token 消耗统计（折叠显示） */}
      <TokenUsageSummary issueId={issueId} />
    </div>
  )
}
```

```tsx
// features/issues/components/issue-timeline.tsx

export function IssueTimeline({ issueId }: { issueId: number }) {
  const bottomRef = useRef<HTMLDivElement>(null)
  const [filter, setFilter] = useState<string>('all')

  // ① 历史数据加载（分页）
  const { data: timeline } = useQuery({
    queryKey: ['issue-timeline', issueId],
    queryFn: () =>
      fetch(`/api/issues/${issueId}/timeline?page=1&size=100`).then(r => r.json()),
  })

  // ② WebSocket 实时追加
  const queryClient = useQueryClient()
  useEffect(() => {
    const unsub = ws.subscribe(`issue:${issueId}`, (msg) => {
      if (msg.type === 'agent_log' || msg.type === 'status_change') {
        queryClient.setQueryData(
          ['issue-timeline', issueId],
          (old) => [...(old || []), {
            id: Date.now(),           // 临时 ID，刷新后由服务端 ID 替换
            source: 'agent',          // 或 msg.source
            event_type: msg.type,
            content: msg.text || msg.hint,
            created_at: new Date(msg.ts * 1000).toISOString(),
          }]
        )
      }
    })
    return () => unsub()
  }, [issueId])

  // ③ 自动滚动到底部
  useEffect(() => { bottomRef.current?.scrollIntoView({ behavior: 'smooth' }) }, [timeline])

  // ④ 前端筛选
  const filtered = (timeline || []).filter(item => {
    if (filter === 'all') return true
    if (filter === 'agent') return item.source === 'agent'
    if (filter === 'system') return item.source === 'system'
    if (filter === 'human') return item.event_type === 'human_prompt'
    return true
  })

  return (
    <div className="flex-1 overflow-y-auto p-3 space-y-1">
      {filtered.map(item => (
        <TimelineItem key={item.id} item={item} />
      ))}
      <div ref={bottomRef} />
    </div>
  )
}

// 单条时间线
function TimelineItem({ item }: { item: TimelineEvent }) {
  const icon = getActionIcon(item)       // 根据 event_type + action 映射图标
  const style = getEventStyle(item)       // 根据 event_type 映射样式类
  return (
    <div className={`flex items-start gap-2 text-sm py-0.5 ${style}`}>
      <span className="text-xs text-muted-foreground w-12 shrink-0">
        {formatTime(item.created_at)}       {/* "12:05" */}
      </span>
      <span className="text-xs text-muted-foreground w-14 shrink-0">
        {item.source}                         {/* "agent" / "system" / "张三" */}
      </span>
      <span className="shrink-0">{icon}</span>
      <span className="flex-1">{item.content}</span>
    </div>
  )
}
```

**工具栏按钮状态控制**：

| Issue 状态 | 编辑 | 暂停 | 恢复 | 停止 | 追加提示词 |
|-----------|------|------|------|------|----------|
| `backlog` | ✅ | - | - | - | - |
| `todo` | ✅ | - | - | - | ✅ |
| `in_progress` | - | ✅ | - | ✅ | ✅ |
| `paused` | - | - | ✅ | ✅ | ✅ |
| `in_review` | - | - | - | - | ✅（PR 被拒绝后退回 todo 时复用沙箱） |
| `done` | - | - | - | - | - |

**API 接口**：

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/issues/:id` | GET | Issue 详情（标题/状态/优先级/负责人/Token 统计） |
| `/api/issues/:id/timeline` | GET | 时间线列表（分页，?page=1&size=50） |
| `/api/issues/:id/pause` | POST | 暂停执行 |
| `/api/issues/:id/resume` | POST | 恢复执行 |
| `/api/issues/:id/stop` | POST | 停止执行 |
| `/api/issues/:id/prompt` | POST | 追加人工提示词 |

**direct 类型下的 UI 条件渲染**：

| 场景 | 隐藏 | 显示 |
|------|------|------|
| 人+人（direct, 无 Agent） | @Agent 选择器、/backlog 按钮、Agent 成员头像、成员管理面板 | 纯文本输入框、/new 按钮 |
| 人+Agent（direct, 有 Agent） | @Agent 选择器（只有 1 个 Agent，无需 @）、成员管理面板 | /backlog 按钮、/new 按钮、Agent 头像标识、Agent 回复消息 |
| 群聊（group） | — | 全部功能 |
