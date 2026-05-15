# 分布式与 Agent 架构

## 五、分布式架构设计

### 5.1 WebSocket 分布式方案

```
                    ┌─────────────┐
                    │  Redis       │
                    │  Pub/Sub     │
                    └──┬───┬───┬──┘
                       │   │   │
          ┌────────────┘   │   └────────────┐
          ▼                ▼                ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Gin #1   │    │ Gin #2   │    │ Gin #3   │
    │ WS连接池 │    │ WS连接池 │    │ WS连接池 │
    └──────────┘    └──────────┘    └──────────┘
```

**原理**：每个 Gin 实例维护自己的 WebSocket 连接池。消息发送时，先推送给本地连接的客户端，再通过 Redis Pub/Sub 广播到其他实例，各实例转发给自己持有的客户端。

**Go 库**：`github.com/gorilla/websocket` + `github.com/redis/go-redis/v9`

**消息协议**：所有 WebSocket 通信统一采用 JSON 信封格式：

```json
{
  "type": "string",       // 消息类型（见下表）
  "seq": 12345,           // 消息序号（客户端递增，用于去重/重放检测）
  "ts": 1715678900,       // 服务端时间戳（Unix 秒）
  "group_id": 42,         // 群组 ID（群聊消息必填）
  "sender": {
    "type": "user|agent",
    "id": 1,
    "name": "张三",
    "avatar_url": "..."
  },
  "content": {},           // 消息体（字段见下表）
  "error": null            // 错误信息（仅 type:error）
}
```

| type | 说明 | content 关键字段 |
|------|------|-----------------|
| `message` | 普通聊天消息 | `text: string` |
| `annotation` | Agent 分析/技术评审（非对话消息） | `text: string`, `role: "analysis"\|"review"\|"estimate"` |
| `backlog` | 方案产出（`/backlog` 指令触发，每次产出一个 Issue） | `issue: {title, status: "backlog", priority, assignee, description}` |
| `system` | 系统通知（Issue 创建/状态变更） | `text: string`, `resource_type: "issue"\|"agent"`, `resource_id` |
| `ping` | 心跳请求（客户端每 30s 发送） | 无 |
| `pong` | 心跳响应 | 无 |
| `typing` | 正在输入状态 | `is_typing: bool` |
| `backlog_ack` | 方案确认/拒绝 | `backlog_id: string`, `accepted: bool` |
| `error` | 错误响应 | `error.code: string`, `error.message: string` |

**心跳与重连**：

- 客户端每 30s 发送 `ping`，服务端回复 `pong`
- 90s 内未收到任何消息视为断连，服务端主动关闭连接
- 客户端重连采用指数退避：`1s → 2s → 4s → 8s → 16s → 32s (max)`
- 重连后客户端发送 `seq` 字段为上次收到的最后序号，服务端据此推送遗漏消息

**消息持久化**：所有 `message` / `system` / `annotation` / `backlog` 类型消息在发送前先写入 `messages` 表，确保消息不丢失。

**Redis 消息缓存**（断线重连恢复）：

断线重连时客户端通过 `seq` 号请求遗漏消息。为减少 MySQL 查询压力，在 Redis 中维护每个群组的最近消息滑动窗口：

```go
// 消息写入时双写
func (h *Hub) SendMessage(msg *Message) {
    // 1. 写入 MySQL（持久化）
    h.repo.InsertMessage(ctx, msg)

    // 2. 写入 Redis 滑动缓存（ZSET，按 seq 排序）
    key := fmt.Sprintf("msg:cache:%d", msg.GroupID)
    h.redis.ZAdd(ctx, key, redis.Z{
        Score:  float64(msg.Seq),
        Member: msg.JSON(),
    })
    h.redis.Expire(ctx, key, 24*time.Hour) // 续期 TTL
    // 3. 裁剪：仅保留最近 500 条（超出则移除最旧）
    h.redis.ZRemRangeByRank(ctx, key, 0, -501)
}
```

| 参数 | 值 | 理由 |
|------|-----|------|
| 缓存条目数 | 最近 500 条/群组 | 覆盖极端断线（30min × 60msg/min = 1800 条，500 条覆盖正常离线窗口 5-10min） |
| TTL | 24 小时（每次写入续期） | 活跃群组保持缓存，冷群组自动过期释放内存 |
| 数据结构 | Redis ZSET（seq → JSON） | 按 seq 范围查询 O(logN+ M)，比 List 更适合续传场景 |
| 内存估算 | 500 条 × 100 群组 × 2KB ≈ 100MB | 生产环境足够 |

**重连流程**：

```
客户端断线重连
    → 发送最后 seq=1230
    → 服务端从 Redis ZSET 拉取 seq > 1230 的消息（最多 500 条）
    → Redis 命中 → 直接返回（~2ms）
    → Redis 未命中（冷群组或超 500 条）→ 回退 MySQL 查询
```

### 5.2 任务队列方案

选用 **Asynq**（https://github.com/hibiken/asynq），基于 Redis 的分布式任务队列：

```
Issue 状态变为 in_progress (assignee = agent)
        │
        ▼
┌──────────────────┐
│ Asynq Client      │  →  enqueue("agent:execute", payload)
│ (Gin HTTP 层)    │     Priority: P0 > P1 > P2...
└──────────────────┘     Timeout: 30min
        │                MaxRetry: 3
        ▼                Payload: {issue_id, agent_id, human_prompts[]}
┌──────────────────┐
│ Redis Queue       │
│ ├── critical (P0) │
│ ├── default  (P1) │
│ └── low      (P2+)│
└──────────────────┘
        │
        ▼
┌──────────────────┐
│ Asynq Worker      │  →  HandleFunc("agent:execute", handler)
│ (独立进程/协程)   │     1. 创建 Docker 沙箱
└──────────────────┘     2. 注入 opencode 配置 + Agent 人设
        │                3. 注入人工提示词（来自 issue_timeline）
        ▼                4. opencode run 执行编码
┌──────────────────┐     5. opencode 检查结果
│ Docker Sandbox    │     6. 通过 → commit + push + PR → in_review
└──────────────────┘     7. 失败 → 写入时间线 → 人工介入重试
```

Asynq 核心特性：

| 特性 | Agent 执行场景 |
|------|---------------|
| 任务优先级 | P0 Issue 插队执行 |
| 重试机制 | 执行失败自动重试 3 次 |
| 超时控制 | 单任务最长 30 分钟 |
| 死信队列 | 3 次重试仍失败 → 人工介入 |
| 定时任务 | 延迟执行（Agent 启动冷却期） |
| Web UI | Asynqmon 可视化管理面板 |

**Issue 调度器**（todo → in_progress 自动调度）：

系统的调度器作为一个轻量的 Gin 后台协程运行，与 Asynq Worker 解耦：

```go
// internal/scheduler/issue_scheduler.go
func (s *IssueScheduler) Run(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    for {
        select {
        case <-ticker.C:
            // ① 扫描所有 org 的 todo Issue，按优先级 ASC + 创建时间 ASC
            issues := s.repo.FindSchedulableIssues(ctx)
            for _, issue := range issues {
                // ② 检查该 org 是否达到并发上限（默认 3 个 Agent 同时执行）
                if s.runningCount(issue.OrgID) >= s.maxConcurrent(issue.OrgID) {
                    continue
                }
                // ③ 状态 → in_progress + 写入时间线
                s.repo.TransitionStatus(issue.ID, "todo", "in_progress")
                s.timelineRepo.Append(issue.ID, "system", "status_change",
                    "todo", "in_progress", "调度器自动分配执行")
                // ④ 入队 Asynq
                s.asynqClient.Enqueue(issue)
                s.ws.NotifyProject(issue.ProjectID, "Issue #%d 开始执行", issue.ID)
            }
        case <-ctx.Done():
            return
        }
    }
}
```

> 每个组织默认最多 3 个 Agent 同时执行（可通过 org settings 调整），超过上限的 Issue 保持 todo 等待。

并发统计直接查询 `issues` 表，无需额外 Redis 计数器：

```go
func (s *IssueScheduler) runningCount(orgID uint) int {
    var count int64
    s.db.Model(&Issue{}).
        Joins("JOIN projects ON projects.id = issues.project_id").
        Where("projects.org_id = ? AND issues.status = ?", orgID, "in_progress").
        Count(&count)
    return int(count)
}
```

#### 调度器对 paused 状态的处理

调度器扫描可调度 Issue 时**明确跳过 `paused` 状态**，避免重复入队：

```go
// internal/scheduler/issue_scheduler.go — 查询条件
func (r *IssueRepo) FindSchedulableIssues(ctx context.Context) ([]*Issue, error) {
    var issues []*Issue
    return issues, r.db.WithContext(ctx).
        Joins("JOIN projects ON projects.id = issues.project_id").
        Where("issues.status = ?", "todo").           // 仅 todo 状态
        Where("issues.retry_count < 3").               // 重试未耗尽
        Where("issues.assignee_role = ?", "agent").    // 仅 Agent 负责
        Order("FIELD(issues.priority, 'p0','p1','p2','p3','p4') ASC, issues.created_at ASC").
        Find(&issues).Error
}
```

> 调度器只扫 `todo`，不会扫到 `paused`。`paused` 状态的 Issue 由 Worker 心跳循环自行管理，与调度器完全解耦。

#### Asynq 任务状态与 Issue 状态双向同步

| 事件 | Issue 状态 | Asynq 任务状态 | 说明 |
|------|-----------|--------------|------|
| 调度器入队 | `todo → in_progress` | Enqueued | 写入 Redis queue |
| Worker 消费 | `in_progress` | Active（已 dequeue） | 不在 Redis 队列中 |
| 人工暂停 | `in_progress → paused` | Active（Worker 心跳等待） | Docker 容器冻结 |
| 人工恢复 | `paused → in_progress` | Active（Worker 心跳跳出） | Docker 容器解冻 |
| 人工停止 | `in_progress/paused → backlog` | 终止（goroutine 退出） | 容器销毁 |
| opencode 成功 | `in_progress → in_review` | Completed | 任务返回 nil |
| opencode 失败 | `in_progress → todo` | Failed → 重新入队（retry_count+1） | 保留沙箱 |
| 重试耗尽 | `todo → backlog` | Archived（死信队列） | 销毁沙箱 |

#### 调度器无限重试防护

为防止配置错误的 Issue 无限循环，在 `issues` 表增加 `retry_count` 字段：

```sql
-- issues 表新增字段
ALTER TABLE issues ADD COLUMN retry_count INT DEFAULT 0;
-- 重试次数仅在人工确认转为 todo 时重置为 0
```

**调度器过滤**（已在 `FindSchedulableIssues` 中加入 `WHERE retry_count < 3`）。

**执行失败时递增**：

```go
// internal/worker/executor.go — opencode 失败处理
func (w *Worker) handleCompletion(ctx context.Context, issueID uint, containerID string, exitCode int) {
    if exitCode != 0 {
        issue, _ := w.issueRepo.FindByID(issueID)
        newCount := issue.RetryCount + 1

        if newCount >= 3 {
            // 重试耗尽 → 回退 backlog，销毁沙箱，通知人工
            w.issueRepo.UpdateStatus(issueID, "backlog")
            w.issueRepo.UpdateRetryCount(issueID, newCount)
            w.cli.ContainerRemove(ctx, containerID, container.RemoveOptions{Force: true})
            w.issueRepo.ClearContainerID(issueID)
            w.timelineRepo.Append(issueID, "system", "status_change",
                "in_progress", "backlog",
                fmt.Sprintf("重试 %d 次仍失败，已自动回退到 backlog，请人工检查", newCount))
            w.notification.NotifyIssueStatusChanged(ctx, issue, issue.CreatedBy)
            return
        }

        // 仍有重试配额 → 回到 todo，保留沙箱
        w.issueRepo.UpdateStatus(issueID, "todo")
        w.issueRepo.UpdateRetryCount(issueID, newCount)
        w.timelineRepo.Append(issueID, "system", "status_change",
            "in_progress", "todo",
            fmt.Sprintf("第 %d 次执行失败，等待人工提示词后重试", newCount))
    }
}
```

**人工确认重置**（仅当用户手动点击 [转为 todo] 时重置）：

```go
// internal/service/issue_service.go
func (s *IssueService) TransitionToTodo(ctx context.Context, issueID uint) error {
    return s.repo.Update(ctx, issueID, map[string]interface{}{
        "status":      "todo",
        "retry_count": 0,                        // 人工确认后重置
    })
}
```

> **防护效果**：同一 Issue 最多经历 3 次自动重试循环（`todo → in_progress → 失败 → todo`），第 4 次自动回退 `backlog` 并通知人工介入。仅人工确认后重置 `retry_count`。

### 5.3 整体分布式拓扑

```
                    ┌──────────────┐
                    │  MySQL       │
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
  ┌─────┴─────┐      ┌─────┴─────┐      ┌─────┴─────┐
  │ Gin #1    │      │ Gin #2    │      │ Gin #3    │
  │ :8080     │      │ :8081     │      │ :8082     │
  │ WS Hub    │      │ WS Hub    │      │ WS Hub    │
  └─────┬─────┘      └─────┬─────┘      └─────┬─────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                    ┌──────┴───────┐
                    │  Redis       │
                    │  ├─ Pub/Sub  │ (WS 跨实例广播)
                    │  ├─ Queue    │ (Asynq 任务队列)
                    │  └─ Cache    │ (会话/热点数据)
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐
                    │  Worker #1   │ (Asynq Worker)
                    │  Worker #2   │
                    │  Worker #N   │
                    │  Docker SDK  │
                    └──────────────┘
```

---

## 六、AI Agent 框架：Eino + 自研封装

### 架构分层

```
┌──────────────────────────────────────────┐
│  anserflow/internal/agent/  (自研业务层)  │
│  ├── orchestrator.go    讨论编排          │
│  ├── command_handler.go /backlog 指令识别 │
│  ├── executor.go        Docker沙箱调度    │
│  ├── skill_loader.go    Skills 加载       │
│  ├── group_discuss.go   群聊 Agent 调度   │
│  ├── backlog_parser.go   方案→Issue 拆解   │
│  ├── prompt_optimizer.go 人工提示词 Eino 优化│
│  └── tool/                                 │
│       ├── registry.go    Tool 注册中心     │
│       ├── dispatch.go    LLM 输出解析→执行 │
│       ├── create_issue.go  创建 Issue     │
│       ├── send_message.go  群聊发言       │
│       ├── read_timeline.go 读取时间线     │
│       └── change_status.go 状态流转       │
├──────────────────────────────────────────┤
│  Eino (底层 LLM 引擎)                    │
│  ├── ChatModel         模型调用           │
│  ├── Tool              工具/Skills 抽象   │
│  ├── Graph             多Agent编排        │
│  ├── Callbacks         回调/日志/监控     │
│  └── Flow              流式处理           │
├──────────────────────────────────────────┤
│  ⚠️ Eino 职责边界 + Skill 驱动调度          │
│  Eino 仅负责：Agent 调度 / 群聊讨论编排    │
│             /backlog 方案拆解             │
│             人工提示词优化                  │
│  Eino 不负责：代码生成 / 编码执行          │
│              └→ 由 opencode 在 Docker 沙箱完成│
│                                           │
│  调度行为由 Eino Skill 定义               │
│  (eino-discuss/eino-backlog/eino-optimizer)│
└──────────────────────────────────────────┘
```

### Eino 初始化与配置

Eino 使用 `config.yaml` 中 `llm` 配置段统一管理多 Agent 的模型连接：

```go
// internal/agent/eino_init.go
package agent

import (
    "github.com/cloudwego/eino/components/model"
    "github.com/cloudwego/eino/schema"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

var chatModel model.ChatModel

func InitEino(cfg *config.EinoConfig) error {
    var err error
    chatModel, err = openai.NewChatModel(context.Background(), &openai.ChatModelConfig{
        APIKey:      cfg.APIKey,
        BaseURL:     cfg.BaseURL,
        Model:       cfg.Model,
        MaxTokens:   cfg.MaxTokens,
        Temperature: &cfg.Temperature,
        Timeout:     cfg.Timeout,
    })
    return err
}

// GetChatModel 返回初始化的模型实例（单例）
func GetChatModel() model.ChatModel { return chatModel }
```

### Agent 运行时配置

`agents.runtime_config` JSON 由绑定的运行时决定其 schema。`runtimes.config_schema` 定义了该运行时可配置的所有字段，前端根据 schema 动态生成表单：

**opencode 运行时配置示例**（`runtimes.config_schema` 驱动）：

```json
{
  "provider": "openai",
  "model": "gpt-4o",
  "agent": "build",
  "api_key_encrypted": "aes256:xxx",
  "max_iterations": 20,
  "thinking": true
}
```

**自定义运行时配置示例**（如 claude-code）：

```json
{
  "api_key_encrypted": "aes256:xxx",
  "model": "claude-sonnet-4-20250514",
  "permission_mode": "acceptEdits"
}
```

**配置流转**：

```
Admin UI (Agent 编辑页)
│  ① 下拉选择运行时（opencode / claude-code / ...）
│  ② 前端根据 runtimes.config_schema 动态渲染配置表单
│  ③ 保存 → agents.runtime_config JSON
│
▼
Worker (沙箱启动时)
│  ① 读取 agents.runtime_id → runtimes.execute_template
│  ② 读取 agents.runtime_config → 填充模板变量
│  ③ 执行: opencode run "{prompt}" --model openai/gpt-4o --agent build
```

### ChatModel 调用示例

```go
// internal/agent/group_discuss.go — Agent 参与群聊讨论
func (o *GroupOrchestrator) InvokeAgent(
    ctx context.Context,
    agent *model.Agent,
    messages []*schema.Message,  // 群聊历史上下文
) (*schema.Message, error) {
    // 1. 构建上下文：System Prompt（1-2句话人设） + Eino 调度 Skill 定义
    systemPrompt := agent.SystemPrompt   // 如 "你是前端开发工程师"
    einoSkills := o.skillLoader.LoadByCategory(ctx, "eino")  // eino-discuss/backlog/optimizer
    skillContext := buildSkillContext(einoSkills)

    // 2. 追加当前讨论上下文
    chatInput := append([]*schema.Message{
        schema.SystemMessage(systemPrompt),
    }, messages...)

    // 3. 调用 LLM（带 Token 用量回调）
    resp, err := GetChatModel().Generate(ctx, chatInput,
        model.WithCallbacks(&callbacks.Handler{
            OnEnd: func(ctx context.Context, info *callbacks.RunInfo, output model.CallbackOutput) context.Context {
                // 记录 Token 用量
                o.tokenTracker.Record(agent.ID, output.TokenUsage)
                return ctx
            },
        }),
    )
    if err != nil {
        return nil, fmt.Errorf("agent invoke failed: %w", err)
    }
    return resp, nil
}
```

### Tool / Skill 抽象

```go
// internal/agent/skill_loader.go — Skills 加载为 Eino Tool
type SkillLoader struct {
    skillRepo *repository.SkillRepo
}

func (sl *SkillLoader) LoadAsTools(
    ctx context.Context,
    agentID uint,
) ([]tool.InvokableTool, error) {
    skills, err := sl.skillRepo.FindEnabledByAgent(ctx, agentID)
    if err != nil {
        return nil, err
    }
    tools := make([]tool.InvokableTool, 0, len(skills))
    for _, skill := range skills {
        // 每个 Skill 注册为一个 Eino Tool
        tools = append(tools, &SkillTool{
            name:        skill.Name,
            description: skill.Description,
            definition:  skill.Definition, // Markdown 正文
            handler: func(ctx context.Context, input string) (string, error) {
                // 将 Skill 定义注入 System Prompt 让 LLM 遵循
                return fmt.Sprintf(
                    "Skill '%s' loaded. Instructions:\n%s",
                    skill.Name, skill.Definition,
                ), nil
            },
        })
    }
    return tools, nil
}
```

### Skill 两层继承（沙箱执行时）

Worker 注入 Skills 到沙箱时，合并 Runtime 默认 + Agent 独立绑定，Agent 可覆盖关闭 Runtime 继承的 Skill：

```go
// internal/agent/skill_loader.go
func (sl *SkillLoader) LoadForSandbox(ctx context.Context, agent *model.Agent) ([]*model.Skill, error) {
    // ① Runtime 默认 Skills（如 opencode→flowcode-executor）
    runtimeSkills, _ := sl.skillRepo.FindEnabledByRuntime(ctx, agent.RuntimeID)

    // ② Agent 独立绑定的 Skills
    agentSkills, _ := sl.skillRepo.FindEnabledByAgent(ctx, agent.ID)

    // ③ 合并去重：Agent 级配置覆盖 Runtime 默认（以 skill_id 为 key）
    merged := make(map[uint]*model.Skill)
    for _, s := range runtimeSkills {
        merged[s.ID] = s
    }
    for _, s := range agentSkills {
        merged[s.ID] = s    // Agent 级覆盖（含 enabled=false 的情况）
    }

    // ④ 仅返回 enabled=true 的 Skill
    result := make([]*model.Skill, 0)
    for _, s := range merged {
        if s.Enabled {
            result = append(result, s)
        }
    }
    return result, nil
}
```

**Skill 注入规则**：

| Skill | 来源 | 能否关闭 | 说明 |
|-------|------|---------|------|
| `flowcode-executor` | Runtime 默认（opencode） | ❌ 不可关闭 | `is_builtin=1`，前端灰掉开关 |
| 用户创建的 Skill | Runtime 默认 / Agent 绑定 | ✅ 可开关 | 后台自由管理 |
| Agent 主动关闭 Runtime Skill | Agent 级覆盖 | ✅ | `agent_skills.enabled=false` 覆盖 Runtime 默认 |

### Eino Tool 系统（Skill 与系统通信）

Eino Skill 不只是 Markdown 文档，每个 Skill 声明一组可调用 Tool。Eino 调度 LLM 时，LLM 决定调用哪个 Tool → Eino 执行对应的 Go handler → 操作数据库/群聊/通知。

**Skill 定义格式**（YAML frontmatter + Markdown）：

```markdown
---
name: eino-backlog
description: /backlog 方案拆解规范，将群聊讨论产出为一个 Issue
tools:
  - create_issue     # 创建 Issue（调用 IssueService）
  - read_issues      # 读取已有 Issue（防重复）
  - send_message     # 向群聊发送消息
is_builtin: true
---

# 方案拆解规范

## 触发条件
收到 /backlog 指令时调用 create_issue 工具。

## 创建规则
- title: 简洁的功能描述（<50字）
- description: 技术方案概述 + 验收标准
- priority: P0=核心路径 P1=重要功能 P2=增强
- 调用 read_issues 检查是否已存在相同 Issue
- 创建成功后调用 send_message 通知群聊
```

**Tool 注册与调度**：

```go
// internal/agent/tool/registry.go
type ToolRegistry struct {
    tools map[string]ToolHandler
}

type ToolHandler func(ctx context.Context, params json.RawMessage) (string, error)

func NewRegistry(services *Services) *ToolRegistry {
    r := &ToolRegistry{tools: make(map[string]ToolHandler)}

    // 注册 Eino Skill 可调用的所有 Tool
    r.Register("create_issue",   services.IssueService.CreateFromAgent)
    r.Register("read_issues",    services.IssueService.ListByProject)
    r.Register("send_message",   services.WS.SendToGroup)
    r.Register("read_timeline",  services.TimelineRepo.FindByIssue)
    r.Register("change_status",  services.IssueService.UpdateStatus)
    r.Register("find_agent",     services.AgentRepo.FindByID)
    return r
}

// GetToolsSchema 生成 OpenAI Function Calling 格式的 tools 定义
func (r *ToolRegistry) GetToolsSchema(skillNames []string) []ToolDef {
    // 根据 Skill 声明的 tools 列表，返回对应的 Function 定义
}
```

```go
// internal/agent/tool/dispatch.go
func (d *Dispatcher) Execute(ctx context.Context, llmOutput string, agent *model.Agent) error {
    // ① 解析 LLM 输出的 JSON: {"tool": "create_issue", "params": {...}}
    var call ToolCall
    json.Unmarshal([]byte(llmOutput), &call)

    // ② Casbin 校验（Agent 是否有权限调用此 Tool）
    if !d.enforcer.Enforce(agent, call.Tool) {
        return fmt.Errorf("Agent %s 无权调用 %s", agent.Name, call.Tool)
    }

    // ③ 执行 Tool → 写入 DB / 发送 WS
    result, err := d.registry.Invoke(ctx, call.Tool, call.Params)

    // ④ 注入 agent_logs 记录
    d.logRepo.Create(ctx, &model.AgentLog{
        AgentID: agent.ID,
        Action:  call.Tool,
        Input:   call.Params,
        Output:  json.RawMessage(result),
        Status:  "success",
    })

    // ⑤ 返回结果给 LLM 继续上下文
    return err
}
```

**Eino Tool 与 opencode Tool 对比**：

| | Eino Tool | opencode Tool |
|------|---------|------------|
| 运行位置 | Go 后端进程 | Docker 沙箱内 |
| 操作对象 | 系统数据（Issue/消息/时间线） | 代码文件（read/write/bash） |
| 权限 | Casbin RBAC | 沙箱隔离 |
| 注册方式 | `registry.Register(name, handler)` | opencode 内置 |
| 典型调用 | `create_issue` / `send_message` | `read` / `write` / `bash` |

### `/backlog` 指令识别

Eino Agent 在群聊中监听 WebSocket 消息，检测到 `/backlog` 指令时触发方案拆解流程：

```go
// internal/agent/command_handler.go
type CommandHandler struct {
    orchestrator *GroupOrchestrator
    parser       *BacklogParser
}

func (h *CommandHandler) OnMessage(msg *ws.Message) {
    if !strings.HasPrefix(msg.Content.Text, "/backlog") {
        return
    }

    // ── 输入校验 ──
    text := strings.TrimPrefix(msg.Content.Text, "/backlog")
    text = strings.TrimSpace(text)

    // 收集群聊历史上下文
    history := h.getRecentMessages(msg.GroupID, 50)

    // 校验 1: /backlog 不能为空（无上下文）
    if text == "" && len(history) == 0 {
        h.ws.Reply(msg, "❌ /backlog 需要描述需求或先进行群聊讨论，不能为空。")
        return
    }

    // 校验 2: 群聊上下文不足（防止误触发）
    if len(history) < 3 && text == "" {
        h.ws.Reply(msg, "❌ 群聊讨论内容不足，请先描述需求或补充更多讨论后再 /backlog。")
        return
    }

    // 校验 3: 调用 Eino 编排 Agent 产出单条方案
    plan := h.orchestrator.GeneratePlan(ctx, msg.GroupID, history, text)

    // 校验 4: Eino 返回空方案
    if plan == nil || plan.Title == "" {
        h.ws.Reply(msg, "❌ /backlog 未能产出有效方案，请补充更多需求细节后重试。")
        return
    }

    // 拆解为 1 个 Issue（状态=backlog）
    issue := h.parser.ParseSingle(ctx, plan, msg.GroupID)
    h.createIssue(ctx, issue)
    h.ws.Broadcast(msg.GroupID, ws.Message{
        Type: "backlog",
        Content: map[string]interface{}{
            "issue": issue,
            "hint":  "请到项目 backlog Tab 确认 Issue 细节，确认后点击 [转为 todo] 启动执行",
        },
    })
}
```

### 人工提示词 Eino 优化

Eino 在将人工提示词注入 opencode 之前，自动进行上下文增强与工程化改写：

```go
// internal/agent/prompt_optimizer.go
func (o *PromptOptimizer) Enhance(ctx context.Context, rawPrompt string, issue *model.Issue) (string, error) {
    // ① 收集上下文：Issue 描述 + 已有时间线 + 代码仓库信息
    timeline := o.timelineRepo.FindByIssue(ctx, issue.ID)
    context := buildContext(issue, timeline)

    // ② 解析描述中的 @Issue #N 引用，读取被引用 Issue 的内容
    refs := parseIssueRefs(issue.Description) // 正则: @Issue\s+#(\d+)
    var refContext strings.Builder
    for _, refID := range refs {
        refIssue, _ := o.issueRepo.FindByID(ctx, refID)
        if refIssue != nil {
            refContext.WriteString(fmt.Sprintf(
                "关联 Issue #%d「%s」: %s\n",
                refIssue.ID, refIssue.Title, refIssue.Description,
            ))
        }
    }

    // ③ Eino 调用 LLM 改写提示词（包含 @Issue 上下文）
    messages := []*schema.Message{
        schema.SystemMessage(`你是提示词优化器。将用户反馈改写为精确的编码指令。
规则：保留用户原意；补充技术细节；如果用户提到具体文件/组件，添加文件路径。
如果提供了关联 Issue 的内容，确保生成代码时保持与关联 Issue 的技术方案一致。`),
        schema.UserMessage(fmt.Sprintf(
            "Issue 上下文：\n%s\n\n关联 Issue 内容：\n%s\n\n用户提示词：%s\n\n输出优化后的编码指令：",
            context, refContext.String(), rawPrompt,
        )),
    }
    resp, _ := GetChatModel().Generate(ctx, messages)
    return resp.Content, nil
}

// 正则: 匹配 "需要 @Issue #42" 或 "参考 @Issue #8 的实现"
func parseIssueRefs(text string) []uint {
    re := regexp.MustCompile(`@Issue\s+#(\d+)`)
    matches := re.FindAllStringSubmatch(text, -1)
    ids := make([]uint, 0, len(matches))
    for _, m := range matches {
        id, _ := strconv.ParseUint(m[1], 10, 64)
        ids = append(ids, uint(id))
    }
    return ids
}
```

> **关键**：Eino 只做调度与提示词优化，不进入 Docker 沙箱。沙箱内的代码生成完全由 opencode 完成。

### Token 用量与成本追踪

```go
// internal/agent/token_tracker.go
type TokenTracker struct {
    redis *redis.Client
}

// Record 记录单次调用用量（按 Agent + 日期聚合）
func (t *TokenTracker) Record(agentID uint, usage *schema.TokenUsage) {
    key := fmt.Sprintf("tokens:agent:%d:date:%s", agentID, time.Now().Format("2006-01-02"))
    t.redis.HIncrBy(ctx, key, "prompt_tokens", int64(usage.PromptTokens))
    t.redis.HIncrBy(ctx, key, "completion_tokens", int64(usage.CompletionTokens))
    t.redis.Expire(ctx, key, 30*24*time.Hour) // 保留 30 天
}

// GetDailyUsage 获取指定日期用量
func (t *TokenTracker) GetDailyUsage(
    agentID uint, date string,
) (prompt, completion int64, err error) {
    key := fmt.Sprintf("tokens:agent:%d:date:%s", agentID, date)
    cmd := t.redis.HMGet(ctx, key, "prompt_tokens", "completion_tokens")
    // ...
}
```

> **LLM API Key 安全模型**：API Key 在 `agents.runtime_config.llm.api_key_encrypted` 中以 AES-256 加密存储；Agent 执行时 Worker 解密后通过环境变量注入 Docker 沙箱，不写入容器文件系统。

#### Token 用量 API 暴露

提供按 Agent/按组织维度的 Token 用量查询接口，用于 Dashboard 成本展示：

```go
// internal/handler/token_handler.go
type TokenUsageResponse struct {
    AgentID          uint    `json:"agent_id"`
    AgentName        string  `json:"agent_name"`
    Date             string  `json:"date"`              // "2026-05-14"
    PromptTokens     int64   `json:"prompt_tokens"`
    CompletionTokens int64   `json:"completion_tokens"`
    TotalTokens      int64   `json:"total_tokens"`
    EstimatedCostUSD float64 `json:"estimated_cost_usd"` // 按 provider 单价估算
}

type OrgTokenSummary struct {
    Period            string                      `json:"period"`             // "7d" / "30d" / "custom"
    TotalPromptTokens int64                       `json:"total_prompt_tokens"`
    TotalCompTokens   int64                       `json:"total_completion_tokens"`
    TotalCostUSD      float64                     `json:"total_cost_usd"`
    ByAgent           []TokenUsageResponse        `json:"by_agent"`
    ByDate            []TokenUsageResponse        `json:"by_date"`
}

// GET /api/orgs/:org_id/agents/:agent_id/token-usage?from=2026-05-01&to=2026-05-14
func (h *TokenHandler) GetAgentTokenUsage(c *gin.Context) {
    agentID := c.Param("agent_id")
    from := c.DefaultQuery("from", time.Now().AddDate(0, 0, -7).Format("2006-01-02"))
    to := c.DefaultQuery("to", time.Now().Format("2006-01-02"))

    usage := h.tracker.GetDailyUsage(ctx, agentID, from, to)
    // 按 provider 单价估算成本（gpt-4o: $2.50/$10.00, gpt-4o-mini: $0.15/$0.60 per 1M tokens）
    cost := estimateCost(agent.Provider, usage.PromptTokens, usage.CompletionTokens)
    // ...
}

// GET /api/orgs/:org_id/token-summary?period=7d
func (h *TokenHandler) GetOrgTokenSummary(c *gin.Context) {
    orgID := c.Param("org_id")
    // 聚合组织下所有 Agent 的 Token 用量
}
```

**成本估算函数**：

```go
// internal/agent/cost.go
var providerPricing = map[string]struct{ Prompt, Completion float64 }{
    "openai/gpt-4o":       {2.50, 10.00},   // per 1M tokens
    "openai/gpt-4o-mini":  {0.15, 0.60},
    "anthropic/claude-3.5":{3.00, 15.00},
    "deepseek/deepseek-v3":{0.14, 0.28},
}

func estimateCost(providerKey string, promptTokens, completionTokens int64) float64 {
    p, ok := providerPricing[providerKey]
    if !ok {
        return 0 // 未知 provider 不估算
    }
    return float64(promptTokens)/1e6*p.Prompt + float64(completionTokens)/1e6*p.Completion
}
```

---
