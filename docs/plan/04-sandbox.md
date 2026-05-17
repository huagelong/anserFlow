# AnserFlow - Sandbox / Agent

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
│  ├── prompt_manager.go  提示词统一管理     │
│  └── tool/                                 │
│       ├── registry.go    Tool 注册中心     │
│       ├── dispatch.go    LLM 输出解析→执行 │
│       ├── create_issue.go  创建 Issue     │
│       ├── send_message.go  群聊发言       │
│       ├── read_timeline.go 读取时间线     │
│       ├── change_status.go 状态流转       │
├──────────────────────────────────────────┤
│  anserflow/prompts/  (提示词统一管理)      │
│  ├── agent_optimizer.go  anserAgent 优化器提示词│
│  ├── agent_mention.go    @Agent 任务注入模板  │
│  ├── agent_memory.go     五层记忆注入提示词模板│
│  ├── agent_new_session.go 新会话提示词       │
│  ├── system_messages.go  群聊系统消息模板     │
│  ├── notification.go     通知标题/正文模板    │
│  └── error_templates.go  错误消息模板        │
├──────────────────────────────────────────┤
│  anserflow/internal/status/  (状态机)      │
│  └── status_manager.go  Issue 状态流转管理  │
├──────────────────────────────────────────┤
│  anserflow/internal/sandbox/ (沙箱生命周期) │
│  └── sandbox_manager.go 创建/暂停/恢复/销毁 │
├──────────────────────────────────────────┤
│  anserflow/internal/runtime/ (运行时管理)   │
│  └── runtime_manager.go 配置构建/命令渲染   │
├──────────────────────────────────────────┤
│  anserflow/internal/notification/ (通知)   │
│  └── channel_manager.go 多渠道路由分发      │
├──────────────────────────────────────────┤
│  anserflow/internal/git/  (Git 管理)       │
│  └── manager.go        GitManager 统一入口 │
├──────────────────────────────────────────┤
│  anserflow/internal/token/  (Token 配额)   │
│  └── token_manager.go   用量追踪/配额/归档  │
├──────────────────────────────────────────┤
│  Eino (底层 LLM 引擎)                    │
│  ├── ChatModel         模型调用           │
│  ├── Tool              工具/Skills 抽象   │
│  ├── Graph             多Agent编排        │
│  ├── Callbacks         回调/日志/监控     │
│  └── Flow              流式处理           │
├──────────────────────────────────────────┤
│  ⚠️ anserAgent 职责边界 + 五层记忆驱动          │
│  anserAgent 负责：Agent 调度 / 群聊讨论编排    │
│               /backlog 方案拆解               │
│               人工提示词优化 / Skill 自改进    │
│  anserAgent 不负责：代码生成 / 编码执行        │
│              └→ 由运行时（opencode/hermes）在 Docker 沙箱完成│
│                                           │
│  调度行为由五层记忆 + L3 Skills/SOPs 驱动    │
│  详见 docs/plan/06-agent.md               │
└──────────────────────────────────────────┘
```

**Agent 编排通用规则（群聊 + 双人聊）**：

Hub 路由层统一根据 `HasAgentMember()` 决定是否调用 Agent 组件，不再区分 `group.type`：

| 场景 | GroupOrchestrator | CommandHandler | MentionResolver | 说明 |
|------|-------------------|----------------|-----------------|------|
| 群聊（含 Agent） | ✅ 调用 | ✅ /backlog /todo /new 可用 | ✅ 调用 | 原有群聊行为不变 |
| 群聊（无 Agent） | ❌ 不调用 | ✅ 仅 /new 可用 | ❌ 不调用 | 纯自然人聊天，不触发 Eino |
| 双人聊（人+Agent） | ✅ 调用 | ✅ /backlog /todo /new 可用 | ❌ 不调用 | Agent 自动参与，与群聊含 Agent 一致 |
| 双人聊（人+人） | ❌ 不调用 | ✅ 仅 /new 可用 | ❌ 不调用 | 纯自然人聊天，不触发 Eino |

以下组件**零修改**，Hub 层通过 `HasAgentMember()` 控制调用入口：
- `GroupOrchestrator.InvokeAgent` — 有 Agent 的群聊和双人聊直接复用
- `CommandHandler` — Hub 判断后调用；/new 全模式可用，/backlog 和 /todo 无 Agent 时返回提示
- `ToolRegistry` — Agent Tool（create_issue / send_message 等）在所有含 Agent 的会话中同样适用

### Eino 初始化与配置

Eino 使用 `config.yaml` 中 `llm` 配置段统一管理多 Agent 的模型连接：

```go
// internal/agent/agent_init.go
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

**配置流转**：

```
Admin UI (Agent 编辑页)
│  ① 下拉选择运行时（opencode / hermes / ...）
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
// 现由 anserAgent 统一编排，此处仅保留调用入口
func (o *GroupOrchestrator) InvokeAgent(
    ctx context.Context,
    agent *model.Agent,
    messages []*schema.Message,  // 群聊历史上下文
) (*schema.Message, error) {
    // 委托给 anserAgent（详见 06-agent.md）
    resp, err := o.anserAgent.Invoke(ctx, AgentInput{
        Context:  messages,
        Mode:     ModeOrchestrate,
        Tags:     []string{"group_discuss"},
    })
    if err != nil {
        return nil, fmt.Errorf("anserAgent invoke failed: %w", err)
    }
    return &schema.Message{Content: resp.Content}, nil
}
```

### Tool / Skill 抽象

```go
// internal/agent/skill_loader.go — Skills 加载为 anserAgent Tool
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
        // 每个 Skill 注册为一个 anserAgent Tool
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
    // ① Runtime 默认 Skills（如 opencode→anser-coder）
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
| `anser-coder` | Runtime 默认（opencode） | ❌ 不可关闭 | `is_builtin=1`，前端灰掉开关 |
| 用户创建的 Skill | Runtime 默认 / Agent 绑定 | ✅ 可开关 | 后台自由管理 |
| Agent 主动关闭 Runtime Skill | Agent 级覆盖 | ✅ | `agent_skills.enabled=false` 覆盖 Runtime 默认 |

### 提示词管理器（PromptManager）

系统中除 Agent System Prompt（DB 存储）和 Skill 定义（DB 存储）外，所有硬编码的提示词模板统一抽到 `prompts/` 目录，由 `PromptManager` 集中加载和管理。

**设计原则**：

- 提示词模板与业务逻辑分离，便于调优、国际化、版本管理
- 运行时通过 `PromptManager.Get("key")` 获取，不直接拼接字符串
- DB 存储的 Agent System Prompt 和 Skill 定义不动，仍走后台 CRUD

**提示词文件清单**：

| 文件 | key | 用途 | 原硬编码位置 |
|------|-----|------|------------|
| `agent_optimizer.go` | `agent.optimizer.system` | anserAgent 优化器系统提示词 | `prompt_optimizer.go` |
| `agent_optimizer.go` | `agent.optimizer.user` | anserAgent 优化器用户消息模板 | `prompt_optimizer.go` |
| `agent_mention.go` | `agent.mention.system` | @Agent 任务注入系统提示词 | `group_discuss.go` |
| `agent_memory.go` | `agent.memory.inject` | 五层记忆注入模板 | `memory/manager.go` |
| `agent_new_session.go` | `agent.new_session.hint` | 新会话 Agent 感知提示词 | `group_discuss.go` |
| `system_messages.go` | `system.issue.{status_change}` | 群聊系统消息模板（按状态变更） | `IssueService` |
| `notification.go` | `notify.issue_status_changed.title` | 通知标题模板 | `notification_service.go` |
| `error_templates.go` | `error.backlog_empty` | /backlog 错误提示 | `command_handler.go` |
| `error_templates.go` | `error.backlog_insufficient` | /backlog 上下文不足 | `command_handler.go` |
| `error_templates.go` | `error.backlog_no_plan` | /backlog 生成失败 | `command_handler.go` |

**实现**：

```go
// prompts/prompt_manager.go
package prompts

import (
    "embed"
    "sync"
)

//go:embed *.go
var promptsFS embed.FS

type PromptManager struct {
    mu       sync.RWMutex
    prompts  map[string]string
}

var defaultManager *PromptManager

func init() {
    defaultManager = &PromptManager{prompts: make(map[string]string)}
    defaultManager.loadAll()
}

// Get 获取提示词模板（支持占位符替换）
func Get(key string, args ...interface{}) string {
    defaultManager.mu.RLock()
    defer defaultManager.mu.RUnlock()
    tmpl, ok := defaultManager.prompts[key]
    if !ok {
        return key // fallback: 返回 key 本身
    }
    if len(args) > 0 {
        return fmt.Sprintf(tmpl, args...)
    }
    return tmpl
}

// MustGet 获取提示词模板（找不到则 panic）
func MustGet(key string, args ...interface{}) string {
    result := Get(key, args...)
    if result == key {
        panic(fmt.Sprintf("prompt not found: %s", key))
    }
    return result
}

// Reload 热重载（开发/运维调试用）
func Reload() {
    defaultManager.mu.Lock()
    defer defaultManager.mu.Unlock()
    defaultManager.loadAll()
}
```

```go
// prompts/agent_optimizer.go
package prompts

func init() {
    defaultManager.prompts["agent.optimizer.system"] = `你是提示词优化器。将用户反馈改写为精确的编码指令。
规则：保留用户原意；补充技术细节；如果用户提到具体文件/组件，添加文件路径。
如果提供了关联 Issue 的内容，确保生成代码时保持与关联 Issue 的技术方案一致。`

    defaultManager.prompts["agent.optimizer.user"] = `Issue 上下文：
%s

关联 Issue 内容：
%s

用户提示词：%s

输出优化后的编码指令：`
}
```

```go
// prompts/agent_mention.go
package prompts

func init() {
    defaultManager.prompts["agent.mention.system"] = `你是 %s。%s 正在群聊中向你布置任务。`
}
```

```go
// prompts/agent_memory.go
package prompts

func init() {
    defaultManager.prompts["agent.memory.inject"] = `[五层记忆] 以下是你的相关记忆，用于辅助决策：\n%s`
}
```

```go
// prompts/agent_new_session.go
package prompts

func init() {
    defaultManager.prompts["agent.new_session.hint"] = `这是一个新讨论主题的开端`
}
```

```go
// prompts/system_messages.go
package prompts

func init() {
    defaultManager.prompts["system.issue.backlog_created"] = `已生成 Issue #%d（backlog），请到 backlog Tab 确认细节并启动`
    defaultManager.prompts["system.issue.to_todo"] = `Issue #%d 已确认转为 todo，排队等待执行`
    defaultManager.prompts["system.issue.start"] = `Issue #%d 开始执行，Agent 已启动`
    defaultManager.prompts["system.issue.paused"] = `Issue #%d 执行已暂停`
    defaultManager.prompts["system.issue.stopped"] = `Issue #%d 执行已停止`
    defaultManager.prompts["system.issue.pr_submitted"] = `Issue #%d PR 已提交，等待审核`
    defaultManager.prompts["system.issue.failed"] = `Issue #%d 执行失败: %s`
    defaultManager.prompts["system.issue.done"] = `Issue #%d 已完成 ✅`
    defaultManager.prompts["system.issue.pr_rejected"] = `Issue #%d PR 被拒绝，已退回 todo`
}
```

```go
// prompts/notification.go
package prompts

func init() {
    defaultManager.prompts["notify.issue_status_changed.title"] = `Issue #%d 状态变更为 %s`
}
```

```go
// prompts/error_templates.go
package prompts

func init() {
    defaultManager.prompts["error.backlog_empty"] = `❌ 需要描述需求或先进行群聊讨论，不能为空。`
    defaultManager.prompts["error.backlog_insufficient"] = `❌ 群聊讨论内容不足，请先描述需求或补充更多讨论后再试。`
    defaultManager.prompts["error.backlog_no_plan"] = `❌ 未能产出有效方案，请补充更多需求细节后重试。`
}
```

**业务代码调用示例**（原硬编码替换为 PromptManager）：

```go
// 替换前（硬编码）：
schema.SystemMessage(`你是提示词优化器。将用户反馈改写为精确的编码指令。...`)

// 替换后（从 PromptManager 获取）：
schema.SystemMessage(prompts.Get("agent.optimizer.system"))

// 替换前：
h.ws.Reply(msg, "❌ 需要描述需求或先进行群聊讨论，不能为空。")

// 替换后：
h.ws.Reply(msg, prompts.Get("error.backlog_empty"))

// 替换前：
s.messageService.SendSystemMessage(ctx, issue.SourceGroupID, fmt.Sprintf(
    "Issue #%d %s → %s", issue.ID, oldStatus, newStatus))

// 替换后：
s.messageService.SendSystemMessage(ctx, issue.SourceGroupID,
    prompts.Get(fmt.Sprintf("system.issue.%s", statusKey), issue.ID))
```

### IssueStatusManager — Issue 状态机管理器

Issue 状态流转逻辑分散在 `IssueService`、`Worker`、`Scheduler`、`WebhookHandler` 四处，通过 `StatusManager` 集中管理状态机规则和副作用触发。

**设计原则**：

- 状态流转合法表集中定义，不散落在 if/switch 中
- 每次流转的副作用（群聊通知、时间线、通知推送）统一 Hook 回调
- 新增状态或修改流转规则只需改一处

**状态流转合法表**：

| from | to | 触发方 | 副作用 |
|------|----|--------|--------|
| `backlog` | `todo` | 人工确认 | 群聊通知 + 时间线 |
| `todo` | `in_progress` | Scheduler 分配 | 群聊通知 + 时间线 + 通知被分配人 |
| `in_progress` | `in_review` | Worker PR 提交 | 群聊通知 + 时间线 + 通知 Reviewer |
| `in_progress` | `paused` | 人工暂停 | 群聊通知 + 时间线 |
| `in_progress` | `todo` | Worker 执行失败 | 群聊通知 + 时间线 + retry_count++ |
| `in_review` | `done` | GitHub Webhook merge | 群聊通知 + 时间线 |
| `in_review` | `todo` | GitHub Webhook reject | 群聊通知 + 时间线（沙箱保留） |
| `todo` | `backlog` | 人工退回 | 时间线 |

**实现**：

```go
// internal/status/status_manager.go
package status

import (
    "context"
    "fmt"
)

// Transition 表示一次状态流转
// type Transition struct {
//     From string
//     To   string
// }

type TransitionHook func(ctx context.Context, issueID uint, from, to string) error

type StatusManager struct {
    transitions map[string]map[string]bool   // from -> to -> allowed
    hooks       map[string][]TransitionHook   // "from->to" -> hooks
}

func NewStatusManager() *StatusManager {
    m := &StatusManager{
        transitions: make(map[string]map[string]bool),
        hooks:       make(map[string][]TransitionHook),
    }
    // 注册合法流转
    m.allow("backlog", "todo")
    m.allow("todo", "in_progress")
    m.allow("todo", "backlog")
    m.allow("in_progress", "in_review")
    m.allow("in_progress", "paused")
    m.allow("in_progress", "todo")
    m.allow("in_review", "done")
    m.allow("in_review", "todo")
    m.allow("paused", "in_progress")
    return m
}

func (m *StatusManager) allow(from, to string) {
    if m.transitions[from] == nil {
        m.transitions[from] = make(map[string]bool)
    }
    m.transitions[from][to] = true
}

// OnTransition 注册流转副作用 Hook
func (m *StatusManager) OnTransition(from, to string, hook TransitionHook) {
    key := from + "->" + to
    m.hooks[key] = append(m.hooks[key], hook)
}

// CanTransition 检查流转是否合法
func (m *StatusManager) CanTransition(from, to string) bool {
    return m.transitions[from][to]
}

// Transition 执行状态流转（校验 + 触发 hooks）
func (m *StatusManager) Transition(ctx context.Context, issueID uint, from, to string) error {
    if !m.CanTransition(from, to) {
        return fmt.Errorf("invalid transition: %s -> %s", from, to)
    }
    key := from + "->" + to
    for _, hook := range m.hooks[key] {
        if err := hook(ctx, issueID, from, to); err != nil {
            return fmt.Errorf("hook failed for %s: %w", key, err)
        }
    }
    return nil
}
```

**业务代码接入**：

```go
// 初始化时注册 hooks
statusMgr := status.NewStatusManager()

// 所有含群聊通知的流转都注册群聊 Hook
statusMgr.OnTransition("backlog", "todo",
    func(ctx context.Context, issueID uint, from, to string) error {
        issue, _ := issueRepo.FindByID(ctx, issueID)
        if issue.SourceGroupID != 0 {
            msgService.SendSystemMessage(ctx, issue.SourceGroupID,
                prompts.Get("system.issue.to_todo", issueID))
        }
        return nil
    },
)

// 业务代码调用：一行代替多处 if/通知/时间线
statusMgr.Transition(ctx, issueID, "backlog", "todo")
```

### SandboxManager — 沙箱生命周期管理器

沙箱创建、暂停、恢复、销毁、复用的逻辑分散在 Worker 和 executor.go 中，通过 `SandboxManager` 统一 Docker API 调用。

**设计原则**：

- Worker 只调 `SandboxManager.Create/Resume/Destroy`，不直接操作 Docker SDK
- 资源限额、网络隔离、超时清理等策略集中管理
- 便于替换沙箱方案（如 Firecracker）

**实现**：

```go
// internal/sandbox/sandbox_manager.go
package sandbox

import (
    "context"
    "fmt"
    "time"

    "github.com/docker/docker/api/types/container"
    "github.com/docker/docker/client"
)

type SandboxConfig struct {
    IssueID        uint
    Image          string
    AllowedDomains []string
    MaxMemoryMB    int
    TimeoutMinutes int
    EnvVars        map[string]string
    ExistingID     string // 复用已有容器
}

type SandboxHandle struct {
    ContainerID string
    CreatedAt    time.Time
}

type SandboxManager struct {
    cli    *client.Client
    config Config
}

func NewSandboxManager(cli *client.Client, cfg Config) *SandboxManager {
    return &SandboxManager{cli: cli, config: cfg}
}

// Create 创建沙箱容器（网络隔离 + 资源限额）
func (m *SandboxManager) Create(ctx context.Context, cfg SandboxConfig) (*SandboxHandle, error) {
    if cfg.ExistingID != "" {
        return &SandboxHandle{ContainerID: cfg.ExistingID}, nil
    }
    // 创建容器 + 网络隔离 + 资源限额
    resp, err := m.cli.ContainerCreate(ctx,
        &container.Config{Image: cfg.Image, Env: buildEnv(cfg.EnvVars)},
        &container.HostConfig{
            Memory:     int64(cfg.MaxMemoryMB) * 1024 * 1024,
            AutoRemove: false,  // 不自动销毁，保障 Worker 重启后可恢复
        },
        nil, nil, fmt.Sprintf("anserflow-issue-%d", cfg.IssueID),
    )
    if err != nil {
        return nil, fmt.Errorf("create sandbox: %w", err)
    }
    return &SandboxHandle{ContainerID: resp.ID, CreatedAt: time.Now()}, nil
}

// Pause 冻结容器（进程挂起，内存保留）
func (m *SandboxManager) Pause(ctx context.Context, containerID string) error {
    return m.cli.ContainerPause(ctx, containerID)
}

// Resume 解冻容器
func (m *SandboxManager) Resume(ctx context.Context, containerID string) error {
    return m.cli.ContainerUnpause(ctx, containerID)
}

// Destroy 销毁容器
func (m *SandboxManager) Destroy(ctx context.Context, containerID string) error {
    return m.cli.ContainerRemove(ctx, containerID,
        container.RemoveOptions{Force: true})
}

// IsAlive 检查容器是否运行中
func (m *SandboxManager) IsAlive(ctx context.Context, containerID string) bool {
    _, err := m.cli.ContainerInspect(ctx, containerID)
    return err == nil
}
```

### RuntimeManager — 运行时管理器（适配器模式）

运行时配置构建（解密 API Key、拼接 execute_template、写 config.json）当前直接写在 Worker 中，通过 `RuntimeAdapter` 接口抽象运行时差异，使切换 AI 工具只需实现接口 + 在 `runtimes` 表插入一行。

**设计原则**：

- `RuntimeAdapter` 接口定义配置注入、环境变量映射等运行时差异点
- `OutputParser` 接口定义 stdout 解析、Token 采集等输出差异点
- 新运行时注册只需：实现接口 + 插入 `runtimes` 表
- Worker 不感知具体运行时，只调用接口

**接口定义**：

```go
// internal/runtime/adapter.go
package runtime

// RuntimeAdapter 运行时适配器 — 封装不同 AI 工具的配置注入差异
// 新运行时只需实现此接口，注册到 Registry 即可
type RuntimeAdapter interface {
    // Name 运行时标识（对应 runtimes.name）
    Name() string

    // HomeDir 运行时在容器内的主目录（bind mount 目标）
    // 项目级 runtime 目录会整体 bind mount 到此路径（读写）
    // opencode → /home/sandbox/.opencode
    // hermes → /home/sandbox/.hermes
    HomeDir() string

    // ConfigPath 配置文件写入路径（容器内，HomeDir 的子路径）
    // opencode → /home/sandbox/.opencode/config.json
    // hermes → /home/sandbox/.hermes/config.yaml
    ConfigPath() string

    // RenderConfig 将通用配置渲染为该工具特有的配置 JSON
    // 不同工具的配置结构差异封装在此方法内
    RenderConfig(config map[string]interface{}) (string, error)

    // EnvMapping API Key → 环境变量映射
    // opencode + openai → OPENAI_API_KEY
    // hermes + openrouter → OPENROUTER_API_KEY
    EnvMapping(config map[string]interface{}, decryptedKey string) map[string]string

    // SkillsMountPath Skills 目录在容器内的路径（HomeDir 子路径）
    // opencode → /home/sandbox/.opencode/skills
    // hermes → /home/sandbox/.hermes/skills
    SkillsMountPath() string

    // SessionPath 会话文件路径（用于事后 Token 汇总），空字符串表示不支持
    // opencode → /home/sandbox/.local/share/opencode/sessions/*.jsonl
    // hermes → /home/sandbox/.hermes/sessions/*.jsonl
    SessionPath() string
}

// OutputParser 输出解析器 — 封装不同 AI 工具的 stdout 格式差异
type OutputParser interface {
    // ParseLine 解析单行 stdout 输出
    // 返回 nil 表示该行不是结构化事件（忽略或按纯文本处理）
    ParseLine(line []byte) *ParsedEvent

    // ParseSessionFile 解析会话文件（执行完成后调用）
    // 返回 nil 表示该运行时不支持 session 文件
    ParseSessionFile(content []byte) (*TokenSummary, error)
}

// ParsedEvent 解析后的单行事件
type ParsedEvent struct {
    Type       string            // "agent_log" / "token_usage" / "error"
    Content    string            // 文本内容（写入 issue_timeline）
    TokenUsage *TokenUsageDetail // Token 用量（非空时写入 token_tracker）
}

// TokenUsageDetail Token 用量明细
type TokenUsageDetail struct {
    InputTokens      int64
    OutputTokens     int64
    CacheReadTokens  int64
    CacheWriteTokens int64
}

// TokenSummary 会话文件 Token 汇总
type TokenSummary struct {
    TotalInput  int64
    TotalOutput int64
}
```

**Registry — 运行时注册表**：

```go
// internal/runtime/registry.go
package runtime

// Registry 全局运行时注册表，初始化时注册内置运行时
type Registry struct {
    adapters map[string]RuntimeAdapter
    parsers  map[string]OutputParser
}

func NewRegistry() *Registry {
    r := &Registry{
        adapters: make(map[string]RuntimeAdapter),
        parsers:  make(map[string]OutputParser),
    }
    // 注册内置运行时
    r.Register(&OpenCodeAdapter{}, &OpenCodeParser{})
    r.Register(&HermesAdapter{}, &HermesParser{})
    return r
}

func (r *Registry) Register(a RuntimeAdapter, p OutputParser) {
    r.adapters[a.Name()] = a
    r.parsers[a.Name()] = p
}

func (r *Registry) GetAdapter(name string) RuntimeAdapter { return r.adapters[name] }
func (r *Registry) GetParser(name string) OutputParser    { return r.parsers[name] }
```

**内置实现 — opencode 适配器**：

```go
// internal/runtime/adapters/opencode.go
package adapters

type OpenCodeAdapter struct{}

func (a *OpenCodeAdapter) Name() string { return "opencode" }
func (a *OpenCodeAdapter) HomeDir() string  { return "/home/sandbox/.opencode" }
func (a *OpenCodeAdapter) ConfigPath() string {
    return "/home/sandbox/.opencode/config.json"
}
func (a *OpenCodeAdapter) SkillsMountPath() string {
    return "/home/sandbox/.opencode/skills"
}
func (a *OpenCodeAdapter) SessionPath() string {
    return "/home/sandbox/.local/share/opencode/sessions/*.jsonl"
}

func (a *OpenCodeAdapter) RenderConfig(config map[string]interface{}) (string, error) {
    cfg := map[string]interface{}{
        "model":    config["model"],
        "provider": config["provider"],
        "agent":    config["agent"],
        "thinking":  config["thinking"],
        "permissions": map[string]bool{
            "read": true, "write": true, "bash": true, "glob": true, "grep": true,
        },
    }
    if maxIter, ok := config["max_iterations"]; ok {
        cfg["maxTurns"] = maxIter
    }
    return toJSON(cfg), nil
}

func (a *OpenCodeAdapter) EnvMapping(config map[string]interface{}, key string) map[string]string {
    provider := config["provider"].(string)
    return map[string]string{
        strings.ToUpper(provider) + "_API_KEY": key,
    }
}

type OpenCodeParser struct{}

func (p *OpenCodeParser) ParseLine(line []byte) *ParsedEvent {
    var event map[string]interface{}
    if json.Unmarshal(line, &event) != nil {
        return nil // 非 JSON，由 Worker 兼容逻辑处理
    }
    result := &ParsedEvent{}
    if usage, ok := event["token_usage"]; ok {
        m := usage.(map[string]interface{})
        result.TokenUsage = &TokenUsageDetail{
            InputTokens:      toInt64(m["input_tokens"]),
            OutputTokens:     toInt64(m["output_tokens"]),
            CacheReadTokens:  toInt64(m["cache_read_input_tokens"]),
            CacheWriteTokens: toInt64(m["cache_creation_input_tokens"]),
        }
    }
    if content, ok := event["content"]; ok {
        result.Type = "agent_log"
        result.Content = fmt.Sprintf("%v", content)
    }
    return result
}

func (p *OpenCodeParser) ParseSessionFile(content []byte) (*TokenSummary, error) {
    var totalInput, totalOutput int64
    scanner := bufio.NewScanner(bytes.NewReader(content))
    for scanner.Scan() {
        var msg struct {
            TokenUsage *struct {
                InputTokens  int64 `json:"input_tokens"`
                OutputTokens int64 `json:"output_tokens"`
            } `json:"token_usage"`
        }
        if json.Unmarshal(scanner.Bytes(), &msg) == nil && msg.TokenUsage != nil {
            totalInput += msg.TokenUsage.InputTokens
            totalOutput += msg.TokenUsage.OutputTokens
        }
    }
    return &TokenSummary{TotalInput: totalInput, TotalOutput: totalOutput}, nil
}
```

**内置实现 — hermes 适配器**：

Hermes Agent（by Nous Research）是开源 AI Agent，Python 编写，支持 20+ Provider、持久记忆、Skills 系统、MCP 集成。配置目录 `~/.hermes/`，配置文件 `config.yaml`（非机密）+ `.env`（API Key），Skills 路径 `~/.hermes/skills/`，Session 路径 `~/.hermes/sessions/`。

```go
// internal/runtime/adapters/hermes.go
package adapters

type HermesAdapter struct{}

func (a *HermesAdapter) Name() string        { return "hermes" }
func (a *HermesAdapter) HomeDir() string     { return "/home/sandbox/.hermes" }
func (a *HermesAdapter) ConfigPath() string  { return "/home/sandbox/.hermes/config.yaml" }
func (a *HermesAdapter) SkillsMountPath() string {
    return "/home/sandbox/.hermes/skills"
}
func (a *HermesAdapter) SessionPath() string {
    return "/home/sandbox/.hermes/sessions/*.jsonl"
}

func (a *HermesAdapter) RenderConfig(config map[string]interface{}) (string, error) {
    provider := config["provider"].(string)
    model := config["model"].(string)
    yaml := fmt.Sprintf("model: %s/%s\n", provider, model)
    if personality, ok := config["personality"]; ok {
        yaml += fmt.Sprintf("personality: %s\n", personality)
    }
    if maxTurns, ok := config["max_iterations"]; ok {
        yaml += fmt.Sprintf("max_turns: %v\n", maxTurns)
    }
    yaml += "terminal:\n  backend: local\n"
    return yaml, nil
}

func (a *HermesAdapter) EnvMapping(config map[string]interface{}, key string) map[string]string {
    provider := config["provider"].(string)
    envKey := strings.ToUpper(provider) + "_API_KEY"
    // OpenRouter 使用 OPENROUTER_API_KEY
    if provider == "openrouter" {
        envKey = "OPENROUTER_API_KEY"
    }
    return map[string]string{envKey: key}
}

type HermesParser struct{}

func (p *HermesParser) ParseLine(line []byte) *ParsedEvent {
    var event map[string]interface{}
    if json.Unmarshal(line, &event) != nil {
        return nil
    }
    result := &ParsedEvent{}
    if usage, ok := event["token_usage"]; ok {
        m := usage.(map[string]interface{})
        result.TokenUsage = &TokenUsageDetail{
            InputTokens:  toInt64(m["input_tokens"]),
            OutputTokens: toInt64(m["output_tokens"]),
        }
    }
    if content, ok := event["content"]; ok {
        result.Type = "agent_log"
        result.Content = fmt.Sprintf("%v", content)
    }
    return result
}

func (p *HermesParser) ParseSessionFile(content []byte) (*TokenSummary, error) {
    // Hermes session 文件格式待确认，暂按 JSONL 解析
    var totalInput, totalOutput int64
    scanner := bufio.NewScanner(bytes.NewReader(content))
    for scanner.Scan() {
        var msg struct {
            TokenUsage *struct {
                InputTokens  int64 `json:"input_tokens"`
                OutputTokens int64 `json:"output_tokens"`
            } `json:"token_usage"`
        }
        if json.Unmarshal(scanner.Bytes(), &msg) == nil && msg.TokenUsage != nil {
            totalInput += msg.TokenUsage.InputTokens
            totalOutput += msg.TokenUsage.OutputTokens
        }
    }
    return &TokenSummary{TotalInput: totalInput, TotalOutput: totalOutput}, nil
}
```

**RuntimeManager 使用适配器**：

```go
// internal/runtime/runtime_manager.go
package runtime

type ResolvedSandboxConfig struct {
    ExecCommand   string            // 最终执行命令
    ContainerEnvs map[string]string // 注入容器的环境变量
    ConfigJSON    string            // 写入容器的 config.json
    ConfigPath    string            // 配置文件路径（HomeDir 子路径）
    HomeDir       string            // 运行时主目录（bind mount 目标）
    SkillsPath    string            // Skills 容器内路径（HomeDir 子路径）
    SessionPath   string            // 会话文件路径
}

type RuntimeManager struct {
    registry *Registry
    crypto   CryptoService
}

// ResolveConfig 通过适配器解析运行时配置
func (m *RuntimeManager) ResolveConfig(
    ctx context.Context,
    runtimeName string,
    executeTemplate string,
    runtimeConfig map[string]interface{},
    prompt string,
) (*ResolvedSandboxConfig, error) {
    adapter := m.registry.GetAdapter(runtimeName)
    if adapter == nil {
        return nil, fmt.Errorf("unknown runtime: %s", runtimeName)
    }

    // 1. 解密 API Key
    apiKey, _ := m.crypto.Decrypt(runtimeConfig["api_key_encrypted"].(string))

    // 2. 通过适配器渲染配置文件
    configJSON, _ := adapter.RenderConfig(runtimeConfig)

    // 3. 渲染 execute_template（通用，不依赖具体运行时）
    tmpl, _ := template.New("exec").Parse(executeTemplate)
    var buf strings.Builder
    tmpl.Execute(&buf, map[string]string{
        "model":   fmt.Sprintf("%v", runtimeConfig["model"]),
        "prompt":  prompt,
    })

    // 4. 通过适配器映射环境变量
    envs := adapter.EnvMapping(runtimeConfig, apiKey)

    return &ResolvedSandboxConfig{
        ExecCommand:   buf.String(),
        ContainerEnvs: envs,
        ConfigJSON:    configJSON,
        ConfigPath:    adapter.ConfigPath(),
        HomeDir:       adapter.HomeDir(),
        SkillsPath:    adapter.SkillsMountPath(),
        SessionPath:   adapter.SessionPath(),
    }, nil
}

// GetParser 获取输出解析器
func (m *RuntimeManager) GetParser(runtimeName string) OutputParser {
    return m.registry.GetParser(runtimeName)
}
```

**Worker 调用（运行时无关）**：

```go
// Worker 不再硬编码 opencode，通过适配器统一处理
resolved, _ := runtimeMgr.ResolveConfig(ctx, runtime.Name, runtime.ExecuteTemplate, agent.RuntimeConfig, issue.Prompt)
parser := runtimeMgr.GetParser(runtime.Name)

sandboxMgr.Create(ctx, SandboxConfig{
    HomeDir:        resolved.HomeDir,     // 运行时主目录（bind mount 目标）
    SkillsMountPath: resolved.SkillsPath, // 由适配器决定（HomeDir 子路径）
    // ...
})

// stdout 解析也通过适配器
for scanner.Scan() {
    if event := parser.ParseLine(scanner.Bytes()); event != nil {
        // 统一处理 ParsedEvent
    }
}
```

### NotificationChannelManager — 通知渠道管理器

`NotificationService` 中 WS 推送、浏览器通知、群聊系统消息、邮件通知四种渠道的触发逻辑硬编码在多个方法里，通过 `ChannelManager` 统一分发。

**设计原则**：

- 统一入口：`manager.Notify(event, payload)` → 自动分发到用户开通的渠道
- 新增渠道只需注册新 Channel，不改业务代码
- 用户通知偏好统一查询

**实现**：

```go
// internal/notification/channel_manager.go
package notification

import (
    "context"
)

type Event string

const (
    EventIssueAssigned     Event = "issue_assigned"
    EventIssueStatusChange Event = "issue_status_changed"
    EventAgentCompleted    Event = "agent_completed"
    EventMention           Event = "mention"
    EventNewDM             Event = "new_dm"
)

type NotifyPayload struct {
    UserID  uint
    Title   string
    Body    string
    Data    map[string]interface{}
    GroupID uint   // 群聊系统消息用
    Event   Event
}

// Channel 通知渠道接口
type Channel interface {
    Name() string
    Send(ctx context.Context, userID uint, payload *NotifyPayload) error
}

type ChannelManager struct {
    channels  []Channel
    userPrefs UserPreferenceService
}

func NewChannelManager(prefs UserPreferenceService) *ChannelManager {
    return &ChannelManager{userPrefs: prefs}
}

// Register 注册渠道
func (m *ChannelManager) Register(ch Channel) {
    m.channels = append(m.channels, ch)
}

// Notify 统一通知入口：查询用户偏好 → 按渠道分发
func (m *ChannelManager) Notify(ctx context.Context, payload *NotifyPayload) error {
    prefs := m.userPrefs.GetNotificationPrefs(ctx, payload.UserID)
    for _, ch := range m.channels {
        if prefs.IsEnabled(ch.Name(), string(payload.Event)) {
            ch.Send(ctx, payload.UserID, payload)
        }
    }
    return nil
}

// NotifyGroup 向群聊发送系统消息（不受用户偏好控制）
func (m *ChannelManager) NotifyGroup(ctx context.Context, groupID uint, message string) {
    // 直接写入 messages 表 type=system
}
```

**渠道注册**：

```go
// 初始化
chMgr := notification.NewChannelManager(userPrefService)
chMgr.Register(&WebSocketChannel{hub: wsHub})
chMgr.Register(&EmailChannel{smtp: smtpClient})
chMgr.Register(&BrowserChannel{hub: wsHub})
// 群聊系统消息通过 NotifyGroup 独立发送
```

### GitManager — Git 管理器

统一管理 Git 平台 API 和仓库操作，分为两个子接口：

| 子接口 | 职责 | 运行位置 | 平台相关性 |
|--------|------|----------|-----------|
| **GitPlatform** | 平台 REST API（Issue/PR/Repo） | Go 后端 Service 层 | 平台相关 |
| **GitOps** | 仓库操作（Clone/Fetch/Push/Commit） | Worker 沙箱内 | 平台无关 |

**设计原则**：

- `manager.Platform(platform)` → 返回对应平台 API 实现
- `manager.NewOps(containerID, workdir)` → 返回仓库操作实例（绑定容器）
- 业务代码不感知具体平台和底层 Git 实现
- Phase 2 可替换为 go-git 库实现，上层无感知

**接口定义**：

```go
// internal/git/manager.go
package git

import (
    "context"
    "fmt"
)

// GitPlatform 平台 REST API 接口（平台相关，运行在 Go 后端 Service 层）
type GitPlatform interface {
    CreateIssue(ctx context.Context, repo, title, body string, labels []string) (issueID string, err error)
    CreatePR(ctx context.Context, repo, title, head, base, body string) (prURL string, err error)
    GetRepoInfo(ctx context.Context, repo string) (*RepoInfo, error)
    ListBranches(ctx context.Context, repo string) ([]string, error)
}

// GitOps 仓库操作接口（平台无关，运行在 Worker 沙箱内）
type GitOps interface {
    IsRepo(ctx context.Context) bool
    Clone(ctx context.Context, repoURL, branch, dest string) error
    FetchAll(ctx context.Context) error
    Checkout(ctx context.Context, branch string) error
    Pull(ctx context.Context, branch string) error
    Commit(ctx context.Context, message string, author Author) (string, error)
    Push(ctx context.Context) error
}

// Author 提交者信息
type Author struct {
    Name  string
    Email string
}

// GitManager Git 管理器（统一入口）
type GitManager struct {
    platforms       map[string]GitPlatform
    defaultPlatform string
}

func NewGitManager() *GitManager {
    return &GitManager{
        platforms:       make(map[string]GitPlatform),
        defaultPlatform: "github",
    }
}

// Register 注册平台 API 实现
func (m *GitManager) Register(platform string, p GitPlatform) {
    m.platforms[platform] = p
}

// Platform 获取平台 API 实现（默认返回 github）
func (m *GitManager) Platform(platform string) (GitPlatform, error) {
    if platform == "" {
        platform = m.defaultPlatform
    }
    p, ok := m.platforms[platform]
    if !ok {
        return nil, fmt.Errorf("unsupported git platform: %s", platform)
    }
    return p, nil
}

// NewOps 为指定容器创建仓库操作实例
func (m *GitManager) NewOps(containerID, workdir string) GitOps {
    return NewContainerGitOps(containerID, workdir)
}
```

**ContainerGitOps — 容器内 Shell 实现**：

```go
// internal/git/container_ops.go
package git

import (
    "context"
    "fmt"
)

// ExecFunc 在容器内执行命令的函数签名
type ExecFunc func(ctx context.Context, containerID, cmd string) error

// ContainerGitOps 基于 execInContainer 的仓库操作实现
type ContainerGitOps struct {
    containerID string
    workdir     string
    exec        ExecFunc
}

func NewContainerGitOps(containerID, workdir string) *ContainerGitOps {
    return &ContainerGitOps{
        containerID: containerID,
        workdir:     workdir,
        exec:        execInContainer, // 注入 Worker 的容器执行函数
    }
}

func (g *ContainerGitOps) run(ctx context.Context, cmd string) error {
    return g.exec(ctx, g.containerID, fmt.Sprintf("cd %s && %s", g.workdir, cmd))
}

func (g *ContainerGitOps) IsRepo(ctx context.Context) bool {
    return g.exec(ctx, g.containerID, "test -d "+g.workdir+"/.git") == nil
}

func (g *ContainerGitOps) Clone(ctx context.Context, repoURL, branch, dest string) error {
    return g.exec(ctx, g.containerID,
        fmt.Sprintf("git clone --branch %s %s %s", branch, repoURL, dest))
}

func (g *ContainerGitOps) FetchAll(ctx context.Context) error {
    return g.run(ctx, "git fetch --all")
}

func (g *ContainerGitOps) Checkout(ctx context.Context, branch string) error {
    return g.run(ctx, "git checkout "+branch)
}

func (g *ContainerGitOps) Pull(ctx context.Context, branch string) error {
    return g.run(ctx, fmt.Sprintf("git checkout %s && git pull", branch))
}

func (g *ContainerGitOps) Commit(ctx context.Context, message string, author Author) (string, error) {
    err := g.run(ctx, fmt.Sprintf(
        `git add . && git commit -m "%s" --author=\"%s <%s>\"`,
        message, author.Name, author.Email))
    return "", err
}

func (g *ContainerGitOps) Push(ctx context.Context) error {
    return g.run(ctx, "git push")
}
```

**初始化**：

```go
gitMgr := git.NewGitManager()
gitMgr.Register("github", &GitHubPlatform{token: cfg.GitHubToken})
// Phase 2: gitMgr.Register("gitea", &GiteaPlatform{...})
// Phase 2: gitMgr.Register("gitlab", &GitLabPlatform{...})
```

### TokenManager — Token 配额管理器

在现有 `TokenTracker` 基础上升级，增加配额检查、周期归档、用量报告能力。

**设计原则**：

- 配额检查：`manager.CheckQuota(orgID)` → 超额时暂停 Agent 执行
- 周期归档：每天凌晨 Redis → MySQL 持久化
- 用量报告：按组织/项目/Agent/日期多维度查询

**实现**：

```go
// internal/token/token_manager.go
package token

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

type Usage struct {
    PromptTokens     int64
    CompletionTokens int64
    Source           string // "agent" | "opencode"
}

type DailyReport struct {
    Date             string
    PromptTokens     int64
    CompletionTokens int64
    TotalCost        float64
    BySource         map[string]Usage
}

type TokenManager struct {
    redis  *redis.Client
    quota  QuotaService
}

func NewTokenManager(rdb *redis.Client, quota QuotaService) *TokenManager {
    return &TokenManager{redis: rdb, quota: quota}
}

// Record 记录单次调用用量（按 Agent + 日期 + 来源聚合）
func (m *TokenManager) Record(ctx context.Context, agentID uint, usage *Usage) {
    key := fmt.Sprintf("tokens:agent:%d:date:%s", agentID, time.Now().Format("2006-01-02"))
    pipe := m.redis.Pipeline()
    pipe.HIncrBy(ctx, key, "prompt_tokens", usage.PromptTokens)
    pipe.HIncrBy(ctx, key, "completion_tokens", usage.CompletionTokens)
    pipe.HIncrBy(ctx, key, "source_"+usage.Source, 1)
    pipe.Expire(ctx, key, 7*24*time.Hour)
    pipe.Exec(ctx)
}

// CheckQuota 检查组织配额（超额返回 false）
func (m *TokenManager) CheckQuota(ctx context.Context, orgID uint) bool {
    limit := m.quota.GetMonthlyLimit(ctx, orgID)
    used := m.quota.GetMonthlyUsage(ctx, orgID)
    return used < limit
}

// GetDailyUsage 获取指定日期用量
func (m *TokenManager) GetDailyUsage(ctx context.Context, agentID uint, date string) (*DailyReport, error) {
    key := fmt.Sprintf("tokens:agent:%d:date:%s", agentID, date)
    result, err := m.redis.HGetAll(ctx, key).Result()
    if err != nil {
        return nil, err
    }
    report := &DailyReport{
        Date:     date,
        BySource: make(map[string]Usage),
    }
    // 解析 result 到 report...
    return report, nil
}

// Archive 归档到 MySQL（定时任务调用）
func (m *TokenManager) Archive(ctx context.Context, date string) error {
    // 扫描 tokens:agent:*:date:{date} → 写入 token_usage 表
    return nil
}
```

### anserAgent Tool 系统（Skill 与系统通信）

Skill 不只是 Markdown 文档，每个 Skill 声明一组可调用 Tool。anserAgent 调度 LLM 时，LLM 决定调用哪个 Tool → 执行对应的 Go handler → 操作数据库/群聊/通知。

**Skill 定义格式**（YAML frontmatter + Markdown）：

```markdown
---
name: issue-backlog
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

    // 注册 anserAgent 可调用的所有 Tool
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

**anserAgent Tool 与 opencode Tool 对比**：

| | anserAgent Tool | opencode Tool |
|------|---------|------------|
| 运行位置 | Go 后端进程 | Docker 沙箱内 |
| 操作对象 | 系统数据（Issue/消息/时间线） | 代码文件（read/write/bash） |
| 权限 | Casbin RBAC | 沙箱隔离 |
| 注册方式 | `registry.Register(name, handler)` | opencode 内置 |
| 典型调用 | `create_issue` / `send_message` | `read` / `write` / `bash` |

### `/backlog` 与 `/todo` 指令识别

anserAgent 在群聊中监听 WebSocket 消息，检测到 `/backlog` 或 `/todo` 指令时触发方案拆解流程。两者共享 anserAgent 编排逻辑，区别仅在于 Issue 创建时的初始状态：

```go
// internal/agent/command_handler.go
type CommandHandler struct {
    orchestrator *GroupOrchestrator
    parser       *BacklogParser
}

// HandleBacklog 统一处理 /backlog 和 /todo 指令
// initialStatus: "backlog" 或 "todo"
func (h *CommandHandler) HandleBacklog(msg *ws.Message, initialStatus string) {
    // ── 输入校验 ──
    var text string
    if strings.HasPrefix(msg.Content.Text, "/backlog") {
        text = strings.TrimSpace(strings.TrimPrefix(msg.Content.Text, "/backlog"))
    } else {
        text = strings.TrimSpace(strings.TrimPrefix(msg.Content.Text, "/todo"))
    }

    // 收集群聊历史上下文
    history := h.getRecentMessages(msg.GroupID, 50)

    // 校验 1: 指令不能为空（无上下文）
    if text == "" && len(history) == 0 {
        h.ws.Reply(msg, "❌ 需要描述需求或先进行群聊讨论，不能为空。")
        return
    }

    // 校验 2: 群聊上下文不足（防止误触发）
    if len(history) < 3 && text == "" {
        h.ws.Reply(msg, "❌ 群聊讨论内容不足，请先描述需求或补充更多讨论后再试。")
        return
    }

    // 校验 3: 调用 anserAgent 产出单条方案
    plan := h.orchestrator.GeneratePlan(ctx, msg.GroupID, history, text)

    // 校验 4: anserAgent 返回空方案
    if plan == nil || plan.Title == "" {
        h.ws.Reply(msg, "❌ 未能产出有效方案，请补充更多需求细节后重试。")
        return
    }

    // 拆解为 1 个 Issue（状态由 initialStatus 决定）
    issue := h.parser.ParseSingle(ctx, plan, msg.GroupID)
    issue.Status = initialStatus               // "backlog" 或 "todo"
    h.createIssue(ctx, issue)

    hint := ""
    if initialStatus == "backlog" {
        hint = "请到项目 backlog Tab 确认 Issue 细节，确认后点击 [转为 todo] 启动执行"
    } else {
        hint = "Issue 已直接创建为 todo 状态，分配给 Agent 后即可进入执行队列"
    }

    h.ws.Broadcast(msg.GroupID, ws.Message{
        Type: initialStatus,  // "backlog" 或 "todo"
        Content: map[string]interface{}{
            "issue": issue,
            "hint":  hint,
        },
    })
}
```

**`/todo` vs `/backlog` 对比**：

| 维度 | `/backlog` | `/todo` |
|------|-----------|--------|
| Agent 参与 | ✅ anserAgent 编排产出方案 | ✅ anserAgent 编排产出方案 |
| Issue 状态 | `backlog` | `todo`（跳过 backlog） |
| 人工确认 | 需确认后转为 todo | 无需确认，直接可执行 |
| 适用场景 | 需求模糊，需讨论出方案后再审 | 需求明确，希望快速推进到执行 |

### `/new` 指令 — 会话上下文隔离

自然人可在群聊中发送 `/new` 开启新会话。系统生成新的 `session_id`，后续 Agent 讨论仅感知该会话之后的消息历史：

```go
// internal/agent/command_handler.go — /new 指令处理
func (h *CommandHandler) HandleNewSession(msg *ws.Message) {
    if !strings.HasPrefix(msg.Content.Text, "/new") {
        return
    }

    // ① 生成新 session_id
    newSessionID := uuid.New().String()

    // ② 写入一条 new_session 系统消息，标记会话边界
    h.ws.Broadcast(msg.GroupID, ws.Message{
        Type: "new_session",
        Content: map[string]interface{}{
            "session_id": newSessionID,
            "by_user":    msg.Sender.Name,
        },
    })

    // ③ 后续该群组的消息自动携带 newSessionID
    h.sessionManager.SetCurrent(msg.GroupID, newSessionID)
}
```

**上下文窗口规则**：

- `GroupOrchestrator.InvokeAgent` 收集群聊历史时，以当前 `session_id` 为过滤条件，仅加载该会话内的消息
- 历史会话消息仍在客户端可见（滚动加载），但不进入 Agent 的 LLM 上下文窗口
- `/new` 不携带额外参数时，仅重置上下文；后跟文本时（如 `/new 下一阶段：部署上线`），该文本作为新会话的首条消息

```go
// internal/agent/group_discuss.go — 上下文加载适配 session 隔离
func (o *GroupOrchestrator) getRecentMessages(groupID uint, limit int) []*schema.Message {
    sessionID := o.sessionManager.GetCurrent(groupID)
    // 按 session_id 过滤，只取当前会话的消息
    return o.msgRepo.FindByGroupAndSession(groupID, sessionID, limit)
}
```

### @Agent 任务布置

群内 Agent 在讨论过程中，可通过 `@AgentName` 语法向群内其他 Agent 布置任务。anserAgent 在调度时解析消息中的 `@AgentName`，匹配群内成员 Agent，将任务注入被 @ Agent 的上下文：

```go
// internal/agent/mention_resolver.go — @Agent 解析
func (r *MentionResolver) Resolve(ctx context.Context, groupID uint, text string) ([]*AgentMention, error) {
    // ① 正则匹配 @AgentName（支持中文/英文 Agent 名）
    re := regexp.MustCompile(`@(\S+)`)
    matches := re.FindAllStringSubmatch(text, -1)

    // ② 查询群内 Agent 成员，匹配名称
    members := r.groupRepo.FindAgentMembers(groupID)
    var mentions []*AgentMention
    for _, m := range matches {
        for _, agent := range members {
            if agent.Name == m[1] {
                mentions = append(mentions, &AgentMention{
                    AgentID: agent.ID,
                    Name:    agent.Name,
                    Role:    agent.SystemPrompt,   // 角色定义
                })
            }
        }
    }
    return mentions, nil
}
```

**调度流程**：

1. Agent A 发言含 `@前端Agent 请实现登录表单组件`
2. `MentionResolver` 解析出 `@前端Agent`，匹配群内 Agent 成员
3. anserAgent 将任务描述 + Agent A 的上下文 + 被提及 Agent 的角色定义（`system_prompt` + 记忆系统）注入被提及 Agent 的调用链
4. 被提及 Agent 根据自身角色和 Skill 决定响应方式：直接回复方案、创建 Issue、或执行操作

```go
// internal/agent/group_discuss.go — @Agent 调度增强
func (o *GroupOrchestrator) InvokeWithMentions(
    ctx context.Context,
    agent *model.Agent,
    messages []*schema.Message,
    mentions []*AgentMention,
) {
    for _, mention := range mentions {
        targetAgent := o.agentRepo.FindByID(ctx, mention.AgentID)
        // 构建被 @ Agent 的上下文：角色定义 + 任务描述 + 当前讨论
        agentCtx := append([]*schema.Message{
            schema.SystemMessage(fmt.Sprintf(
                "你是 %s。%s 正在群聊中向你布置任务。",
                targetAgent.SystemPrompt, agent.Name,
            )),
        }, messages...)

        // 异步调度被 @ Agent 回复
        go o.InvokeAgent(ctx, targetAgent, agentCtx)
    }
}
```

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

系统 Token 消耗来自两个阶段，需要分别追踪后汇总：

| 阶段 | 谁调 LLM | 追踪方式 | 占比（典型） |
|------|---------|---------|------------|
| **anserAgent 调度** | Go 后端进程 | `TokenTracker.Record(agentID, usage, "agent")` | ~10-20% |
| **opencode 执行** | Docker 沙箱内 | 解析 stdout JSON + session 文件 | ~80-90% |

#### TokenTracker — 统一记录（区分来源）

```go
// internal/agent/token_tracker.go
type TokenTracker struct {
    redis *redis.Client
}

// TokenRecord 单次用量记录
type TokenRecord struct {
    Source           string // "agent" 或 "opencode"
    PromptTokens     int64
    CompletionTokens int64
}

// Record 记录单次调用用量（按 Agent + 日期 + 来源聚合）
func (t *TokenTracker) Record(agentID uint, usage *schema.TokenUsage, source string) {
    key := fmt.Sprintf("tokens:agent:%d:date:%s", agentID, time.Now().Format("2006-01-02"))
    t.redis.HIncrBy(ctx, key, "prompt_tokens", int64(usage.PromptTokens))
    t.redis.HIncrBy(ctx, key, "completion_tokens", int64(usage.CompletionTokens))
    t.redis.HIncrBy(ctx, key, "agent_prompt_tokens", func() int64 {
        if source == "agent" { return int64(usage.PromptTokens) }
        return 0
    }())
    t.redis.HIncrBy(ctx, key, "agent_completion_tokens", func() int64 {
        if source == "agent" { return int64(usage.CompletionTokens) }
        return 0
    }())
    t.redis.HIncrBy(ctx, key, "opencode_prompt_tokens", func() int64 {
        if source == "opencode" { return int64(usage.PromptTokens) }
        return 0
    }())
    t.redis.HIncrBy(ctx, key, "opencode_completion_tokens", func() int64 {
        if source == "opencode" { return int64(usage.CompletionTokens) }
        return 0
    }())
    t.redis.Expire(ctx, key, 30*24*time.Hour) // 保留 30 天
}

// GetDailyUsage 获取指定日期用量（含来源明细）
func (t *TokenTracker) GetDailyUsage(
    agentID uint, date string,
) (prompt, completion int64, err error) {
    key := fmt.Sprintf("tokens:agent:%d:date:%s", agentID, date)
    cmd := t.redis.HMGet(ctx, key, "prompt_tokens", "completion_tokens")
    // ...
}
```

#### anserAgent 调度阶段 — 回调记录（已有）

```go
// 群聊讨论 / /backlog / PromptOptimizer 中的 Eino 调用
resp, err := GetChatModel().Generate(ctx, chatInput,
    model.WithCallbacks(&callbacks.Handler{
        OnEnd: func(ctx context.Context, info *callbacks.RunInfo, output model.CallbackOutput) context.Context {
            o.tokenTracker.Record(agent.ID, output.TokenUsage, "agent")
            return ctx
        },
    }),
)
```

#### opencode 执行阶段 — 双通道采集

**通道 ① 实时：`opencode run --format json` stdout 解析**

```go
// internal/worker/executor.go — Worker 执行 opencode
func (w *Worker) executeWithTokenTracking(
    ctx context.Context,
    containerID string,
    cmd string,
    agentID uint,
) error {
    // opencode run 命令追加 --format json
    cmd += " --format json"
    exec, _ := w.docker.ContainerExecCreate(ctx, containerID, types.ExecConfig{
        Cmd:          []string{"sh", "-c", cmd},
        AttachStdout: true,
        AttachStderr: true,
    })
   
    hijack, _ := w.docker.ContainerExecAttach(ctx, exec.ID, types.ExecStartCheck{})
    defer hijack.Close()

    scanner := bufio.NewScanner(hijack.Reader)
    for scanner.Scan() {
        line := scanner.Bytes()
        
        // 尝试解析为 JSON 事件
        var event map[string]interface{}
        if err := json.Unmarshal(line, &event); err == nil {
            // 解析 token_usage 字段
            if usage, ok := event["token_usage"]; ok {
                tu := parseTokenUsage(usage)
                w.tokenTracker.Record(agentID, &schema.TokenUsage{
                    PromptTokens:     tu.InputTokens,
                    CompletionTokens: tu.OutputTokens,
                }, "opencode")
            }
            
            // 解析文本内容 → 推送到 issue_timeline
            if content, ok := event["content"]; ok {
                w.timelineRepo.Append(issueID, "agent", "opencode_output", "", "", fmt.Sprintf("%v", content))
            }
        } else {
            // 非 JSON 行 → 原有的 stdout 解析逻辑（兼容）
            w.parseStdoutLine(string(line), issueID)
        }
    }
    return nil
}

// parseTokenUsage 从 opencode JSON 事件中提取 token 用量
func parseTokenUsage(raw interface{}) *TokenUsageDetail {
    m, ok := raw.(map[string]interface{})
    if !ok { return nil }
    return &TokenUsageDetail{
        InputTokens:       toInt64(m["input_tokens"]),
        OutputTokens:      toInt64(m["output_tokens"]),
        CacheReadTokens:   toInt64(m["cache_read_input_tokens"]),
        CacheWriteTokens:  toInt64(m["cache_creation_input_tokens"]),
    }
}
```

**通道 ② 事后汇总：opencode session 文件解析**

opencode 在 `~/.local/share/opencode/sessions/` 下保存 JSONL 格式的会话文件，每条消息包含 `token_usage` 字段。Worker 在执行完成后从容器中提取：

```go
// internal/worker/session_parser.go — 执行完成后调用
func (w *Worker) collectSessionTokens(
    ctx context.Context,
    containerID string,
    agentID uint,
    issueID uint,
) error {
    // 从容器中读取最新 session 文件的 token 汇总
    exec, _ := w.docker.ContainerExecCreate(ctx, containerID, types.ExecConfig{
        Cmd: []string{"sh", "-c",
            "cat $(ls -t ~/.local/share/opencode/sessions/*.jsonl | head -1)"},
    })
    // ... 读取输出 ...

    var totalInput, totalOutput int64
    scanner := bufio.NewScanner(bytes.NewReader(output))
    for scanner.Scan() {
        var msg struct {
            TokenUsage *struct {
                InputTokens  int64 `json:"input_tokens"`
                OutputTokens int64 `json:"output_tokens"`
            } `json:"token_usage"`
        }
        if json.Unmarshal(scanner.Bytes(), &msg) == nil && msg.TokenUsage != nil {
            totalInput += msg.TokenUsage.InputTokens
            totalOutput += msg.TokenUsage.OutputTokens
        }
    }

    // 最终汇总写入（与实时采集去重：取 max 而非累加）
    w.tokenTracker.RecordFinal(agentID, issueID, &schema.TokenUsage{
        PromptTokens:     totalInput,
        CompletionTokens: totalOutput,
    }, "opencode")

    // 写入 agent_logs
    w.logRepo.Create(ctx, &model.AgentLog{
        AgentID: agentID,
        Action:  "token_summary",
        Input:   json.RawMessage(fmt.Sprintf(`{"source":"opencode","issue_id":%d}`, issueID)),
        Output:  json.RawMessage(fmt.Sprintf(
            `{"input_tokens":%d,"output_tokens":%d,"total_tokens":%d}`,
            totalInput, totalOutput, totalInput+totalOutput)),
        Status:  "success",
    })
    return nil
}
```

> **双通道去重策略**：实时通道在 opencode 执行过程中持续累加 token，事后通道在执行完成后读取 session 文件获得最终精确值。取 `max(实时, 事后)` 作为最终值，覆盖 Redis 中的 opencode 部分（通过 `RecordFinal` 实现）。这样即使实时通道部分 JSON 解析失败，事后通道也能兜底。

#### Token 总量公式

```
Agent 总 Token = anserAgent 调度 Token + opencode 执行 Token
              = (讨论 + /backlog + 提示词优化) + (编码 + 测试 + 修复 + PR)
```

| 来源 | 包含的 LLM 调用 | 触发位置 |
|------|---------------|---------|
| `agent` | 群聊 Agent 讨论、`/backlog` 方案拆解、`PromptOptimizer.Enhance()` | `anserAgent.Invoke` / `CommandHandler.HandleBacklog` |
| `opencode` | `opencode run` 全过程（读取代码、生成代码、运行测试、修复错误、commit） | `Worker.executeWithTokenTracking` |

> **LLM API Key 安全模型**：API Key 在 `agents.runtime_config.llm.api_key_encrypted` 中以 AES-256 加密存储；Agent 执行时 Worker 解密后通过环境变量注入 Docker 沙箱，不写入容器文件系统。

#### Token 用量 API 暴露

提供按 Agent/按组织维度的 Token 用量查询接口，用于 Dashboard 成本展示：

```go
// internal/handler/token_handler.go
type TokenUsageResponse struct {
    AgentID                 uint    `json:"agent_id"`
    AgentName               string  `json:"agent_name"`
    Date                    string  `json:"date"`              // "2026-05-14"
    PromptTokens            int64   `json:"prompt_tokens"`
    CompletionTokens        int64   `json:"completion_tokens"`
    TotalTokens             int64   `json:"total_tokens"`
    AgentPromptTokens        int64   `json:"agent_prompt_tokens"`        // anserAgent 调度阶段
    AgentCompletionTokens    int64   `json:"agent_completion_tokens"`
    OpencodePromptTokens    int64   `json:"opencode_prompt_tokens"`    // opencode 执行阶段
    OpencodeCompletionTokens int64  `json:"opencode_completion_tokens"`
    EstimatedCostUSD        float64 `json:"estimated_cost_usd"` // 按 provider 单价估算
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

---

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
│  │  ├── NamedVol: workspace-{projectID}:/workspace  │  │
│  │  │   (代码持久化，容器销毁后不丢失，避免重复 clone)│  │
│  │  ├── BindMount: project-runtime→home (rw)│  │
│  │  │   /var/lib/.../projects/{id}/runtime/ → /home/sandbox/.opencode│  │
│  │  │   (项目级运行时数据：Skills + 配置 + 插件，读写)│  │
│  │  └── AutoRemove: false (保障重启后可恢复) │  │
│  │                                │  │
│  │  Step 3: 注入运行时配置         │  │
│  │  ├── 读取 Agent runtime_config  │  │
│  │  ├── 解密 API Key → 环境变量    │  │
│  │  ├── 写入 config.json 到容器    │  │
│  │  ├── 写入 Runtime 默认 Skills   │  │
│  │  │   (opencode→anser-coder)│  │
│  │  └── 写入 Agent 绑定 Skills     │  │
│  │                                │  │
│  │  Step 4: 执行（编码闭环 + 消息互通）   │  │
│  │  ├── /workspace/repo 已存在?             │  │
│  │  │   ├── 是 → git fetch（增量更新）     │  │
│  │  │   └── 否 → git clone（首次拉取）     │  │
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

暂停/恢复操作直接操作 Docker 容器，但 Asynq 中的任务**已经出队**（dequeue），不再受 Redis 队列管理。为保证状态一致性，Worker 采用 **Redis Pub/Sub 事件驱动** 机制（替代 DB 轮询）：

```go
// internal/sandbox/control.go — 暂停时持久化任务上下文
func (w *Worker) PauseIssue(issueID uint) error {
    // ... ContainerPause ...
    
    // 关键：写入 issue_timeline 记录当前执行进度
    w.issueRepo.UpdateStatus(issueID, "paused")
    
    // 将当前 opencode 进度写入 agent_logs（标记暂停点）
    w.logRepo.Create(ctx, &model.AgentLog{
        IssueID: issueID,
        Action:  "paused",
        Status:  "running",
        Output:  json.RawMessage(`{"event":"paused","sandbox_id":"` + containerID + `"}`),
    })

    // 发布暂停事件（通知 Worker goroutine 挂起）
    w.redis.Publish(ctx, "issue:control:"+strconv.Itoa(int(issueID)),
        `{"action":"pause","issue_id":`+strconv.Itoa(int(issueID))+"}")
    return nil
}
```

**Worker 事件驱动状态监听**：Worker 在执行循环中通过 Redis Pub/Sub 订阅 Issue 控制频道，收到控制事件时立即响应（替代原先 10s DB 轮询 + 3s 暂停轮询）：

```go
// internal/worker/executor.go — Worker 事件驱动控制
func (w *Worker) executeWithControl(ctx context.Context, issueID uint, runFn func() error) error {
    // 订阅 Issue 控制频道
    controlCh := "issue:control:" + strconv.Itoa(int(issueID))
    sub := w.redis.Subscribe(ctx, controlCh)
    defer sub.Close()
    ch := sub.Channel()

    // 执行业务逻辑（捕获 stdout 流等）
    done := make(chan error, 1)
    go func() { done <- runFn() }()

    for {
        select {
        case msg := <-ch:
            var ctrl struct{ Action string `json:"action"` }
            json.Unmarshal([]byte(msg.Payload), &ctrl)
            switch ctrl.Action {
            case "pause":
                // Docker 暂停已在 PauseIssue 中完成
                // Worker goroutine 进入等待，直到收到 resume 或 stop
                if err := w.waitForResume(ctx, sub, issueID); err != nil {
                    return err
                }
            case "stop":
                return errors.New("issue was stopped externally")
            }
        case err := <-done:
            return err // 业务逻辑正常结束
        }
    }
}

// waitForResume 等待恢复信号（事件驱动，不轮询 DB）
func (w *Worker) waitForResume(ctx context.Context, sub *redis.PubSub, issueID uint) error {
    ch := sub.Channel()
    for {
        select {
        case msg := <-ch:
            var ctrl struct{ Action string `json:"action"` }
            json.Unmarshal([]byte(msg.Payload), &ctrl)
            switch ctrl.Action {
            case "resume":
                return nil // 恢复执行
            case "stop":
                return errors.New("issue was stopped during pause")
            }
        case <-ctx.Done():
            return ctx.Err()
        }
    }
}
```

> **优势对比**：
>
> | 方案 | 响应延迟 | DB 压力 | Worker 资源占用 |
> |------|---------|---------|----------------|
> | DB 轮询（旧） | 3-10s | 每 Issue 3-10次/s | CPU 占用（循环轮询） |
> | Redis Pub/Sub（新） | <100ms | 0 | 零开销（channel 阻塞等待） |

**恢复/停止操作同样发布事件**：

```go
func (w *Worker) ResumeIssue(issueID uint) error {
    // ... ContainerUnpause + 状态更新 ...
    w.redis.Publish(ctx, "issue:control:"+strconv.Itoa(int(issueID)),
        `{"action":"resume","issue_id":`+strconv.Itoa(int(issueID))+"}")
    return nil
}

func (w *Worker) StopIssue(issueID uint) error {
    // ... ContainerStop + Remove + 状态更新 ...
    w.redis.Publish(ctx, "issue:control:"+strconv.Itoa(int(issueID)),
        `{"action":"stop","issue_id":`+strconv.Itoa(int(issueID))+"}")
    return nil
}
```

**Worker 重启后恢复**：Worker 进程重启时，需恢复两类异常状态的 Issue：

```
Worker 启动
    │
    ├── 1. RecoverRunningIssues — 恢复 in_progress
    │       │
    │       ▼
    │   扫描 issues WHERE status='in_progress' AND sandbox_container_id IS NOT NULL
    │       │
    │       ├── 容器 running → 重新 Attach stdout 流 + 订阅控制频道，继续处理
    │       │
    │       ├── 容器 stopped（Docker 重启导致容器停止）
    │       │       ├── 检查 opencode 退出码（docker inspect .State.ExitCode）
    │       │       │   ├── exit 0 → 成功但未来得及提交 → 执行 commit/PR 流程 → in_review
    │       │       │   └── exit 非0 / unknown → 标记失败，Issue → todo，记录异常时间线
    │       │       └── 更新 Issue 状态 + 通知前端
    │       │
    │       └── 容器已销毁 → Issue → backlog + 记录异常时间线
    │
    └── 2. RecoverPausedIssues — 恢复 paused（原有逻辑）
            │
            ▼
        扫描 issues WHERE status='paused' AND sandbox_container_id IS NOT NULL
            │
            ├── 容器存活（paused）→ 恢复 Attach + 订阅控制频道
            └── 容器已销毁 → Issue → backlog + 记录异常时间线
```

```go
// internal/worker/recovery.go

// RecoverRunningIssues 恢复服务器重启前正在执行的 Issue
// 场景：服务器重启 → Docker 守护进程重启 → 容器停止（AutoRemove=false，容器还在）
func (w *Worker) RecoverRunningIssues(ctx context.Context) {
    runningIssues, _ := w.issueRepo.FindByStatus(ctx, "in_progress")
    for _, issue := range runningIssues {
        if issue.SandboxContainerID == "" {
            // 异常：in_progress 但没有容器 → 回退到 todo
            w.issueRepo.UpdateStatus(issue.ID, "todo")
            w.timelineRepo.Append(issue.ID, "system", "status_change",
                "in_progress", "todo", "Worker 重启后未找到沙箱容器，已自动回退等待重试")
            continue
        }

        info, err := w.cli.ContainerInspect(ctx, issue.SandboxContainerID)
        if err != nil {
            // 容器已销毁
            w.issueRepo.UpdateStatus(issue.ID, "todo")
            w.issueRepo.ClearContainerID(issue.ID)
            w.timelineRepo.Append(issue.ID, "system", "status_change",
                "in_progress", "todo", "沙箱容器已不存在，已自动回退等待重试")
            continue
        }

        switch info.State.Status {
        case "running":
            // 容器仍在运行（Docker 先于 Worker 重启完成）
            // → 重新 Attach stdout 流 + 订阅控制频道
            go w.attachAndProcess(ctx, issue)
            w.timelineRepo.Append(issue.ID, "system", "system_note",
                "", "", "Worker 重启，已重新连接到运行中的沙箱")

        case "exited", "dead":
            // 容器已退出（Docker 重启导致容器停止）
            exitCode := info.State.ExitCode
            if exitCode == 0 {
                // opencode 成功退出但未提交 → 执行 commit/PR 流程
                go w.commitAndCreatePR(ctx, issue)
                w.timelineRepo.Append(issue.ID, "system", "system_note",
                    "", "", "Worker 重启，检测到沙箱已完成编码，正在提交")
            } else {
                // 执行失败 → 回退到 todo，等待重试
                w.issueRepo.UpdateStatus(issue.ID, "todo")
                w.issueRepo.ClearContainerID(issue.ID)
                w.timelineRepo.Append(issue.ID, "system", "status_change",
                    "in_progress", "todo",
                    fmt.Sprintf("沙箱执行失败（退出码 %d），已自动回退等待重试", exitCode))
            }

        case "paused":
            // 容器被 Docker 暂停但 Issue 状态是 in_progress（状态不一致）
            // → 归并到 paused 恢复流程
            w.issueRepo.UpdateStatus(issue.ID, "paused")
            w.timelineRepo.Append(issue.ID, "system", "system_note",
                "", "", "检测到状态不一致（容器 paused 但 Issue in_progress），已修正为 paused")
        }
    }
}

// RecoverPausedIssues 恢复暂停的 Issue（原有逻辑）
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
        // 容器存活 → 重新 Attach + 订阅控制频道，继续事件驱动监听
    }
}
```

> **设计要点**：
> - **恢复优先级**：先恢复 `in_progress`（可能需要继续执行），再恢复 `paused`（等待人工操作）
> - **幂等安全**：所有恢复操作基于 DB 中的 `sandbox_container_id`，重复执行不会产生副作用
> - **容器保护**：`AutoRemove: false` 确保容器不会随 Docker 重启被自动清理
> - **暂停/恢复的可靠性**依赖于 `sandbox_container_id` 持久化到 DB + Redis Pub/Sub 事件驱动。只要容器未被销毁，Worker 重启后总能重新 Attach、订阅控制频道并恢复上下文。控制频道 `issue:control:{id}` 仅在有 Worker 消费时存在，无额外资源开销。

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

### 7.2.1 数据持久化保障

服务器重启、Worker 进程崩溃等异常场景下的数据保护：

| 数据类型 | 存储位置 | 重启后 | 保障机制 |
|---------|---------|--------|----------|
| Issue / Agent / 配置 / 日志 | MySQL | ✅ 不丢失 | InnoDB WAL + binlog，ACID 事务保证 |
| 代码仓库（/workspace） | Docker Named Volume | ✅ 不丢失 | 生命周期独立于容器，Docker 重启后卷自动恢复 |
| 运行时数据（Skills/插件/配置） | 宿主机 bind mount | ✅ 不丢失 | 本地文件系统，与容器/进程生命周期解耦 |
| Asynq 未消费任务 | Redis（AOF） | ✅ 不丢失 | `appendonly yes` + `fsync everysec`，重启后自动重放 |
| Asynq 已出队未完成任务 | MySQL（Issue 状态） | ✅ 不丢失 | 任务出队时 Issue 已标记 `in_progress`，Worker 重启时 `RecoverRunningIssues` 兜底 |
| 容器内进程内存状态 | 容器内存 | ❌ 丢失 | 容器停止 → opencode 进程上下文丢失，回退到 `todo` 重试 |

**部署要求**（config.yaml 或部署文档注明）：

```yaml
# Redis 必须开启 AOF 持久化，否则 Asynq 未消费任务在 Redis 重启后丢失
redis:
  host: 127.0.0.1
  port: 6379
  # ↓ 部署时需在 redis.conf 中配置：
  # appendonly yes
  # appendfsync everysec
```

> **容器重启策略**：沙箱容器设置 `AutoRemove: false`，**不设置** `RestartPolicy`。原因：容器停止后由 Worker 的 `RecoverRunningIssues` 统一处理，避免容器自启动后无 Worker 监听导致 stdout 丢失。

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

**opencode 配置注入流程**（已由 RuntimeAdapter 接口替代，此为旧代码参考）：

> 以下硬编码逻辑已迁移至 `OpenCodeAdapter.RenderConfig()` + `OpenCodeAdapter.EnvMapping()`。新运行时只需实现 `RuntimeAdapter` 接口，无需修改 Worker。

```go
// [已废弃] 旧版硬编码注入，保留作为参考
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

    // 项目级运行时数据目录（从全局模板复制而来）
    // 路径格式: /var/lib/anserflow/projects/{projectID}/runtime/
    projectRuntimeDir := cfg.ProjectRuntimeDir  // 来自 projects.runtime_data_dir
    homeDir := cfg.Runtime.HomeDir              // 由 RuntimeAdapter.HomeDir() 提供

    // Named Volume：按项目隔离，容器销毁后代码保留，避免大仓库重复 clone
    workspaceVol := fmt.Sprintf("anserflow-workspace-%d", cfg.ProjectID)

    resp, _ := cli.ContainerCreate(ctx, &container.Config{
        Image: image,
        Env:   buildEnvVars(cfg),
    }, &container.HostConfig{
        Memory:     512 * 1024 * 1024,
        NanoCPUs:   2 * 1e9,
        AutoRemove: false,
        // bind mount: 项目级 runtime 目录 → 容器内运行时主目录（读写）
        // 目录结构（opencode 示例）:
        //   projectRuntimeDir/skills/   → /home/sandbox/.opencode/skills/
        //   projectRuntimeDir/config.json → /home/sandbox/.opencode/config.json
        //   projectRuntimeDir/plugins/  → /home/sandbox/.opencode/plugins/
        Binds: []string{
            projectRuntimeDir + ":" + homeDir + ":rw",
        },
        Mounts: []mount.Mount{
            {
                Type:   mount.TypeVolume,
                Source: workspaceVol,
                Target: "/workspace",
            },
        },
    }, nil, nil, "")

    cli.ContainerStart(ctx, resp.ID, container.StartOptions{})

    // 首次：git clone；后续：git fetch 增量更新
    prepareRepo(ctx, resp.ID, cfg)

    // 注入运行时配置 + 执行（写入 homeDir 子路径的 config.json）
    injectRuntimeConfig(ctx, resp.ID, cfg.Runtime)
    // 根据 execute_template 渲染命令: "opencode run \"...\" --model ..."
    cmd := renderTemplate(cfg.Runtime.ExecuteTemplate, cfg.Runtime.Config, cfg.TaskPrompt)
    execInContainer(ctx, resp.ID, cmd)
    return resp.ID, nil
}

// prepareRepo 首次 clone 或增量 fetch（通过 GitOps 抽象）
func prepareRepo(ctx context.Context, containerID string, cfg SandboxConfig) {
    ops := gitMgr.NewOps(containerID, "/workspace/repo")
    if ops.IsRepo(ctx) {
        ops.FetchAll(ctx)
        ops.Checkout(ctx, cfg.Branch)
        ops.Pull(ctx, cfg.Branch)
    } else {
        ops.Clone(ctx, cfg.RepoURL, cfg.Branch, "/workspace/repo")
    }
}

// 在 Issue→done 时销毁容器（Named Volume 保留，后续 Issue 可复用）
func destroySandbox(ctx context.Context, containerID string) {
    cli, _ := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
    cli.ContainerRemove(ctx, containerID, container.RemoveOptions{Force: true})
}

// 在项目删除时清理 Named Volume
func destroyWorkspaceVolume(ctx context.Context, projectID uint) error {
    cli, _ := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
    return cli.VolumeRemove(ctx, fmt.Sprintf("anserflow-workspace-%d", projectID), true)
}
}```

### 7.3.1 运行时数据目录 — 三层架构

```
宿主机目录结构（由 config.yaml sandbox.runtime_data_dir 指定根目录）

/var/lib/anserflow/                                ← runtime_data_dir
├── runtimes/                                      ← Layer 1: 全局模板（管理员维护）
│   └── opencode/                                  ← 对应 runtimes.name
│       ├── skills/                                ← 默认 Skills（如 anser-coder）
│       │   └── anser-coder/SKILL.md
│       ├── config.json                            ← 默认配置模板
│       └── plugins/                               ← 预装插件（如 opencode-agent-memory）
│
├── projects/                                      ← Layer 2: 项目实例
│   └── 42/                                        ← projects.id
│       └── runtime/                               ← projects.runtime_data_dir
│           ├── skills/                            ← 项目级 Skills（可增删）
│           ├── config.json                        ← 项目级配置（可自定义）
│           └── plugins/                           ← 项目级插件（可安装）
│
沙箱容器                                          ← Layer 3: bind mount
└── /home/sandbox/.opencode/                       ← RuntimeAdapter.HomeDir()
    ├── skills/                                    ← ← bind mount 自 projects/42/runtime/
    ├── config.json
    └── plugins/
```

```go
// internal/sandbox/runtime_init.go

// initProjectRuntime 项目创建时调用：从全局模板复制到项目实例目录
// 1. runtime_data_dir 为空 → 首次初始化，从全局模板复制
// 2. runtime_data_dir 已存在 → 跳过（幂等）
func initProjectRuntime(ctx context.Context, projectID uint, runtimeName string) (string, error) {
    dataDir := viper.GetString("sandbox.runtime_data_dir") // /var/lib/anserflow

    projectDir := filepath.Join(dataDir, "projects", strconv.Itoa(int(projectID)), "runtime")

    // 幂等：已初始化则跳过
    if _, err := os.Stat(projectDir); err == nil {
        return projectDir, nil
    }

    // 从全局模板复制（递归）
    templateDir := filepath.Join(dataDir, "runtimes", runtimeName)
    if err := copyDir(templateDir, projectDir); err != nil {
        return "", fmt.Errorf("复制运行时模板失败: %w", err)
    }

    // 确保权限正确（容器内 sandbox 用户 uid=1000）
    filepath.Walk(projectDir, func(path string, info os.FileInfo, err error) error {
        os.Chown(path, 1000, 1000)
        return nil
    })

    // 回写 projects.runtime_data_dir
    projectRepo.UpdateRuntimeDir(ctx, projectID, projectDir)

    return projectDir, nil
}

// copyDir 递归复制目录
func copyDir(src, dst string) error {
    os.MkdirAll(dst, 0755)
    filepath.Walk(src, func(path string, info os.FileInfo, err error) error {
        rel, _ := filepath.Rel(src, path)
        target := filepath.Join(dst, rel)
        if info.IsDir() {
            os.MkdirAll(target, info.Mode())
        } else {
            data, _ := os.ReadFile(path)
            os.WriteFile(target, data, info.Mode())
        }
        return nil
    })
    return nil
}
```

**调用时机**：项目创建时（ProjectService.Create）或首次创建沙箱时（懒初始化）

```go
// 项目创建时调用
project, _ := projectRepo.Create(ctx, ...)
projectRuntimeDir, _ := initProjectRuntime(ctx, project.ID, defaultRuntimeName)
// 后续 createSandbox 使用 projectRuntimeDir
```

**清理**：

```go
// 在项目删除时清理 Named Volume + 运行时数据目录
func destroyWorkspaceVolume(ctx context.Context, projectID uint) error {
    cli, _ := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
    cli.VolumeRemove(ctx, fmt.Sprintf("anserflow-workspace-%d", projectID), true)

    // 同时清理项目运行时数据目录
    dataDir := viper.GetString("sandbox.runtime_data_dir")
    projectDir := filepath.Join(dataDir, "projects", strconv.Itoa(int(projectID)))
    return os.RemoveAll(projectDir)
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

后期支持 Gitea / GitLab / Gitee 等平台，通过 `GitPlatform` + `GitOps` 双接口抽象：

```
┌──────────────────────────────────────────────────────────┐
│  internal/git/                                            │
│  ├── manager.go           GitManager 统一入口             │
│  ├── platform.go          GitPlatform 接口定义             │
│  ├── ops.go               GitOps 接口 + Author 定义       │
│  ├── container_ops.go     容器 Shell 实现（当前）          │
│  ├── gogit_ops.go         go-git 库实现（Phase 2 可选）    │
│  ├── github.go            GitHub GitPlatform 实现          │
│  ├── gitea.go             Gitea 实现（Phase 2）            │
│  └── gitlab.go            GitLab 实现（Phase 2）           │
└──────────────────────────────────────────────────────────┘
```

```go
// internal/git/platform.go — 平台无关接口
type GitPlatform interface {
    CreateIssue(ctx, repo, title, body string, labels []string) (issueID string, err error)
    CreatePR(ctx, repo, title, head, base, body string) (prURL string, err error)
    GetRepoInfo(ctx, repo string) (*RepoInfo, error)
    ListRepos(ctx) ([]RepoInfo, error)
}
```

扩展策略：

| 层 | 扩展方式 | 说明 |
|----|---------|------|
| **GitPlatform** | 新增平台实现 | 每个 REST API SDK 封装为一个 `GitPlatform` 实现 |
| **GitOps** | 替换底层实现 | `ContainerGitOps`（Shell）→ `GoGitOps`（go-git 库），上层无需改动 |

| 平台 | Go SDK | 备注 |
|------|--------|------|
| **GitHub** | `go-github/v68` | 已实现 |
| **Gitea** | `code.gitea.io/sdk/gitea` | API 兼容 GitHub，迁移成本低 |
| **GitLab** | `github.com/xanzy/go-gitlab` | API 结构不同，独立实现 |
| **Gitee** | REST API（无官方 Go SDK） | 可基于 `net/http` 封装 |

> 当前阶段只闭环 GitHub 集成：`git_platform` 字段保留，但本轮验收默认取值为 `github`。Gitea / GitLab / Gitea Platform 进入后续独立任务，不作为当前计划阻塞项。仓库操作当前使用 `ContainerGitOps`（Shell），Phase 2 可选替换为 `GoGitOps`（go-git 库直接操作）。


---

## 八、Skills 技能系统

### 8.1 两种导入方式

```
Skills 数据表：

┌──────────────────────────────────────────────┐
│ skills                                        │
├──────────────────────────────────────────────┤
│ source_type:  'manual' | 'zip'                │
│                                             │
│ 手动模式 (source_type='manual'):              │
│   definition: TEXT  ← 直接在 UI 编辑 Markdown │
│                                             │
│ ZIP 模式 (source_type='zip'):                 │
│   zip_hash: VARCHAR(64)   ← ZIP 包 SHA256    │
│   file_tree: JSON          ← 文件树快照       │
│   definition: TEXT         ← 解压后的 SKILL.md│
└──────────────────────────────────────────────┘
```

### 8.2 ZIP 包格式

```
my-skill.zip
├── SKILL.md              # 必须：Skill 定义
│                         #   ---
│                         #   name: my-skill
│                         #   description: ...
│                         #   ---
│                         #   # Skill 正文
├── agents/               # 可选：Agent 配置
│   └── openai.yaml
├── tools/                # 可选：辅助脚本
│   └── lint.sh
└── examples/             # 可选：示例
    └── sample.md
```

### 8.4 ZIP 导入完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│                         Web 端 (Next.js)                        │
├─────────────────────────────────────────────────────────────────┤
│  <input type="file" accept=".zip" onChange={handleImport} />    │
│       │                                                         │
│       ▼                                                         │
│  const formData = new FormData()                                │
│  formData.append('file', file)       // 文件对象                │
│  formData.append('name', name)       // Skill 名称（可选）      │
│  formData.append('enabled', 'true')  // 导入后默认启用          │
│       │                                                         │
│       ▼                                                         │
│  POST /api/orgs/:org_id/skills/import/zip                       │
│  Content-Type: multipart/form-data                              │
│  Body: { file: Blob, name?: string, enabled?: boolean }         │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      后端 (Go + Gin)                             │
├─────────────────────────────────────────────────────────────────┤
│  // POST /api/orgs/:org_id/skills/import/zip                    │
│  func (h *SkillHandler) ImportZip(c *gin.Context) {             │
│      orgID := c.Param("org_id")                                 │
│                                                                │
│      // 1. 校验文件大小上限 10MB                                 │
│      c.Request.Body = http.MaxBytesReader(c.Writer,             │
│          c.Request.Body, 10<<20)                                │
│                                                                │
│      // 2. 解析 multipart form                                  │
│      file, header, err := c.Request.FormFile("file")            │
│      if err != nil {                                            │
│          respondError(c, "ERR_INVALID_UPLOAD", err)             │
│          return                                                 │
│      }                                                          │
│      defer file.Close()                                         │
│                                                                │
│      // 3. 读取到内存（不落盘）                                  │
│      zipBytes, _ := io.ReadAll(file)                            │
│      zipHash := sha256Hex(zipBytes) // 存入 zip_hash 字段       │
│                                                                │
│      // 4. 内存中解压 ZIP                                       │
│      zipReader, _ := zip.NewReader(                             │
│          bytes.NewReader(zipBytes), int64(len(zipBytes)))       │
│                                                                │
│      var definition string                                      │
│      var fileTree []FileNode                                    │
│                                                                │
│      for _, f := range zipReader.File {                         │
│          // 跳过目录、跳过系统文件（__MACOSX 等）                │
│          if f.FileInfo().IsDir() ||                              │
│             strings.HasPrefix(f.Name, "__MACOSX/") {            │
│              continue                                           │
│          }                                                      │
│                                                                │
│          rc, _ := f.Open()                                      │
│          content, _ := io.ReadAll(rc)                           │
│          rc.Close()                                             │
│                                                                │
│          // 记录到文件树                                         │
│          fileTree = append(fileTree, FileNode{                  │
│              Path:    f.Name,                                   │
│              Size:    int64(len(content)),                      │
│          })                                                     │
│                                                                │
│          // 提取 SKILL.md                                       │
│          if f.Name == "SKILL.md" {                              │
│              definition = string(content)                       │
│          }                                                      │
│      }                                                          │
│                                                                │
│      // 5. 校验：必须有 SKILL.md                                │
│      if definition == "" {                                      │
│          respondError(c, "ERR_MISSING_SKILL_MD", nil)           │
│          return                                                 │
│      }                                                          │
│                                                                │
│      // 6. 解析 SKILL.md 头部元信息                              │
│      meta := parseSkillFrontMatter(definition)                  │
│      // meta.Name, meta.Description                             │
│                                                                │
│      // 7. 写入数据库                                            │
│      skill := &Skill{                                           │
│          OrgID:       &orgID,                                   │
│          Name:        meta.Name,                                │
│          Description: meta.Description,                         │
│          SourceType:  "zip",                                    │
│          Definition:  definition,                               │
│          ZipHash:     zipHash,                                  │
│          FileTree:    marshalJSON(fileTree),                    │
│          Enabled:     true,                                     │
│      }                                                          │
│      h.skillRepo.Create(skill)                                  │
│                                                                │
│      // 8. 返回结果                                              │
│      c.JSON(200, gin.H{                                         │
│          "id":   skill.ID,                                      │
│          "name": skill.Name,                                    │
│          "file_count": len(fileTree),                           │
│      })                                                         │
│  }                                                              │
└─────────────────────────────────────────────────────────────────┘
```

> **约束**：ZIP 文件全程在内存中处理，最大 10MB。前端在 `onChange` 中可做客户端预检（文件类型 + 大小），后端 `MaxBytesReader` 兜底。重复导入同一 ZIP（相同 SHA256）不自动去重，由用户判断是否覆盖更新。

### 8.3 启用控制

```
                 ┌─────────────────┐
                 │  Skill A (全局)  │
                 │  enabled: true   │────────── 全局开关
                 └────────┬────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
    ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
    │ Agent CEO │   │ Agent CTO │   │ Agent Dev │
    │ Skill A ✅ │   │ Skill A ✅ │   │ Skill A ❌ │── 单Agent开关
    └───────────┘   └───────────┘   └───────────┘
```

