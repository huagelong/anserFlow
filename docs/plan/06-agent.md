# AnserFlow - anserAgent 智能体系统

---

## 一、系统定位

anserAgent 是 AnserFlow 的通用智能体内核，基于 Eino（Go 原生 AI 框架）构建，替代原 eino-* Skills 硬编码编排，承担两大职责：

| 职责 | 场景 | 说明 |
|------|------|------|
| **调度编排** | 群聊讨论、/backlog 方案拆解、/todo 快速建任务 | Agent 参与 IM 群聊协作 |
| **执行编排** | 沙箱编码任务（Issue 执行） | Agent 在 Docker 沙箱内完成编码 |

```
┌─────────────────────────────────────────────────────┐
│  anserAgent                                          │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  Eino Graph（底层引擎）                        │   │
│  │  ├── ChatModel   LLM 调用                     │   │
│  │  ├── Tool        行动能力（Skills → Tools）    │   │
│  │  └── Callbacks   回调/日志/Token 追踪         │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌────────────┐  ┌────────────┐  ┌───────────────┐ │
│  │ 五层记忆    │  │ Skills 引擎 │  │ 深度分析引擎  │ │
│  │ (L0~L4)   │  │ (自改进)    │  │ (任务/方案)   │ │
│  └────────────┘  └────────────┘  └───────────────┘ │
└─────────────────────────────────────────────────────┘
```

---

## 二、目录结构

```
internal/agent/
├── core/                       # anserAgent 内核
│   ├── agent.go                # Agent 主循环（Eino Graph 驱动）
│   ├── config.go               # Agent 配置（模型/记忆/Skills 绑定）
│   └── context.go              # 上下文构建（System Prompt + 五层记忆注入）
├── memory/                     # 五层记忆系统
│   ├── manager.go              # MemoryManager（读/写/检索/归档）
│   ├── meta_rules.go           # L0 元规则加载
│   ├── insight_index.go        # L1 记忆索引（路由表）
│   ├── global_facts.go         # L2 全局事实（稳定知识）
│   ├── skill_sop.go            # L3 任务 Skills / SOPs（可复用流程）
│   ├── session_archive.go      # L4 会话归档（长程召回）
│   ├── retriever.go            # 记忆检索（关键词 + 向量相似度）
│   └── wiki_store.go           # Wiki 格式存储（Markdown 文件读写）
├── skill_engine/               # Skills 自改进引擎
│   ├── manager.go              # SkillManager（加载/注册/调度）
│   ├── generator.go            # Skill 自动生成器（结晶化）
│   ├── improver.go             # Skill 自改进（执行后评估 → 优化）
│   ├── validator.go            # Skill 校验器（过时/报错检测）
│   └── rules.go                # Skill 生成规则定义（R01~R05）
├── analysis/                   # 深度分析引擎
│   ├── task_analyzer.go        # 任务拆解 + 依赖分析
│   ├── code_reviewer.go        # 代码审查建议
│   └── plan_evaluator.go       # 方案对比评估
└── orchestrator/               # 编排器（调度 + 执行）
    ├── group_orchestrator.go   # 群聊编排（替代原 GroupOrchestrator）
    └── sandbox_orchestrator.go # 沙箱执行编排
```

---

## 三、五层记忆系统

### 3.1 分层架构

记忆在任务执行过程中持续沉淀，使 Agent 逐步形成稳定且高效的工作方式：

```
┌──────────────────────────────────────────────────┐
│  L0 — 元规则（Meta Rules）                        │
│  Agent 的基础行为规则和系统约束                     │
│  存储：L0-meta-rules.md（单文件）                  │
│  生命周期：永久，极少变                            │
│  谁写：人工定义                                    │
├──────────────────────────────────────────────────┤
│  L1 — 记忆索引（Insight Index）                    │
│  极简索引层，用于快速路由与召回                      │
│  存储：L1-index.md（路由表）                       │
│  生命周期：永久，随下层更新                         │
│  谁写：Agent 自动维护                              │
├──────────────────────────────────────────────────┤
│  L2 — 全局事实（Global Facts）                     │
│  在长期运行过程中积累的稳定知识                      │
│  存储：L2-facts/ 目录（多个 Markdown 文件）          │
│  生命周期：长期稳定，缓慢演进                        │
│  谁写：Agent 从执行中积累                           │
├──────────────────────────────────────────────────┤
│  L3 — 任务 Skills / SOPs                           │
│  完成特定任务类型的可复用流程                        │
│  存储：L3-skills/ 目录（可自改进）                  │
│  生命周期：中期，随实践优化                         │
│  谁写：Agent 自动生成 + 自改进                      │
├──────────────────────────────────────────────────┤
│  L4 — 会话归档（Session Archive）                  │
│  从已完成任务中提炼出的归档记录，用于长程召回         │
│  存储：L4-archive/ 目录（按月归档）                 │
│  生命周期：归档后只读                              │
│  谁写：任务完成后自动归档                           │
└──────────────────────────────────────────────────┘
```

### 3.2 项目级记忆目录

```
/var/lib/anserflow/projects/{project_id}/memory/
├── L0-meta-rules.md              # L0 元规则（单文件，简洁）
├── L1-index.md                    # L1 记忆索引（路由表）
├── L2-facts/                      # L2 全局事实
│   ├── README.md                  #   事实总览
│   ├── tech-stack.md              #   技术栈 & 依赖
│   ├── architecture.md            #   架构决策
│   ├── conventions.md             #   编码规范
│   └── api-contracts.md           #   API 约定
├── L3-skills/                     # L3 任务 Skills / SOPs
│   ├── README.md                  #   Skills 索引
│   ├── react-auth/
│   │   └── SKILL.md               #   可复用流程
│   ├── api-crud/
│   │   └── SKILL.md
│   └── ...
└── L4-archive/                    # L4 会话归档
    ├── README.md                  #   归档索引
    ├── 2026-05/
    │   ├── issue-42-auth-refactor.md
    │   ├── issue-58-token-refresh.md
    │   └── ...
    └── ...
```

### 3.3 各层内容示例

**L0 — 元规则**（`L0-meta-rules.md`）：

```markdown
# 元规则

## 行为约束
- 收到编码任务时，先阅读相关 L2 事实和 L3 Skills，再动手
- 不确定的 API 调用，先查 L2 事实中的 API 约定，不要猜测
- 每次执行完成后，检查是否有新的 L2 事实或 L3 Skill 需要更新

## 沟通风格
- 群聊讨论中简洁发言，每次不超过 3 段
- 发现方案风险时主动提出，不要默认执行

## 安全边界
- 永远不要删除其他 Agent 的代码，除非有明确指令
- 涉及数据库 schema 变更时，必须先讨论再执行
- 不在代码中硬编码密钥或 Token
```

**L1 — 记忆索引**（`L1-index.md`）：

```markdown
# 记忆索引

## 快速路由

| 关键词 | 路由目标 | 层级 |
|--------|---------|------|
| JWT / 认证 / Token | L2: api-contracts.md, L3: react-auth/ | L2+L3 |
| React / 前端 | L2: tech-stack.md | L2 |
| 数据库 / 迁移 | L2: architecture.md | L2 |
| CRUD / 增删改查 | L3: api-crud/ | L3 |
| 登录页 | L4: issue-42-auth-refactor.md | L4 |

## 检索策略
- 编码任务 → 先查 L3 Skills，无匹配则查 L2 事实
- 方案讨论 → 先查 L2 事实 + L4 归档中的历史决策
- 不确定 → L1 索引路由 → 按关键词定位具体文件
```

**L2 — 全局事实**（稳定知识，例如 `L2-facts/tech-stack.md`）：

```markdown
> 来源：项目初始化 + Agent 执行中持续更新
> 更新频率：低（月度级别）

# 技术栈

## 后端
- Go 1.24 + Gin + GORM
- 数据库：MySQL 8.0
- 缓存/队列：Redis 7.0（AOF 持久化）
- AI 框架：Eino（字节 CloudWeGo）

## 前端
- Next.js 14（SPA 静态导出）
- UI：shadcn/ui + Tailwind CSS
- 状态：Zustand + TanStack Query

## 沙箱
- Docker SDK for Go
- 运行时：opencode / hermes
```

**L3 — 任务 Skill/SOP**（可复用流程，例如 `L3-skills/react-auth/SKILL.md`）：

```markdown
---
name: react-auth
version: 2
success_rate: 0.92
use_count: 5
last_used: 2026-05-16
tags: [frontend, react, auth, jwt]
---

# React 认证模块 SOP

## 触发条件
需要实现登录/注册/Token 刷新的前端认证功能

## 标准流程
1. api.ts 添加 Token 拦截器（401 → 自动刷新）
2. use-auth.ts 实现认证状态（Zustand store）
3. ProtectedRoute 组件（路由守卫）
4. 登录页组件（表单 + Zod 校验）

## 已知陷阱
- JWT 过期需统一 401 拦截，不要每个请求单独处理
- bcrypt 在后端做，前端只传明文密码

## 改进历史
| 版本 | 日期 | 变更 |
|------|------|------|
| v1 | 2026-05-10 | 初始生成（从 Issue #42 提取） |
| v2 | 2026-05-16 | 修复：添加 Token 刷新逻辑（Issue #58 失败后改进） |
```

**L4 — 会话归档**（任务完成后提炼，例如 `L4-archive/2026-05/issue-42-auth-refactor.md`）：

```markdown
> 归档时间：2026-05-16
> 关联 Issue：#42
> 关联 Skill：react-auth (v2)

# Issue #42 — 登录模块重构

## 任务摘要
将 Session 认证迁移到 JWT + bcrypt

## 关键决策
- JWT 无状态，水平扩展友好
- bcrypt cost = 12

## 踩过的坑
- 前端未处理 401 → 需统一拦截器
- 时序攻击：需 ConstantTimeCompare

## 经验贡献
- 更新了 L3: react-auth（添加 Token 刷新步骤）
- 更新了 L2: api-contracts（添加 JWT 约定）
```

### 3.4 核心接口

```go
// memory/manager.go
type MemoryManager struct {
    store     WikiStore           // Markdown 文件读写
    retriever MemoryRetriever     // 检索器
}

// Recall 检索记忆（按层级路由）
func (m *MemoryManager) Recall(ctx context.Context, req RecallRequest) ([]Memory, error)

// Store 存储记忆（根据层级写入对应文件）
func (m *MemoryManager) Store(ctx context.Context, mem Memory) error

// SummarizeAndArchive 将会话总结归档到 L4
func (m *MemoryManager) SummarizeAndArchive(ctx context.Context, sessionID string) error

// UpdateIndex 同步更新 L1 索引
func (m *MemoryManager) UpdateIndex(ctx context.Context, projectID uint) error

type RecallRequest struct {
    AgentID   uint
    ProjectID uint
    Query     string            // 检索查询
    Layers    []MemoryLayer     // 指定检索层级
    Limit     int               // 返回条数
}

type Memory struct {
    ID        string
    Layer     MemoryLayer       // L0 / L1 / L2 / L3 / L4
    Content   string
    FilePath  string            // 来源 Markdown 文件路径
    Tags      []string
    Source    string            // session_id / issue_id / manual
    CreatedAt time.Time
}

type MemoryLayer string

const (
    LayerL0 MetaRules    MemoryLayer = "L0"  // 元规则
    LayerL1 InsightIndex MemoryLayer = "L1"  // 记忆索引
    LayerL2 GlobalFacts  MemoryLayer = "L2"  // 全局事实
    LayerL3 SkillSOP     MemoryLayer = "L3"  // 任务 Skills / SOPs
    LayerL4 SessionArchive MemoryLayer = "L4" // 会话归档
)
```

### 3.5 记忆读写时机

| 时机 | 操作 | 层级 |
|------|------|------|
| Agent 被调用 | Recall 相关记忆 | L1 路由 → L2 + L3 + L4 |
| Agent 调用中 | Store 中间结果到上下文 | 运行时内存（不持久化） |
| Agent 调用结束 | 回复内容记入会话 | 运行时内存 |
| /new 切换会话 | SummarizeAndArchive 当前会话 | L4（归档） |
| Issue 执行完成 | 执行经验总结写入文件 | L4（归档）+ L2（新事实） |
| 同类任务 ≥3 次 | MaybeCrystallize 生成新 SOP | L3（Skills） |
| Skill 执行失败 | Improve 失败 Skill | L3（Skills 自改进） |
| L2/L3/L4 更新后 | UpdateIndex 同步路由表 | L1（索引） |

### 3.6 更新频率与权限

| 层级 | 更新频率 | 谁能写 | 自动/人工 |
|------|---------|--------|----------|
| L0 元规则 | 极低（月度/季度） | 人工 | 人工 |
| L1 索引 | 跟随 L2-L4 更新 | Agent 自动 | 自动 |
| L2 全局事实 | 低（周度） | Agent 积累 + 人工审核 | 半自动 |
| L3 Skills | 中（每次执行后评估） | Agent 自动生成 + 自改进 | 自动 |
| L4 会话归档 | 高（每次任务完成） | Agent 自动归档 | 自动 |

---

## 四、Skills 自改进引擎

### 4.1 Skill 生命周期

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  定义    │───►│  加载    │───►│  执行    │───►│  评估    │───►│  改进    │
│ (创建)   │    │ (注册)   │    │ (调用)   │    │ (打分)   │    │ (重写)   │
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
     ▲                                              │              │
     └──────────────────── 结晶化（自动生成新 Skill）─┘              │
     ▲                                                             │
     └─────────────────── 改进后的 Skill 覆盖原版本 ─────────────────┘
```

### 4.2 Skill 自动生成规则

```go
// skill_engine/rules.go — Skill 自动生成规则定义

type SkillGenerationRule struct {
    Trigger         SkillTrigger   // 何时触发自动生成
    Strategy        GenStrategy    // 如何生成
    MinScore        float64        // 最低质量分（0-1）
    RequireApproval bool           // 是否需要人工审批
}

type SkillTrigger string

const (
    TriggerOnComplexTask   SkillTrigger = "on_complex_task"    // 复杂任务完成后
    TriggerOnRepeatedTask  SkillTrigger = "on_repeated_task"   // 同类任务 ≥3 次
    TriggerOnFailure       SkillTrigger = "on_failure"         // 执行失败后分析
    TriggerOnManual        SkillTrigger = "on_manual"          // 人工请求创建
)

type GenStrategy string

const (
    StrategyExtract       GenStrategy = "extract"              // 从执行日志提取模式
    StrategyFailure       GenStrategy = "failure_learning"     // 失败经验总结
    StrategyBestPractice  GenStrategy = "best_practice"        // 多次成功归纳最佳实践
)
```

**预定义自动生成规则**：

| 规则 ID | 触发条件 | 生成策略 | 说明 |
|---------|---------|---------|------|
| `R01` | 同类 Issue 执行 ≥3 次成功 | `best_practice` | 提取编码模式生成 Skill |
| `R02` | Issue 执行失败后重试成功 | `failure_learning` | 总结踩坑经验 |
| `R03` | 跨 ≥3 个文件的重构任务 | `extract` | 生成重构流程 Skill |
| `R04` | 单次执行 Token > 50k | `extract` | 高成本任务提取优化模式 |
| `R05` | 人工 `/skill save` 指令 | `extract` | 手动保存当前会话为 Skill |

### 4.3 Skill 结晶化（借鉴 GenericAgent）

```go
// skill_engine/generator.go

type SkillGenerator struct {
    chatModel model.ChatModel
    store     SkillStore
}

// Crystallize 将执行路径结晶为 Skill
func (g *SkillGenerator) Crystallize(ctx context.Context, record ExecutionRecord) (*Skill, error) {
    // 1. 检查是否满足结晶条件
    if !g.shouldCrystallize(record) {
        return nil, nil
    }

    // 2. 提取执行路径（从 agent_logs 中）
    path := g.extractExecutionPath(record)

    // 3. LLM 浓缩为 SOP（信息密度最大化）
    sop := g.condenseToSOP(ctx, path)

    // 4. 写入 L3 Skills 目录（Markdown 文件）
    skill := g.writeSkillFile(sop)

    // 5. 更新 L1 索引
    g.updateIndex(skill)

    return skill, nil
}

func (g *SkillGenerator) shouldCrystallize(record ExecutionRecord) bool {
    switch {
    case record.IsNewTaskType:
        return true
    case record.ToolCallCount >= 5:
        return true
    case record.TokenUsed > 30000:
        return true
    default:
        return false
    }
}
```

### 4.4 Skill 自改进

```go
// skill_engine/improver.go

type SkillImprover struct {
    chatModel model.ChatModel
    store     SkillStore
}

type ImproveRequest struct {
    SkillID    uint
    Execution  ExecutionRecord
    Error      *TaskError        // nil 表示成功但可优化
}

func (s *SkillImprover) Improve(ctx context.Context, req ImproveRequest) error {
    if !s.needsImprovement(req) {
        return nil
    }
    analysis := s.analyze(ctx, req)
    improved := s.generateImproved(ctx, req.SkillID, analysis)
    if err := s.validate(improved); err != nil {
        return err
    }
    return s.store.Save(ctx, improved)
}

func (s *SkillImprover) needsImprovement(req ImproveRequest) bool {
    switch {
    case req.Error != nil:
        return true                    // 执行报错 → 必须改进
    case req.Execution.IsOutdated():
        return true                    // 内容过时 → 改进
    case req.Execution.Score < 0.6:
        return true                    // 质量评分低 → 改进
    default:
        return false
    }
}
```

### 4.5 Skill 过时检测

| 维度 | 检测方式 | 示例 |
|------|---------|------|
| **API 版本** | 对比 Skill 引用的 SDK 版本与项目当前版本 | "go-github/v67" → 项目已升级到 v68 |
| **库依赖** | 检查 Skill 引用的库是否已废弃 | "request" → 项目已迁移到 "httpx" |
| **目录结构** | 检查 Skill 引用的路径是否仍存在 | "internal/handler/" → 已重构为 "internal/api/" |
| **编码规范** | 对比 Skill 中的规范与 L2 conventions | Skill 说 "var 命名" → 规范已改为 "camelCase" |
| **执行成功率** | 统计近 N 次执行的成功率 | 最近 5 次执行 3 次失败 → 标记为 broken |

---

## 五、Agent 主循环

### 5.1 核心接口

```go
// core/agent.go

type anserAgent struct {
    id         uint
    name       string
    role       string               // 角色人设
    chatModel  model.ChatModel      // Eino ChatModel
    memory     *memory.Manager      // 五层记忆
    skills     *skill_engine.Manager
    tools      *ToolRegistry        // Eino Tool 注册表
    analyzer   *analysis.Engine
}

func (a *anserAgent) Invoke(
    ctx context.Context,
    input AgentInput,
) (*AgentResponse, error) {
    // 1. 五层记忆注入
    memories := a.memory.Recall(ctx, RecallRequest{
        AgentID: a.id, Query: input.Query, Limit: 10,
    })

    // 2. 构建 System Prompt（L0 元规则 + 角色人设）
    systemPrompt := a.buildSystemPrompt(memories)

    // 3. 加载 L3 Skills → Eino Tools
    tools := a.skills.LoadAsTools(ctx, a.id)

    // 4. 组装消息
    messages := a.assembleMessages(systemPrompt, memories, input)

    // 5. 调用 Eino ChatModel
    resp, err := a.chatModel.Generate(ctx, messages,
        model.WithTools(tools),
        model.WithCallbacks(a.tokenCallback()),
    )
    if err != nil {
        return nil, err
    }

    // 6. 后处理
    a.memory.Store(ctx, Memory{
        Layer: LayerL4, Content: resp.Content, Tags: input.Tags,
    })

    // 7. Skill 结晶化评估（异步）
    go a.skills.MaybeCrystallize(ctx, input, resp)

    return a.parseResponse(resp), nil
}

type AgentInput struct {
    Query     string
    Context   []*schema.Message
    Mode      AgentMode           // orchestrate / execute
    IssueID   *uint
    SessionID string
    Tags      []string
}

type AgentMode string

const (
    ModeOrchestrate AgentMode = "orchestrate"  // 调度编排
    ModeExecute     AgentMode = "execute"      // 沙箱执行编排
)

type AgentResponse struct {
    Content    string
    ToolCalls  []ToolCallResult
    TokenUsage TokenUsageDetail
    MemStored  int
}
```

### 5.2 上下文构建（信息密度最大化）

```go
// core/context.go — 分层记忆注入，控制总 Token 预算

func (a *anserAgent) buildSystemPrompt(memories []Memory) string {
    var sb strings.Builder

    // L0 元规则（永远注入，~500 token）
    sb.WriteString(a.loadMetaRules())

    // L1 索引（路由表，~200 token）
    sb.WriteString(a.loadInsightIndex())

    // L2 全局事实（按相关性筛选，~2K token）
    facts := a.selectRelevantFacts(memories, 2000)
    sb.WriteString(formatFacts(facts))

    // L3 Skills（匹配到的 SOP，~3K token）
    skills := a.matchSkills(memories)
    sb.WriteString(formatSkills(skills))

    // L4 会话归档（最近相关的，~2K token）
    archives := a.searchArchives(memories, 2000)
    sb.WriteString(formatArchives(archives))

    // 总计约 ~8K token 系统提示，留 ~22K 给对话
    return sb.String()
}
```

### 5.3 调度编排模式

```go
// orchestrator/group_orchestrator.go

func (o *GroupOrchestrator) OnMessage(ctx context.Context, msg Message) {
    agent := o.agentRegistry.Get(msg.GroupID)

    input := AgentInput{
        Query:     msg.Content.Text,
        Context:   o.getSessionHistory(msg.GroupID, msg.SessionID),
        Mode:      ModeOrchestrate,
        SessionID: msg.SessionID,
    }

    resp, err := agent.Invoke(ctx, input)
    if err != nil {
        o.sendErrorMessage(msg.GroupID, err)
        return
    }

    o.persistAndBroadcast(msg.GroupID, resp.Content)

    // 会话结束时归档到 L4
    if o.isSessionEnding(resp) {
        o.memory.SummarizeAndArchive(ctx, msg.SessionID)
    }
}
```

### 5.4 沙箱执行编排模式

```go
// orchestrator/sandbox_orchestrator.go

func (s *SandboxOrchestrator) ExecuteIssue(ctx context.Context, issue *Issue) error {
    agent := s.agentRegistry.GetByID(issue.AgentID)

    // 加载项目级长期记忆（L2 + L3）
    memories := s.memory.Recall(ctx, RecallRequest{
        AgentID: issue.AgentID, ProjectID: issue.ProjectID,
        Query: issue.Title, Layers: []MemoryLayer{LayerL2, LayerL3},
    })

    input := AgentInput{
        Query:   issue.BuildTaskPrompt(),
        Mode:    ModeExecute,
        IssueID: &issue.ID,
        Tags:    s.extractTags(issue),
    }

    resp, err := agent.Invoke(ctx, input)
    if err != nil {
        // 执行失败 → 触发 Skill 自改进
        s.skillImprover.Improve(ctx, ImproveRequest{
            SkillID:   s.detectRelatedSkill(issue),
            Execution: resp.Record,
            Error:     wrapTaskError(err),
        })
        return err
    }

    // 执行成功 → 评估是否结晶为新 Skill
    s.skillGenerator.MaybeCrystallize(ctx, resp.Record)

    return nil
}
```

---

## 六、与其他模块的集成

```
┌───────────────────────────────────────────────────────┐
│  Go 后端（Gin + Asynq）                                │
│                                                        │
│  Hub.OnMessage()                                       │
│       │                                                │
│       ▼                                                │
│  anserAgent.Invoke(mode=orchestrate)                  │
│       │                                                │
│       ├── MemoryManager ──► memory/ (L0~L4 Markdown)  │
│       ├── SkillManager  ──► L3-skills/ + MySQL        │
│       └── Eino ChatModel ──► LLM API                  │
│                                                        │
│  Worker（Asynq）                                      │
│       │                                                │
│       ▼                                                │
│  anserAgent.Invoke(mode=execute)                      │
│       │                                                │
│       ├── MemoryManager ──► L2 事实 + L3 Skills       │
│       ├── SkillManager  ──► anser-coder 等            │
│       └── RuntimeAdapter ──► opencode / hermes 沙箱   │
└───────────────────────────────────────────────────────┘
```

| 模块 | anserAgent 关系 |
|------|----------------|
| **Eino** | 底层引擎，提供 ChatModel / Graph / Tool / Callbacks |
| **RuntimeAdapter** | 执行模式的行动层（opencode/hermes） |
| **MemoryManager** | 五层记忆读写，Markdown Wiki 文件存储 |
| **SkillManager** | Skills 加载、自动生成、自改进 |
| **GitManager** | 执行模式中调用 GitOps |
| **SandboxManager** | 执行模式中创建/管理沙箱容器 |
| **TokenManager** | 记录所有 LLM Token 用量 |
| **NotificationManager** | 调度模式中推送消息到 IM |

---

## 七、数据库表

### 7.1 agent_memories

```sql
CREATE TABLE agent_memories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    agent_id BIGINT NOT NULL,
    project_id BIGINT NOT NULL,
    org_id BIGINT NOT NULL,
    layer ENUM('L0','L1','L2','L3','L4') NOT NULL,
    title VARCHAR(256),
    content TEXT NOT NULL,                  -- 记忆内容摘要
    file_path VARCHAR(512),                 -- Markdown 文件路径（L2~L4）
    tags JSON,                              -- 标签
    source ENUM('auto_summary','skill_created','manual','execution') NOT NULL,
    session_id VARCHAR(64),
    issue_id BIGINT,
    relevance_score FLOAT DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_agent_project (agent_id, project_id),
    INDEX idx_project_layer (project_id, layer),
    FOREIGN KEY (agent_id) REFERENCES agents(id),
    FOREIGN KEY (project_id) REFERENCES projects(id)
);
```

### 7.2 skill_generation_logs

```sql
CREATE TABLE skill_generation_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    agent_id BIGINT NOT NULL,
    project_id BIGINT NOT NULL,
    rule_id VARCHAR(16) NOT NULL,           -- R01~R05
    trigger_type ENUM('on_complex_task','on_repeated_task','on_failure','on_manual') NOT NULL,
    strategy ENUM('extract','failure_learning','best_practice') NOT NULL,
    source_issue_id BIGINT,
    skill_name VARCHAR(128),
    skill_version INT DEFAULT 1,
    quality_score FLOAT,
    status ENUM('generated','approved','rejected','improved') DEFAULT 'generated',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (agent_id) REFERENCES agents(id),
    FOREIGN KEY (project_id) REFERENCES projects(id)
);
```

---

## 八、从 eino-* Skills 到 anserAgent 的迁移映射

原 eino-* Skills 的功能全部由 anserAgent 五层记忆系统接管：

| 原 Skill | 迁移到 | 说明 |
|----------|--------|------|
| ~~eino-discuss~~ | **L0 元规则** | 讨论行为约束内化为元规则 |
| ~~eino-backlog~~ | **L3 Skill** | /backlog 方案拆解作为可自改进的 SOP |
| ~~eino-optimizer~~ | **SkillImprover** | 提示词优化逻辑由自改进引擎接管 |
| ~~eino-planner~~ | **L3 Skill** | 任务编排 SOP，可自动生成 |
| anser-coder | 保留 | 沙箱执行规范，不变 |

---

## 九、当前阶段范围

| 功能 | Phase 1（当前） | Phase 2 |
|------|:---:|:---:|
| anserAgent 内核（Eino ChatModel） | ✅ | ✅ |
| 五层记忆（L0~L4）+ Wiki 文件存储 | ✅ | ✅ |
| Skill 自动生成（R01~R05） | ✅ | ✅ |
| Skill 过时检测 + 自改进 | ✅ | ✅ |
| 调度编排 + 沙箱执行编排 | ✅ | ✅ |
| 记忆语义检索（向量） | ❌ | ✅ |
| 多 Agent 并行执行编排 | ❌ | ✅ |
| Skill 版本回滚 | ❌ | ✅ |
