# 沙箱镜像与执行方案

### 3.4 sandbox-image.yml — Docker 沙箱镜像

仅在 `docker/sandbox/Dockerfile` 或相关文件变更时构建，避免浪费 CI 时间：

```yaml
# .github/workflows/sandbox-image.yml
name: Sandbox Image
on:
  push:
    branches: [main]
    paths:
      - 'docker/sandbox/**'
      - '.github/workflows/sandbox-image.yml'
  workflow_dispatch:  # 允许手动触发

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          file: docker/sandbox/Dockerfile
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/sandbox:latest
            ghcr.io/${{ github.repository }}/sandbox:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Image size report
        run: |
          echo "## Sandbox Image Size" >> $GITHUB_STEP_SUMMARY
          docker pull ghcr.io/${{ github.repository }}/sandbox:latest
          docker images ghcr.io/${{ github.repository }}/sandbox:latest --format '{{.Size}}' >> $GITHUB_STEP_SUMMARY
```

> 使用 GitHub Actions cache 加速构建，仅 Dockerfile 变更时才重新构建。

## 七、Docker 沙箱方案

### 7.1 执行流程

```
┌──────────────────────────────────────┐
│  Asynq Worker                        │
│  ┌────────────────────────────────┐  │
│  │  Step 1: 准备                  │  │
│  │  ├── 读取 Issue 上下文         │  │
│  │  ├── 加载 Agent 配置           │  │
│  │  └── 加载绑定的 Skills         │  │
│  │                                │  │
│  │  Step 2: 创建沙箱容器          │  │
│  │  ├── Image: anserflow/sandbox  │  │
│  │  ├── Memory: 512MB             │  │
│  │  ├── CPU: 2 cores              │  │
│  │  ├── Volume: /workspace        │  │
│  │  ├── Mount: .qoder/skills→容器  │  │
│  │  │   (opencode 可直接使用 Skill)│  │
│  │  └── AutoRemove: true          │  │
│  │                                │  │
│  │  Step 3: 注入 opencode 配置     │  │
│  │  ├── 读取 Agent runtime_config  │  │
│  │  ├── 解密 API Key → 环境变量    │  │
│  │  ├── 写入 config.json 到容器    │  │
│  │  ├── 写入 Runtime 默认 Skills   │  │
│  │  │   (opencode→flowcode-executor)│  │
│  │  └── 写入 Agent 绑定 Skills     │  │
│  │                                │  │
│  │  Step 4: 执行（编码闭环 + 消息互通）   │  │
│  │  ├── git clone 关联仓库               │  │
│  │  │                                      │  │
│  │  │  ┌─ opencode run ←→ Worker 消息环 ─┐│  │
│  │  │  │                                  ││  │
│  │  │  │  沙箱内 (opencode)              ││  │
│  │  │  │  ┌────────────────────┐         ││  │
│  │  │  │  │ ① 读取 TASK_PROMPT │         ││  │
│  │  │  │  │ ② LLM 生成代码     │ stdout  ││  │
│  │  │  │  │ ③ write/login.tsx  │────────→││  │
│  │  │  │  │ ④ run test/lint   │         ││  │
│  │  │  │  │ ⑤ 失败→LLM修正    │         ││  │
│  │  │  │  │ ⑥ 成功→exit 0    │         ││  │
│  │  │  │  └────────────────────┘         ││  │
│  │  │  │                                  ││  │
│  │  │  │  Worker (宿主机)                 ││  │
│  │  │  │  ┌────────────────────────────┐  ││  │
│  │  │  │  │ Docker SDK 实时捕获 stdout  │  ││  │
│  │  │  │  │ ↓                            │  ││  │
│  │  │  │  │ 解析输出 → 写入 DB + 推送 WS │  ││  │
│  │  │  │  │                              │  ││  │
│  │  │  │  │ agent_logs:                  │  ││  │
│  │  │  │  │  [12:02] agent 生成 login.tsx│  ││  │
│  │  │  │  │  [12:05] agent 运行 test     │  ││  │
│  │  │  │  │                              │  ││  │
│  │  │  │  │ issue_timeline:              │  ││  │
│  │  │  │  │  [12:02] event=agent_log     │  ││  │
│  │  │  │  │    "正在生成 src/login.tsx"  │  ││  │
│  │  │  │  │                              │  ││  │
│  │  │  │  │ WebSocket → 前端实时更新     │  ││  │
│  │  │  │  └────────────────────────────┘  ││  │
│  │  │  └──────────────────────────────────┘│  │
│  │  │                                      │  │
│  │  └── 退出码 + 工作区检查 → 判定通过/失败 │  │
│  │                                │  │
│  │  Step 5: 收尾                  │  │
│  │  ├── opencode 检查通过:          │  │
│  │  │   ├── git add → commit → push│  │
│  │  │   ├── 创建 PR               │  │
│  │  │   ├── 更新 Issue → in_review│  │
│  │  │   └── 销毁容器（不再需要）   │  │
│  │  ├── opencode 检查失败:          │  │
│  │  │   ├── 写入失败原因到时间线    │  │
│  │  │   ├── Issue → todo（保留沙箱）│  │
│  │  │   └── 等待人工提示词 → 重试  │  │
│  │  └── PR merge → done:            │  │
│  │      └── 销毁容器（如果还存在）   │  │
│  └────────────────────────────────┘  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### 7.1.1 沙箱 ↔ 系统消息互通闭环

opencode 在 Docker 沙箱内通过 **stdout 流 + 退出码** 与 Worker 通信，Worker 负责解析、存储、推送：

```
沙箱内 opencode                    宿主机 Worker                   前端 Issue 时间线
═══════════════                    ════════════                   ════════════════
                                    ┌─ 启动容器，attach stdout ─┐
                                    │                           │
"正在分析 Issue 描述..."  ──stdout──→ 解析 → agent_logs          │
                                    │         issue_timeline     │
                                    │         WS push ──────────→ "Agent 开始分析需求"
                                    │                           │
"生成文件: src/login.tsx" ──stdout──→ 解析 → agent_logs          │
                                    │         WS push ──────────→ "正在生成 src/login.tsx"
                                    │                           │
"运行测试: 3/4 passed"    ──stdout──→ 解析 → agent_logs          │
"FAIL: login.test.tsx"             │         WS push ──────────→ "测试 3/4 通过，1 个失败"
                                    │                           │
"正在修复 login.test.tsx" ─stdout──→ 解析 → agent_logs          │
                                    │         WS push ──────────→ "正在修复测试"
                                    │                           │
"测试全部通过"            ──stdout──→ 解析 → agent_logs          │
                                    │         WS push ──────────→ "测试全部通过"
                                    │                           │
exit 0                              │ 检查: 退出码=0             │
                                    │       workspace 有变更     │
                                    │                           │
                                    │ git commit + push + PR    │
                                    │ issue → in_review ────────→ "PR 已提交，等待审核"
                                    └───────────────────────────┘
```

**Worker 侧 stdout 流式捕获**：

```go
// internal/sandbox/stream.go
func (w *Worker) streamLogs(ctx context.Context, containerID string, issueID uint) {
    reader, _ := w.cli.ContainerLogs(ctx, containerID, container.LogsOptions{
        ShowStdout: true,
        ShowStderr: true,
        Follow:     true,   // 持续跟踪
    })
    defer reader.Close()

    scanner := bufio.NewScanner(reader)
    for scanner.Scan() {
        line := scanner.Text()

        // ① 写 agent_logs（结构化存储）
        w.logRepo.Create(ctx, &model.AgentLog{
            IssueID: issueID,
            Action:  classifyAction(line),    // 根据关键词分类: generate / test / fix / error
            Status:  "running",
            Output:  json.RawMessage(`{"raw":"` + line + `"}`),
        })

        // ② 写 issue_timeline（供前端展示）
        w.timelineRepo.Append(ctx, issueID, "agent", "agent_log", "", "", line)

        // ③ 实时推送到 WebSocket → 前端 Issue 时间线组件实时更新
        w.ws.SendToIssue(issueID, map[string]interface{}{
            "type": "agent_log",
            "text": line,
            "ts":   time.Now().Unix(),
        })
    }

    // opencode 退出后检查退出码
    inspect, _ := w.cli.ContainerInspect(ctx, containerID)
    exitCode := inspect.State.ExitCode
    w.handleCompletion(ctx, issueID, containerID, exitCode)
}
```

**前端实时展现**：

Issue 详情页的时间线面板通过 WebSocket 订阅该 Issue 的 `agent_log` 事件，消息到达后即时追加到时间线末尾，无需轮询或刷新页面。

| 通信方向 | 通道 | 内容 | 频率 |
|---------|------|------|------|
| opencode → Worker | Docker stdout | 实时日志行（生成/测试/修复） | 每秒数次 |
| Worker → MySQL | GORM Insert | `agent_logs`（结构存储）+ `issue_timeline`（展示存储） | 每条 stdout |
| Worker → Redis | ZADD | 最近 N 条日志缓存（可选，加速前端加载历史） | 每条 |
| Worker → 前端 | WebSocket | JSON 事件 `{type:"agent_log", text, ts}` | 每条 stdout |
| opencode → Worker | Exit Code | 0=成功, 非0=失败 | 结束时 1 次 |

### 7.1.2 执行控制（暂停 / 恢复 / 停止）

Worker 监听 Issue 控制命令，通过 Docker API 直接操作沙箱容器：

```go
// internal/sandbox/control.go
func (w *Worker) PauseIssue(issueID uint) error {
    issue, _ := w.issueRepo.FindByID(issueID)
    if issue.Status != "in_progress" {
        return errors.New("只能暂停执行中的 Issue")
    }

    // Docker 冻结容器（进程挂起，内存保留，恢复时从断点继续）
    w.cli.ContainerPause(ctx, issue.SandboxContainerID)

    // 状态流转 + 时间线
    w.issueRepo.UpdateStatus(issueID, "paused")
    w.timelineRepo.Append(issueID, "system", "status_change",
        "in_progress", "paused", "执行已暂停")

    w.ws.SendToIssue(issueID, map[string]interface{}{
        "type": "status_change",
        "status": "paused",
        "hint": "执行已暂停，沙箱冻结中。点击恢复继续。",
    })
    return nil
}

func (w *Worker) ResumeIssue(issueID uint) error {
    issue, _ := w.issueRepo.FindByID(issueID)
    if issue.Status != "paused" {
        return errors.New("只能恢复已暂停的 Issue")
    }

    // Docker 解冻容器（从断点继续运行）
    w.cli.ContainerUnpause(ctx, issue.SandboxContainerID)

    w.issueRepo.UpdateStatus(issueID, "in_progress")
    w.timelineRepo.Append(issueID, "system", "status_change",
        "paused", "in_progress", "执行已恢复")

    w.ws.SendToIssue(issueID, map[string]interface{}{
        "type": "status_change",
        "status": "in_progress",
        "hint": "沙箱已解冻，opencode 继续执行。",
    })
    return nil
}

func (w *Worker) StopIssue(issueID uint) error {
    issue, _ := w.issueRepo.FindByID(issueID)
    if issue.Status != "in_progress" && issue.Status != "paused" {
        return errors.New("只能停止执行中或已暂停的 Issue")
    }

    // Docker 终止容器 + 销毁
    timeout := 10
    w.cli.ContainerStop(ctx, issue.SandboxContainerID, container.StopOptions{Timeout: &timeout})
    w.cli.ContainerRemove(ctx, issue.SandboxContainerID, container.RemoveOptions{Force: true})

    // 回到 backlog，可重新编辑后再执行
    w.issueRepo.UpdateStatus(issueID, "backlog")
    w.issueRepo.ClearContainerID(issueID)
    w.timelineRepo.Append(issueID, "system", "status_change",
        issue.Status, "backlog", "执行已停止，容器已销毁")

    w.ws.SendToIssue(issueID, map[string]interface{}{
        "type": "status_change",
        "status": "backlog",
        "hint": "执行已停止。可编辑 Issue 后重新转为 todo 启动。",
    })
    return nil
}
```

**前端控制按钮**（仅在 `in_progress` / `paused` 状态下显示）：

```
┌─ Issue #1 (in_progress) ────────────────────────────────────────┐
│  [⏸ 暂停]  [⏹ 停止]                                             │
│                                                                  │
│  时间线:                                                         │
│   12:02  agent   正在生成 src/login.tsx                           │
│   12:03  agent   正在生成 src/auth.ts                             │
│   ───────────────────────────────────────────                    │
│   [追加提示词: ________________________] [发送并重新执行]          │
└──────────────────────────────────────────────────────────────────┘
```

暂停后按钮变为：

```
┌─ Issue #1 (paused) ────────────────────────────────────────────┐
│  [▶ 恢复]  [⏹ 停止]                                             │
│                                                                  │
│  时间线:                                                         │
│   12:02  agent   正在生成 src/login.tsx                           │
│   12:05  system  执行已暂停（沙箱冻结中）                           │
└──────────────────────────────────────────────────────────────────┘
```

| 操作 | Docker API | 沙箱状态 | Issue 状态 | 数据保留 |
|------|-----------|---------|-----------|---------|
| **暂停** | `ContainerPause` | 进程冻结，内存保留 | `paused` | ✅ 全部保留 |
| **恢复** | `ContainerUnpause` | 从断点继续 | `in_progress` | ✅ 继续 |
| **停止** | `ContainerStop` + `Remove` | 容器销毁 | `backlog` | ❌ 沙箱清空，MySQL 数据保留 |

#### 暂停/恢复与 Asynq 任务生命周期

暂停/恢复操作直接操作 Docker 容器，但 Asynq 中的任务**已经出队**（dequeue），不再受 Redis 队列管理。为保证状态一致性，Worker 采用以下机制：

```go
// internal/sandbox/control.go — 暂停时持久化任务上下文
func (w *Worker) PauseIssue(issueID uint) error {
    // ... ContainerPause ...
    
    // 关键：写入 issue_timeline 记录当前执行进度
    // Worker 会在心跳循环中定期检查 issue.status，若发现变为 paused 则挂起自身
    w.issueRepo.UpdateStatus(issueID, "paused")
    
    // 将当前 opencode 进度写入 agent_logs（标记暂停点）
    w.logRepo.Create(ctx, &model.AgentLog{
        IssueID: issueID,
        Action:  "paused",
        Status:  "running",
        Output:  json.RawMessage(`{"event":"paused","sandbox_id":"` + containerID + `"}`),
    })
    return nil
}
```

**Worker 心跳与状态监听**：Worker 在执行循环中每 10 秒检查一次对应 Issue 的状态，若变为 `paused` 则挂起自身 goroutine，等待 Issue 恢复信号：

```go
// internal/worker/executor.go — Worker 心跳检查
func (w *Worker) executeWithHeartbeat(ctx context.Context, issueID uint, runFn func() error) error {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            issue, _ := w.issueRepo.FindByID(issueID)
            switch issue.Status {
            case "paused":
                // 进入等待循环，直到恢复或停止
                for {
                    time.Sleep(3 * time.Second)
                    current, _ := w.issueRepo.FindByID(issueID)
                    if current.Status == "in_progress" {
                        break // 恢复执行
                    }
                    if current.Status != "paused" {
                        return errors.New("issue was stopped during pause") // 已停止
                    }
                }
            case "backlog", "done":
                return errors.New("issue was stopped externally")
            }
        }
    }
}
```

**Worker 重启后恢复**：Worker 进程重启时，通过以下方式找回暂停的容器：

```
Worker 启动
    │
    ▼
扫描 issues 表 WHERE status='paused' AND sandbox_container_id IS NOT NULL
    │
    ├── 存在 → Docker API Inspect 检查容器是否存活
    │           ├── 存活 → 恢复 Attach stdout 流，继续心跳监听
    │           └── 已销毁 → Issue → backlog + 记录异常时间线
    │
    └── 不存在 → 无操作
```

```go
// internal/worker/recovery.go
func (w *Worker) RecoverPausedIssues(ctx context.Context) {
    pausedIssues, _ := w.issueRepo.FindByStatus(ctx, "paused")
    for _, issue := range pausedIssues {
        if issue.SandboxContainerID == "" {
            // 异常：paused 但没有容器 → 回退到 backlog
            w.issueRepo.UpdateStatus(issue.ID, "backlog")
            w.timelineRepo.Append(issue.ID, "system", "status_change",
                "paused", "backlog", "Worker 重启后未找到沙箱容器，已自动回退")
            continue
        }
        // 尝试恢复容器关联
        _, err := w.cli.ContainerInspect(ctx, issue.SandboxContainerID)
        if err != nil {
            w.issueRepo.UpdateStatus(issue.ID, "backlog")
            w.issueRepo.ClearContainerID(issue.ID)
            w.timelineRepo.Append(issue.ID, "system", "status_change",
                "paused", "backlog", "沙箱容器已不存在，已自动回退")
        }
        // 容器存活 → 重新 Attach 并继续心跳监听
    }
}
```

> **设计要点**：暂停/恢复的可靠性依赖于 `sandbox_container_id` 持久化到 DB。只要容器未被销毁，Worker 重启后总能重新 Attach 并恢复上下文。

### 7.2 安全策略

| 策略 | 配置 |
|------|------|
| 资源限制 | CPU 2核 / 内存 512MB / 磁盘 1GB |
| 执行超时 | 单任务最长 30 分钟，超时强制 SIGKILL |
| 网络隔离 | 仅允许出站到 GitHub API + LLM API 白名单 |
| 凭证注入 | GitHub Token 通过环境变量注入，不落盘 |
| 自动清理 | 执行成功 → 立即销毁容器；执行失败 → 保留容器等待人工介入重试，Issue done 时最终销毁 |
| 非 root 运行 | User `sandbox` (uid 1000)，无 sudo 权限 |
| 镜像最小化 | Alpine 3.21 基础，预装 Node/Python/Git，不含 Go 运行时 |

**网络白名单实现**：

Docker 沙箱通过自定义网络 + 容器内 iptables 规则实现出站域名白名单：

```go
// internal/sandbox/network.go — 创建带网络隔离的容器
func createSandboxNetwork(ctx context.Context, cli *client.Client) (string, error) {
    // 创建自定义 bridge 网络（启用 iptables）
    network, err := cli.NetworkCreate(ctx, "anserflow-sandbox", types.NetworkCreate{
        Driver: "bridge",
        Options: map[string]string{
            "com.docker.network.bridge.enable_ip_masquerade": "true",
        },
    })
    return network.ID, err
}

func createSandboxContainer(ctx context.Context, cli *client.Client, cfg SandboxConfig) (string, error) {
    allowedDomains := cfg.AllowedDomains // 来自 config.yaml sandbox.allowed_domains

    resp, _ := cli.ContainerCreate(ctx, &container.Config{
        Image: cfg.Image,
        Env:   buildEnvVars(cfg),
        // 容器启动后执行 iptables 规则
        Entrypoint: []string{"/entrypoint.sh"},
    }, &container.HostConfig{
        Memory:     cfg.Memory * 1024 * 1024,
        NanoCPUs:   cfg.CPU * 1e9,
        NetworkMode: container.NetworkMode("anserflow-sandbox"),
        // DNS 仅解析白名单域名所需
        DNS: []string{"8.8.8.8"},
        // 禁用宿主机端口映射
        PortBindings: nat.PortMap{},
        CapDrop:      []string{"NET_RAW"},  // 禁止容器内修改 iptables
    }, nil, nil, "")

    // 注入白名单到容器环境变量，entrypoint.sh 据此配置 iptables
    return resp.ID, nil
}
```

**容器内 iptables 脚本**（在 `entrypoint.sh` 开头执行）：

```bash
#!/bin/sh
# entrypoint.sh — 网络白名单配置

# 默认 DROP 所有出站
iptables -P OUTPUT DROP
# 允许 lo 回环
iptables -A OUTPUT -o lo -j ACCEPT
# 允许 DNS 查询（UDP 53）
iptables -A OUTPUT -p udp --dport 53 -j ACCEPT

# 解析白名单域名并添加 iptables 规则
# ALLOWED_DOMAINS 环境变量由 Worker 注入，格式: "github.com,api.github.com,api.openai.com"
for domain in $(echo $ALLOWED_DOMAINS | tr ',' ' '); do
    # 解析域名到 IP，添加 iptables ACCEPT 规则
    for ip in $(dig +short $domain); do
        iptables -A OUTPUT -d $ip -j ACCEPT
    done
    # 也允许 HTTPS (443) 出去（基于域名解析不够稳定时作为补充）
    iptables -A OUTPUT -p tcp --dport 443 -j ACCEPT
done

# 执行后续命令（git clone / opencode run）
exec "$@"
```

> **备选方案**：如果 iptables 方式在某些 CI 环境中不可用（如 Docker-in-Docker 受限），可退化为**无网络过滤**（仅依赖 Docker 默认 bridge 网络隔离 + 容器内不暴露任何端口）。生产环境建议在 K8s NetworkPolicy 层统一管理。

**Dockerfile** 位于 `docker/sandbox/Dockerfile`：

```bash
# 构建沙箱镜像
docker build -t anserflow/sandbox:latest -f docker/sandbox/Dockerfile .
```

> `docker/sandbox/.dockerignore` 已排除 `node_modules/`、`.git/`、`dist/` 等，确保构建上下文最小。

**Dockerfile**（`docker/sandbox/Dockerfile`）：

```dockerfile
FROM alpine:3.21

RUN apk add --no-cache \
    nodejs npm \
    python3 py3-pip \
    git \
    curl \
    bash \
    && rm -rf /var/cache/apk/*

RUN adduser -D -u 1000 sandbox

# opencode — 开源 AI 编码代理（npm 全局安装）
# https://github.com/anomalyco/opencode
RUN npm install -g opencode-ai@latest \
    && npm cache clean --force

RUN mkdir -p /workspace && chown sandbox:sandbox /workspace
WORKDIR /workspace

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

USER sandbox

ENTRYPOINT ["/entrypoint.sh"]
```

```bash
# entrypoint.sh — 沙箱入口脚本
# 由 Worker 传入环境变量和配置注入文件控制行为：
#   GIT_REPO_URL / GIT_BRANCH / GITHUB_TOKEN → git clone
#   TASK_PROMPT                              → opencode run 的 prompt 参数
#
# opencode 配置由 Worker 在容器启动后、执行编码前通过以下方式注入（每次覆盖）：
#   ① 写入 ~/.config/opencode/config.json   → opencode 读取 provider / model 配置
#   ② 注入环境变量                           → API Key（如 OPENAI_API_KEY）不落盘
#   ③ opencode run --model provider/model    → 运行时指定模型
```
```

**opencode 配置注入流程**（Worker 侧伪代码）：

```go
// Worker 从 Agent runtime_config 读取 opencode 配置，写入沙箱
func injectOpenCodeConfig(ctx context.Context, containerID string, agent *model.Agent) error {
    cfg := agent.RuntimeConfig.OpenCode

    // ① 生成 opencode 配置文件（模型、provider、tools 权限等）
    configJSON := OpenCodeConfig{
        Model:       cfg.Model,       // "gpt-4o"
        Provider:    cfg.Provider,    // "openai"
        Agent:       cfg.Agent,       // "build"
        MaxTurns:    cfg.MaxIterations,
        Thinking:    cfg.Thinking,
        Permissions: map[string]bool{  // opencode 工具权限白名单
            "read":  true,
            "write": true,
            "bash":  true,
            "glob":  true,
            "grep":  true,
        },
    }
    // 写入容器内 ~/.config/opencode/config.json（通过 docker cp 或 exec）
    execInContainer(ctx, containerID,
        "mkdir -p /home/sandbox/.config/opencode",
        "echo '"+toJSON(configJSON)+"' > /home/sandbox/.config/opencode/config.json",
    )

    // ② 注入 API Key（环境变量已在容器创建时设置，此处为 Provider → 环境变量映射）
    //    容器创建时的 Env: "OPENAI_API_KEY=" + decrypt(cfg.APIKeyEncrypted)
    return nil
}
```

> **安全说明**：API Key 在数据库中以加密存储，Worker 解密后通过环境变量注入容器，不写入容器文件系统。

```dockerfile
# .dockerignore
node_modules/
.git/
dist/
.next/
*.log
.DS_Store
```

> 预估镜像大小约 400MB（含 Node.js + npm + opencode + Python + Git）。

### 7.3 Go Docker SDK

使用官方 `github.com/docker/docker/client`：

```go
import (
    "github.com/docker/docker/client"
    "github.com/docker/docker/api/types/container"
)

func createSandbox(ctx context.Context, cfg SandboxConfig) (string, error) {
    cli, _ := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())

    if cfg.ExistingContainerID != "" {
        injectPromptAndRun(ctx, cfg.ExistingContainerID, cfg.TaskPrompt, cfg.Runtime)
        return cfg.ExistingContainerID, nil
    }

    // 根据运行时选择镜像
    image := cfg.Runtime.DockerImage       // 来自 runtimes.docker_image

    // 项目根目录（含 .qoder/skills）
    projectRoot := cfg.WorkspaceRoot

    resp, _ := cli.ContainerCreate(ctx, &container.Config{
        Image: image,
        Env:   buildEnvVars(cfg),
    }, &container.HostConfig{
        Memory:     512 * 1024 * 1024,
        NanoCPUs:   2 * 1e9,
        AutoRemove: false,
        // bind mount: 宿主机 .qoder/skills → 容器内 /home/sandbox/.qoder/skills（只读）
        Binds: []string{
            projectRoot + "/.qoder/skills:/home/sandbox/.qoder/skills:ro",
        },
    }, nil, nil, "")

    cli.ContainerStart(ctx, resp.ID, container.StartOptions{})

    // 注入运行时配置 + 执行
    injectRuntimeConfig(ctx, resp.ID, cfg.Runtime)
    // 根据 execute_template 渲染命令: "opencode run \"...\" --model ..."
    cmd := renderTemplate(cfg.Runtime.ExecuteTemplate, cfg.Runtime.Config, cfg.TaskPrompt)
    execInContainer(ctx, resp.ID, cmd)
    return resp.ID, nil
}

// 在 Issue→done 时销毁沙箱
func destroySandbox(ctx context.Context, containerID string) {
    cli.ContainerRemove(ctx, containerID, container.RemoveOptions{Force: true})
}
```

### 7.4 GitHub SDK 集成

Git 操作与 GitHub API 分别使用两套 Go SDK，职责分离：

| SDK | 用途 | 运行位置 | 认证 |
|------|------|----------|------|
| **go-git** (`github.com/go-git/go-git/v5`) | `clone` / `commit` / `push` / `checkout` | Docker 沙箱内（Worker） | HTTP Token / SSH 私钥 |
| **go-github** (`github.com/google/go-github/v68`) | 创建 PR / Issue / Review Comment / 读取仓库信息 | Go 后端（Service 层） | HTTP Personal Access Token |

```
┌─────────────────────────────────────────────────┐
│  go-github（GitHub REST API）                    │
│  ├── PullRequests.Create → 创建 PR              │
│  ├── Issues.Create / Edit → 操作 Issue          │
│  ├── PullRequests.CreateReview → 添加 Review    │
│  └── Repositories.GetContents → 读取文件树      │
│  运行位置：internal/service/（Gin HTTP 层）      │
├─────────────────────────────────────────────────┤
│  go-git（纯 Go Git 实现，无需系统 git 二进制）     │
│  ├── PlainClone → git clone                     │
│  ├── Worktree.Add + Commit → git add / commit   │
│  ├── Push → git push origin                     │
│  └── Checkout → git checkout -b                 │
│  运行位置：Docker 沙箱内（Asynq Worker）          │
└─────────────────────────────────────────────────┘
```

#### go-github 示例

```go
import "github.com/google/go-github/v68/github"

client := github.NewClient(nil).WithAuthToken(githubToken)

// 创建 Issue
issue, _, _ := client.Issues.Create(ctx, owner, repo, &github.IssueRequest{
    Title:  github.Ptr("实现用户登录页面"),
    Body:   github.Ptr("需要 JWT + bcrypt 认证"),
    Labels: &[]string{"frontend", "p1"},
})

// 创建 Pull Request
pr, _, _ := client.PullRequests.Create(ctx, owner, repo, &github.NewPullRequest{
    Title: github.Ptr("feat: 用户登录页面"),
    Head:  github.Ptr("feature/login"),
    Base:  github.Ptr("main"),
    Body:  github.Ptr("Closes #" + issueID),
})
```

#### go-git 认证示例

**HTTP Token 认证**（对应 `github_auth_type = 'http'`）：

```go
import (
    "github.com/go-git/go-git/v5"
    "github.com/go-git/go-git/v5/plumbing/transport/http"
)

repo, _ := git.PlainClone("/workspace/repo", false, &git.CloneOptions{
    URL: "https://github.com/org/repo.git",
    Auth: &http.BasicAuth{
        Username: "x-access-token", // GitHub PAT 认证固定用户名（非真实 GitHub 用户名）
        Password: githubToken,       // ghp_xxxxxxxxxxxx
    },
})
```

**SSH 私钥认证**（对应 `github_auth_type = 'ssh'`）：

```go
import (
    "github.com/go-git/go-git/v5"
    "github.com/go-git/go-git/v5/plumbing/transport/ssh"
)

publicKeys, _ := ssh.NewPublicKeys("git", []byte(privateKey), passphrase)
repo, _ := git.PlainClone("/workspace/repo", false, &git.CloneOptions{
    URL:  "git@github.com:org/repo.git",
    Auth: publicKeys,
})
```

**commit → push**：

```go
w, _ := repo.Worktree()
w.Add(".")
w.Commit("feat: Agent 自动生成的登录页", &git.CommitOptions{
    Author: &object.Signature{Name: agentName, Email: agentEmail, When: time.Now()},
})
repo.Push(&git.PushOptions{Auth: auth})
```

#### GitHub Token 权限要求

| 权限 | 用途 |
|------|------|
| `repo`（全部） | 读写仓库代码 |
| `issues:write` | 创建/更新 Issue |
| `pull_requests:write` | 创建 PR |

> Classic Token 授权全部仓库即可；Fine-grained Token 需额外指定具体仓库。

#### Go 模块依赖

```
# go.mod
github.com/google/go-github/v68 v68.0.0
github.com/go-git/go-git/v5 v5.15.0
```

#### 多平台扩展设计

后期支持 Gitea / GitLab / Gitee 等平台，通过接口抽象实现：

```
┌──────────────────────────────────────────────┐
│  internal/git/                                │
│  ├── provider.go        GitProvider 接口定义  │
│  ├── github.go          GitHub 实现           │
│  ├── gitea.go           Gitea 实现            │
│  └── gitlab.go          GitLab 实现           │
└──────────────────────────────────────────────┘
```

```go
// internal/git/provider.go — 平台无关接口
type GitProvider interface {
    CreateIssue(ctx, repo, title, body string, labels []string) (issueID string, err error)
    CreatePR(ctx, repo, title, head, base, body string) (prURL string, err error)
    GetRepoInfo(ctx, repo string) (*RepoInfo, error)
    ListRepos(ctx) ([]RepoInfo, error)
}
```

`go-git` 层无需任何改动 — 它只做 `clone/commit/push`，协议层（HTTP/SSH）与平台无关。只有 REST API 层需要为每个平台实现 `GitProvider` 接口。

| 平台 | Go SDK | 备注 |
|------|--------|------|
| **GitHub** | `go-github/v68` | 已实现 |
| **Gitea** | `code.gitea.io/sdk/gitea` | API 兼容 GitHub，迁移成本低 |
| **GitLab** | `github.com/xanzy/go-gitlab` | API 结构不同，独立实现 |
| **Gitee** | REST API（无官方 Go SDK） | 可基于 `net/http` 封装 |

> 当前阶段只闭环 GitHub 集成：`git_platform` 字段保留，但本轮验收默认取值为 `github`。Gitea / GitLab / Gitee Provider 进入后续独立任务，不作为当前计划阻塞项。

---
