# CI/CD 与构建部署

## 三、GitHub Flow 与 CI/CD

### 3.1 GitHub Flow 分支策略

AnserFlow 采用 GitHub Flow，保持主干可部署、分支短生命周期：

```
main ─────────────────────────●──────────────────●────  (始终可部署)
      \                      /                  /
       feature/xxx ──●──●──●       fix/yyy ──●
```

| 规则 | 说明 |
|------|------|
| `main` 保护 | 禁止直接 push，必须通过 PR 合并 |
| 功能分支 | `feature/<描述>` / `fix/<描述>` / `docs/<描述>` |
| PR 要求 | 至少 1 人 Review + CI 全绿 |
| Commit 规范 | [Conventional Commits](https://www.conventionalcommits.org/zh-hans/)：`feat:` / `fix:` / `docs:` / `refactor:` / `ci:` |
| 发布标签 | `vX.Y.Z` 触发 CD 构建与发布 |
| 合并方式 | Squash & Merge（保持 main 线性历史） |

```bash
# 分支命名示例
git checkout -b feature/agent-orchestration
git checkout -b fix/issue-status-sync
git checkout -b docs/api-examples

# Commit 示例
feat: Agent 编排支持并行执行
docs: 补充 Docker 沙箱架构文档
fix: 修复 Issue 状态同步竞态条件
ci: 添加 Next.js lint 检查 workflow
```

### 3.2 GitHub Actions 工作流总览

```
┌──────────────────────────────────────────────────────────────┐
│  PR → main                                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  ci.yml (每次 push PR)                                │    │
│  │  ├── Go lint + test + build                          │    │
│  │  ├── Next.js lint + type-check + build (admin)       │    │
│  │  └── Next.js lint + type-check + build (desktop)     │    │
│  └──────────────────────────────────────────────────────┘    │
│                           ↓ 合并                              │
├──────────────────────────────────────────────────────────────┤
│  main → 发布                                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  sandbox-image.yml (push main / Dockerfile 变更)      │    │
│  │  ├── Build sandbox Docker image                      │    │
│  │  ├── Push to ghcr.io/anserflow/sandbox               │    │
│  │  └── Tag: latest + commit-sha                        │    │
│  ├──────────────────────────────────────────────────────┤    │
│  │  go-release.yml (push tag v*)                        │    │
│  │  ├── Cross-compile Go backend                        │    │
│  │  ├── Upload anserflow binary (linux/windows/macos)   │    │
│  │  └── Create GitHub Release                           │    │
│  ├──────────────────────────────────────────────────────┤    │
│  │  desktop-release.yml (push tag desktop-v*)            │    │
│  │  ├── Cross-compile Tauri desktop                     │    │
│  │  ├── Sign + Notarize                                 │    │
│  │  ├── Create GitHub Release                           │    │
│  │  └── Generate latest.json (updater)                  │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

| 工作流 | 触发条件 | 耗时 | 产物 |
|--------|---------|------|------|
| `ci.yml` | PR / push main | ~3min | 无（仅检查） |
| `sandbox-image.yml` | push main (Dockerfile) | ~5min | `ghcr.io/anserflow/sandbox` |
| `go-release.yml` | tag `v*` | ~8min | 多平台二进制 + Release |
| `desktop-release.yml` | tag `desktop-v*` | ~20min | Tauri 安装包 + `latest.json` |

### 3.3 ci.yml — Pull Request 检查

Go 后端 + 两套 Next.js 在一份工作流中并行检查：

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # ── Go 后端 ──
  go:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'
          cache: true

      - name: Lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
          args: --timeout=3m

      - name: Test
        run: go test -race -coverprofile=coverage.out ./...

      - name: Build
        run: go build -o /dev/null ./...

  # ── Next.js admin ──
  admin:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: admin
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run build

  # ── Next.js desktop ──
  desktop:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: desktop
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run build
```

> 三个 job 并行运行，总耗时取最慢者（通常 admin build ~2min）。Go test 启用 `-race` 检测数据竞态。

### 3.5 go-release.yml — Go 后端发布

推 tag `v*` 时触发，交叉编译三平台二进制并发布 Release：

```yaml
# .github/workflows/go-release.yml
name: Go Release
on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - goos: linux
            goarch: amd64
          - goos: linux
            goarch: arm64
          - goos: windows
            goarch: amd64
            ext: .exe
          - goos: darwin
            goarch: amd64
          - goos: darwin
            goarch: arm64

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }

      - name: Build admin SPA
        run: |
          npm ci
          npm run build -w @anserflow/admin

      - uses: actions/setup-go@v5
        with: { go-version: '1.24' }

      - name: Build
        env:
          GOOS: ${{ matrix.goos }}
          GOARCH: ${{ matrix.goarch }}
          CGO_ENABLED: 0
        run: |
          go build -ldflags="-s -w -X main.Version=${GITHUB_REF_NAME}" \
            -o anserflow${{ matrix.ext }} .

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: anserflow-${{ matrix.goos }}-${{ matrix.goarch }}
          path: anserflow${{ matrix.ext }}

  release:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v4
      - name: Generate checksums
        run: |
          find . -type f -name 'anserflow*' -exec sha256sum {} \; > checksums.txt
      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          name: 'AnserFlow ${{ github.ref_name }}'
          body: 'See [CHANGELOG.md](./CHANGELOG.md)'
          files: |
            */anserflow*
            checksums.txt
          generate_release_notes: true
```

> `CGO_ENABLED=0` 编译纯静态二进制，无需 glibc 依赖。`-ldflags="-s -w"` 减小体积。

## 四、构建与部署

### 4.1 单一可执行文件

整个后端编译为一个独立二进制文件，后台管理 SPA（`admin/dist`）通过 Go `embed` 嵌入，零运行时依赖（除配置文件外）。

```
┌─────────────────────────────────────────┐
│              anserflow.exe               │
│  ┌───────────────────────────────────┐  │
│  │  Go 运行时 (Gin + GORM + ...)     │  │
│  │                                   │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │ embed.FS (//go:embed admin/dist) │  │  │
│  │  │ ┌─────────────────────────┐ │  │  │
│  │  │ │ admin SPA 构建产物       │ │  │  │
│  │  │ │ index.html + JS + CSS   │ │  │  │
│  │  │ └─────────────────────────┘ │  │  │
│  │  └─────────────────────────────┘  │  │
│  │                                   │  │
│  │  Gin 路由:                        │  │
│  │  /api/*    → 后端 API             │  │
│  │  /ws       → WebSocket            │  │
│  │  /admin/*  → admin SPA            │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**关键实现**：

```go
//go:embed admin/dist
var adminFiles embed.FS

func main() {
    r := gin.Default()

    // API 路由
    api := r.Group("/api")
    // ... 注册 API

    // WebSocket
    r.GET("/ws", handleWebSocket)

    // admin SPA 静态文件（嵌入）
    adminFS, _ := fs.Sub(adminFiles, "admin/dist")
    r.StaticFS("/admin", http.FS(adminFS))
    // 对于 admin SPA 路由，所有 /admin 下的非静态路径返回 index.html
    r.NoRoute(func(c *gin.Context) {
        if strings.HasPrefix(c.Request.URL.Path, "/admin") {
            data, _ := adminFiles.ReadFile("admin/dist/index.html")
            c.Data(http.StatusOK, "text/html", data)
            return
        }
        c.Status(http.StatusNotFound)
    })

    r.Run(":8080")
}
```

### 4.2 交叉编译

支持三平台编译，产物无外部依赖：

```bash
# Windows
GOOS=windows GOARCH=amd64 go build -o anserflow.exe

# macOS (Intel + Apple Silicon)
GOOS=darwin  GOARCH=amd64 go build -o anserflow-darwin-amd64
GOOS=darwin  GOARCH=arm64 go build -o anserflow-darwin-arm64

# Linux
GOOS=linux   GOARCH=amd64 go build -o anserflow-linux-amd64
```

### 4.3 CLI 命令

使用 `github.com/spf13/cobra`：

```bash
# 初始化配置和数据目录
anserflow init
  --db      MySQL 连接串（默认 localhost:3306）
  --redis   Redis 地址（默认 localhost:6379）
  --data    数据目录（默认 ./data）
  --output  配置文件输出路径（默认 ./config.yaml）
  # 自动生成 config.yaml + 数据库表结构

# 启动服务
anserflow server
  --config  config.yaml 路径（默认 ./config.yaml）
  --port    监听端口（默认 8080）
  # 启动 Gin HTTP 服务 + Asynq Worker

# 查看版本
anserflow version

# 仅启动 Worker（分布式部署时）
anserflow worker
  --config  config.yaml 路径
  # 仅启动 Asynq Worker，不启动 HTTP 服务

# 数据库迁移
anserflow migrate
  --config  config.yaml 路径
  --dry-run 仅打印 SQL 不执行（默认 false）
  --backup  迁移前自动生成备份 SQL（默认 true）
  --seed    同时执行种子数据 SQL（Casbin 策略 / 系统默认 Skill 等，默认 true）
  # ① GORM AutoMigrate 自动同步所有表结构
  # ② 执行 internal/seed/*.sql 中的种子数据（Casbin 策略、系统默认 Skill 等）

# 版本升级（下载最新版本并替换当前二进制）
anserflow upgrade
  --channel  更新通道（stable/beta，默认 stable）
  --dry-run  仅检查新版本不执行升级（默认 false）
  # 自动检测最新版本 → 下载 → 校验 → 替换 → 重启
```

```go
// cmd/root.go
var rootCmd = &cobra.Command{
    Use:   "anserflow",
    Short: "AnserFlow - 多智能体协作项目管理系统",
}

// cmd/server.go
var serverCmd = &cobra.Command{
    Use:   "server",
    Short: "启动 AnserFlow 服务",
    Run: func(cmd *cobra.Command, args []string) {
        startServer(cfg)
    },
}

// cmd/init.go
var initCmd = &cobra.Command{
    Use:   "init",
    Short: "初始化配置和数据目录",
    Run: func(cmd *cobra.Command, args []string) {
        initConfig(cfg)
    },
}

// cmd/worker.go
var workerCmd = &cobra.Command{
    Use:   "worker",
    Short: "仅启动 Asynq Worker（分布式部署）",
    Run: func(cmd *cobra.Command, args []string) {
        startWorker(cfg)
    },
}

// cmd/migrate.go
var migrateCmd = &cobra.Command{
    Use:   "migrate",
    Short: "数据库自动迁移（GORM AutoMigrate）",
    Run: func(cmd *cobra.Command, args []string) {
        dryRun, _ := cmd.Flags().GetBool("dry-run")
        backup, _ := cmd.Flags().GetBool("backup")
        runMigrate(cfg, dryRun, backup)
    },
}

// cmd/upgrade.go
var upgradeCmd = &cobra.Command{
    Use:   "upgrade",
    Short: "下载并安装最新版本",
    Run: func(cmd *cobra.Command, args []string) {
        channel, _ := cmd.Flags().GetString("channel")
        dryRun, _ := cmd.Flags().GetBool("dry-run")
        runUpgrade(channel, dryRun)
    },
}
```

`anserflow upgrade` 完整实现流程：

```go
// cmd/upgrade.go
func runUpgrade(channel string, dryRun bool) {
    // ① 获取当前版本
    currentVer := version.Version // -ldflags="-X main.Version=v1.0.0" 注入

    // ② 从 GitHub Releases 获取最新版本
    releaseURL := fmt.Sprintf(
        "https://github.com/anserflow/anserflow/releases/%s/latest", channel)
    latest, err := fetchLatestRelease(releaseURL)
    if err != nil {
        log.Fatal("获取最新版本失败:", err)
    }

    if latest.Version == currentVer {
        fmt.Println("已是最新版本")
        return
    }

    if dryRun {
        fmt.Printf("新版本可用: %s (当前: %s)\n", latest.Version, currentVer)
        return
    }

    // ③ 下载对应平台二进制
    assetName := fmt.Sprintf("anserflow-%s-%s%s",
        runtime.GOOS, runtime.GOARCH, ext())
    assetURL := findAssetURL(latest, assetName)
    checksumURL := findAssetURL(latest, "checksums.txt")

    tmpDir, _ := os.MkdirTemp("", "anserflow-upgrade")
    defer os.RemoveAll(tmpDir)

    downloadFile(assetURL, filepath.Join(tmpDir, assetName))
    downloadFile(checksumURL, filepath.Join(tmpDir, "checksums.txt"))

    // ④ SHA256 校验
    verifyChecksum(tmpDir, assetName)

    // ⑤ 备份当前二进制
    execPath, _ := os.Executable()
    os.Rename(execPath, execPath+".old")

    // ⑥ 替换二进制
    copyFile(filepath.Join(tmpDir, assetName), execPath)
    os.Chmod(execPath, 0755)

    fmt.Printf("升级完成: %s → %s\n", currentVer, latest.Version)
    fmt.Println("请重启服务: anserflow server")
}

// 回滚: mv anserflow.old anserflow && anserflow server
```

> **安全考量**：二进制下载后必须校验 SHA256 签名（与 `go-release.yml` 产物配套）。旧版本保留为 `anserflow.old`，手动回滚。升级本身不自动重启服务，需运维确认后执行。

### 4.4 Next.js SPA 模式

后台管理前端（`admin/`）使用静态导出模式，不依赖 Node.js 服务端：

```js
// admin/next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  basePath: '/admin',      // 后台统一挂载到 /admin
  output: 'export',        // 关键：SPA 静态导出
  distDir: 'dist',         // 输出目录
  images: {
    unoptimized: true,     // export 模式不支持 image optimization
  },
  // API 请求代理到 Go 后端（仅 dev 模式需要）
  async rewrites() {
    return [
      { source: '/api/:path*', destination: 'http://localhost:8080/api/:path*' },
    ]
  },
}
```

**构建流程**：

```bash
# 1. 构建 admin SPA（后台管理）
npm run build -w @anserflow/admin    # → admin/dist/

# 2. Go embed 嵌入 admin 产物
go build -o anserflow                 # admin/dist/ 被 go:embed 打入二进制

# 3. 部署：单个文件即可
./anserflow init    # 首次运行初始化
./anserflow server  # 启动，访问 http://localhost:8080/admin/dashboard
```

### 4.5 项目目录结构（Monorepo）

Gin 后端、两套 Next.js 前端（后台管理 + 桌面客户端）、Tauri 放在同一个仓库。

```
anserflow/
├── package.json                # npm workspace 根配置
├── package-lock.json
├── cmd/                        # Go CLI 入口（Cobra）
│   ├── root.go                 #   根命令注册
│   ├── server.go               #   anserflow server
│   ├── worker.go               #   anserflow worker
│   ├── init.go                 #   anserflow init
│   ├── migrate.go              #   anserflow migrate
│   └── upgrade.go              #   anserflow upgrade
├── internal/                   # Go 业务逻辑
│   ├── handler/                #   Gin Handler（API 路由）
│   ├── service/                #   业务服务层
│   ├── model/                  #   GORM Model
│   ├── middleware/             #   Gin 中间件（JWT / CORS / Casbin）
│   ├── ws/                     #   WebSocket Hub
│   ├── agent/                  #   Agent 编排（Eino 封装）
│   ├── sandbox/                #   Docker 沙箱
│   └── invite/                 #   邀请服务
├── config/                     # Go 配置加载（Viper）
├── admin/                      # ① npm workspace: @anserflow/admin
│   ├── package.json            #   "name": "@anserflow/admin"
│   ├── next.config.js          #   output: "export"
│   ├── tsconfig.json
│   ├── src/                    #   Next.js 源码
│   │   ├── app/                #     /login /dashboard /agents /projects ...
│   │   ├── components/         #     共享 UI 组件
│   │   ├── features/           #     业务模块
│   │   ├── hooks/              #     自定义 Hook
│   │   ├── stores/             #     Zustand Store
│   │   ├── lib/                #     工具函数 & API Client
│   │   └── types/              #     TypeScript 类型
│   └── dist/                   #   构建产物 → //go:embed admin/dist/*
├── desktop/                    # ② npm workspace: @anserflow/desktop
│   ├── package.json            #   "name": "@anserflow/desktop"
│   ├── next.config.js          #   output: "export"
│   ├── tsconfig.json
│   ├── src/                    #   Next.js 源码（客户端视角）
│   │   ├── app/                #     /dashboard /projects/:id /chat ...
│   │   ├── components/
│   │   ├── features/
│   │   ├── hooks/
│   │   ├── stores/
│   │   ├── lib/
│   │   │   └── tauri.ts        #     isTauri() 环境检测
│   │   └── types/
│   ├── dist/                   #   构建产物 → Tauri frontendDist
│   └── src-tauri/              #   Tauri Rust 项目
│       ├── Cargo.toml
│       ├── tauri.conf.json     #     frontendDist: "../dist"
│       ├── capabilities/
│       │   └── default.json
│       ├── icons/
│       └── src/
│           ├── main.rs         #     桌面入口
│           ├── lib.rs          #     核心 + 移动端入口
│           └── commands.rs     #     IPC 命令
├── packages/                   # ③ npm workspace: packages/*
│   └── shared-ui/              #   @anserflow/shared-ui
│       ├── package.json        #     公共组件 / 类型 / lib
│       ├── src/
│       │   ├── components/     #     公共 UI 组件
│       │   ├── lib/            #     公共工具函数
│       │   └── types/          #     公共 TypeScript 类型
│       └── tsconfig.json
├── embed.go                    # //go:embed admin/dist/*
├── main.go                     # Go 入口
├── go.mod
├── go.sum
├── config.yaml                 # 运行配置
└── Makefile                    # 构建脚本
```

**npm workspace 配置**：

```json
// 根 package.json
{
  "name": "anserflow",
  "private": true,
  "workspaces": ["admin", "desktop", "packages/*"]
}
```

```json
// admin/package.json
{ "name": "@anserflow/admin", "dependencies": { "@anserflow/shared-ui": "*" } }

// desktop/package.json
{ "name": "@anserflow/desktop", "dependencies": { "@anserflow/shared-ui": "*" } }

// packages/shared-ui/package.json
{ "name": "@anserflow/shared-ui", "main": "./src/index.ts" }
```

> 所有前端依赖统一提升到根 `node_modules/`，React / Next.js / shadcn/ui 只安装一份。

**三种产物、两个前端入口**：

| 产物 | 前端 | 用户 | 部署方式 |
|------|------|------|----------|
| `anserflow` 二进制 | `admin/` 嵌入 | 管理员/团队负责人（浏览器） | 服务器部署 |
| Tauri 安装包 | `desktop/` 打包 | 普通成员/被邀请者（桌面） | MSI/DMG/AppImage |
| 移动端 | `desktop/` + Tauri mobile | 移动用户 | APK/IPA |

**`admin/` vs `desktop/` 职责划分**：

| | `admin/` 后台管理 | `desktop/` 桌面客户端 |
|------|------|------|
| 访问方式 | 浏览器 `http://host:8080/admin/dashboard` | Tauri 原生窗口 |
| 嵌入 Go | ✅ `//go:embed admin/dist/*` | ❌ 由 Tauri 加载 |
| 目标用户 | 管理员、组织负责人 | 普通成员、被邀请者 |
| 核心页面 | Agent管理 / Skills管理 / 项目创建 / 组织设置 / 系统配置 | Issue 状态Tab / 群聊 / 个人工作台 |
| 路由前缀 | `/admin/*` | `/dashboard` `/projects/:id` `/chat` `/invite/:token` |
| API 地址 | 同源 `/api/*`（无跨域） | 配置的远程 `https://<server>/api/*`；开发环境可指向 `http://localhost:8080/api/*` |

**Admin 端组织上下文**：admin 前端路由为扁平结构（`/admin/agents`），但 API 需要 `org_id`。admin 通过 Sidebar 顶部组织选择器确定当前组织，以 Zustand store 传递：

```tsx
// admin/src/stores/org-store.ts
export const useCurrentOrg = create<{ orgId: string | null; setOrgId: (id: string) => void }>(
  (set) => ({
    orgId: null,
    setOrgId: (orgId) => set({ orgId }),
  })
)

// admin/src/lib/api.ts — 自动注入 org_id
function apiFetch(path: string, init?: RequestInit) {
  const orgId = useCurrentOrg.getState().orgId
  if (!orgId) throw new Error('未选择组织')
  const url = path.includes('/api/orgs/')
    ? path
    : path.replace('/api/', `/api/orgs/${orgId}/`)
  return fetch(url, init)
}
```

```tsx
// admin/src/components/layout/Sidebar.tsx — 组织选择器
<Select value={orgId} onValueChange={setOrgId}>
  {organizations?.map(o => <SelectItem key={o.id} value={o.id}>{o.name}</SelectItem>)}
</Select>
```

**开发运行**：

```bash
# 首次安装（根目录执行一次，所有 workspace 共享 node_modules）
npm install

# ====== 后台管理开发 ======
终端1: go run main.go server                        # Gin :8080
终端2: npm run dev -w @anserflow/admin              # Next.js :3000
#     浏览器打开 http://localhost:3000

# ====== 桌面端开发 ======
终端1: go run main.go server                        # Gin :8080
终端2: npm run dev -w @anserflow/desktop            # Next.js :3001
终端3: npm run tauri -w @anserflow/desktop          # Tauri 窗口加载 :3001
```

**构建流程**：

```bash
# ====== 纯 Web 部署 ======
npm run build -w @anserflow/admin    # → admin/dist/
go build -o anserflow                 # 嵌入 admin/dist/
./anserflow server                    # 浏览器访问 :8080

# ====== 桌面端打包 ======
npm run build -w @anserflow/desktop   # → desktop/dist/
npm run tauri build -w @anserflow/desktop  # → MSI/DMG/AppImage
```

**当前实现决策**：Go 后端本阶段只嵌入 `admin/dist`。`desktop/dist` 由 Tauri 桌面应用自行加载，不在 Go 进程内同时嵌入两套 SPA，也不做路径前缀分发。

```go
//go:embed admin/dist
var adminFiles embed.FS
```

> 💡 后期可从 `admin/` 和 `desktop/` 中提取公共 UI 组件/类型/API Client 到 `packages/shared-ui/`，减少重复代码。

---
