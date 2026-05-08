# Vibe Coding 任务计划

来源：`wiki/README.md`。本计划按“先简单后复杂”排列，每个任务都指向对应的 wiki 小文档；执行时建议一个任务对应一个 Issue/分支，先读文档，再写最小实现，最后按验收标准检查。

## 执行约定

- 每个任务只实现当前目标，不提前扩展相邻能力。
- 每个任务开始前先阅读“参考 wiki 小文档”。
- 每个任务完成后补充或更新必要测试，至少跑通当前任务的核心验收。
- 遇到文档与现有代码冲突时，先记录冲突点，再决定是否调整实现。

## L1 入门：跑起来与基础契约

- [ ] T01 项目骨架与健康检查
  - 目标：确认前端、后端、CLI 的模块边界，完成最小可运行服务和健康检查接口。
  - 参考 wiki 小文档：
    - [1.7 技术栈速览](wiki/01-项目概述/1.7-技术栈速览.md)
    - [2.3.1 前端模块 web](wiki/02-系统架构/2.3-模块详细划分/2.3.1-前端模块-web.md)
    - [2.3.2 后端模块 server](wiki/02-系统架构/2.3-模块详细划分/2.3.2-后端模块-server.md)
    - [4.3.0 健康检查](wiki/04-API设计/4.3-接口清单/4.3.0-健康检查.md)
  - 验收：本地服务可启动；健康检查接口返回统一成功响应。

- [ ] T02 统一 API 响应与错误格式
  - 目标：建立后端统一响应结构、错误码约定和基础中间件。
  - 参考 wiki 小文档：
    - [4.1 设计原则](wiki/04-API设计/4.1-设计原则.md)
    - [4.2 统一响应格式](wiki/04-API设计/4.2-统一响应格式.md)
    - [4.4 OpenAPI 文档生成](wiki/04-API设计/4.4-OpenAPI-文档生成/README.md)
  - 验收：至少覆盖成功、业务错误、参数错误、未认证四类响应。

- [ ] T03 基础数据模型迁移
  - 目标：先落地用户、项目、需求、Issue 四类核心表，保证后续 CRUD 有稳定基础。
  - 参考 wiki 小文档：
    - [3.2.1 User 用户表](wiki/03-数据模型/3.2-核心表结构/3.2.1-User-用户表.md)
    - [3.2.2 Project 项目表](wiki/03-数据模型/3.2-核心表结构/3.2.2-Project-项目表.md)
    - [3.2.3 Requirement 需求表](wiki/03-数据模型/3.2-核心表结构/3.2.3-Requirement-需求表.md)
    - [3.2.4 Issue 核心表](wiki/03-数据模型/3.2-核心表结构/3.2.4-Issue-核心表.md)
    - [3.4 索引策略](wiki/03-数据模型/3.4-索引策略.md)
  - 验收：迁移可重复执行；核心字段、索引、外键或逻辑关联符合文档。

- [ ] T04 认证登录与 API Key 基础能力
  - 目标：实现登录、当前用户、API Key 鉴权的最小闭环。
  - 参考 wiki 小文档：
    - [4.3.1 认证模块](wiki/04-API设计/4.3-接口清单/4.3.1-认证模块-apiv1auth.md)
    - [4.3.1c API 密钥管理](wiki/04-API设计/4.3-接口清单/4.3.1c-API-密钥管理-apiv1api-keys.md)
    - [3.2.0c ApiKey API 密钥表](wiki/03-数据模型/3.2-核心表结构/3.2.0c-ApiKey-API-密钥表.md)
  - 验收：密码登录和 API Key 调用均可访问受保护接口；无效凭证被拒绝。

## L2 基础业务：项目、需求、Issue

- [ ] T05 项目管理 CRUD
  - 目标：实现项目创建、列表、详情、更新、归档或删除的基础接口和页面入口。
  - 参考 wiki 小文档：
    - [4.3.2 项目管理](wiki/04-API设计/4.3-接口清单/4.3.2-项目管理-apiv1projects.md)
    - [Project.settings](wiki/03-数据模型/3.3-配置与状态字段设计细节/Project.settings.md)
  - 验收：项目可完整增删改查；列表支持基础分页和按组织/用户隔离。

- [ ] T06 需求管理 CRUD 与状态字段
  - 目标：实现需求创建、编辑、列表、状态流转入口，为后续拆解 Issue 做准备。
  - 参考 wiki 小文档：
    - [4.3.3 需求管理](wiki/04-API设计/4.3-接口清单/4.3.3-需求管理-apiv1requirements.md)
    - [3.2.3 Requirement 需求表](wiki/03-数据模型/3.2-核心表结构/3.2.3-Requirement-需求表.md)
    - [Requirement.status 状态字段](wiki/03-数据模型/3.3-配置与状态字段设计细节/Requirement.status-状态枚举.md)
  - 验收：需求状态变更被记录；非法状态值不能写入。

- [ ] T07 Issue、评论、标签基础管理
  - 目标：实现 Issue CRUD、评论、标签和 IssueLabel 关联，形成最小任务池。
  - 参考 wiki 小文档：
    - [4.3.4 Issue 管理](wiki/04-API设计/4.3-接口清单/4.3.4-Issue-管理-apiv1issues.md)
    - [4.3.9 标签管理](wiki/04-API设计/4.3-接口清单/4.3.9-标签管理-apiv1labels.md)
    - [3.2.4b Comment Issue 评论表](wiki/03-数据模型/3.2-核心表结构/3.2.4b-Comment-Issue-评论表.md)
    - [3.2.14 IssueLabel 多对多关联表](wiki/03-数据模型/3.2-核心表结构/3.2.14-IssueLabel-多对多关联表.md)
  - 验收：Issue 可按项目、状态、标签筛选；评论和标签不会跨项目串数据。

- [ ] T08 看板视图映射
  - 目标：根据 Issue 状态生成看板列，提供前端可用的数据结构。
  - 参考 wiki 小文档：
    - [5.6 看板视图映射](wiki/05-工作流引擎/5.6-看板视图映射.md)
    - [5.1 Issue 生命周期状态机](wiki/05-工作流引擎/5.1-Issue-生命周期状态机.md)
  - 验收：看板列与状态机一致；拖拽或状态更新不会产生非法状态。

## L3 工作流闭环：状态机、日志、续跑

- [ ] T09 Issue FSM 状态机最小实现
  - 目标：使用状态机约束 Issue 从 draft 到 done 的核心流转。
  - 参考 wiki 小文档：
    - [5.1 Issue 生命周期状态机](wiki/05-工作流引擎/5.1-Issue-生命周期状态机.md)
    - [5.2 状态详解](wiki/05-工作流引擎/5.2-状态详解.md)
    - [5.3.1 Issue 状态机定义](wiki/05-工作流引擎/5.3-状态机实现-—-基于-looplabfsm/5.3.1-Issue-状态机定义.md)
    - [5.3.2 转换验证](wiki/05-工作流引擎/5.3-状态机实现-—-基于-looplabfsm/5.3.2-转换验证.md)
  - 验收：合法流转通过，非法流转失败；状态变化有测试覆盖。

- [ ] T10 执行日志、历史记录与审计记录
  - 目标：为 Issue 状态变化、AI 执行、人工操作写入可追踪日志。
  - 参考 wiki 小文档：
    - [3.2.5 ExecutionLog 执行日志表](wiki/03-数据模型/3.2-核心表结构/3.2.5-ExecutionLog-执行日志表.md)
    - [3.2.5c HistoryRecord 历史记录表](wiki/03-数据模型/3.2-核心表结构/3.2.5c-HistoryRecord-历史记录表.md)
    - [3.2.12 AuditLog 操作审计表](wiki/03-数据模型/3.2-核心表结构/3.2.12-AuditLog-操作审计表.md)
    - [5.8 历史记录自动采集](wiki/05-工作流引擎/5.8-历史记录自动采集/README.md)
  - 验收：关键业务动作可在详情页或接口中追溯到操作者、时间和变更内容。

- [ ] T11 Vibe Coding 续跑能力
  - 目标：允许 AI 执行失败或用户补充要求后继续编辑同一 Issue。
  - 参考 wiki 小文档：
    - [5.4.3 继续编辑 vibe coding 续跑](wiki/05-工作流引擎/5.4-关键业务逻辑-—-回调实现/5.4.3-继续编辑-vibe-coding-续跑.md)
    - [Issue.aiConfig](wiki/03-数据模型/3.3-配置与状态字段设计细节/Issue.aiConfig.md)
    - [4.3.5 执行管理](wiki/04-API设计/4.3-接口清单/4.3.5-执行管理-apiv1executions.md)
  - 验收：同一 Issue 可多轮追加上下文；历史执行记录不被覆盖。

## L4 AI 核心：分析、拆解、工具适配、沙箱

- [ ] T12 AI 工具配置与 Adapter 接口
  - 目标：实现 AI 工具配置表、工具列表接口和统一 Adapter 契约。
  - 参考 wiki 小文档：
    - [3.2.8 AIToolConfig AI 工具配置表](wiki/03-数据模型/3.2-核心表结构/3.2.8-AIToolConfig-AI-工具配置表.md)
    - [4.3.8 AI 工具配置](wiki/04-API设计/4.3-接口清单/4.3.8-AI-工具配置-apiv1ai-tools.md)
    - [7.6 工具选择策略](wiki/07-AI编程工具集成/7.6-工具选择策略/README.md)
  - 验收：可新增/启用/禁用 AI 工具；后端能按任务选择一个可用工具。

- [ ] T13 IssueAnalyzer 前置分析
  - 目标：在 Issue 执行前生成复杂度、风险、建议工具和初步计划。
  - 参考 wiki 小文档：
    - [分析 Prompt 模板](wiki/07-AI编程工具集成/7.6b-前置分析与分类适配器-IssueAnalyzer/分析-Prompt-模板.md)
    - [异步分析任务](wiki/07-AI编程工具集成/7.6b-前置分析与分类适配器-IssueAnalyzer/异步分析任务.md)
    - [分析与计划触发时机](wiki/07-AI编程工具集成/7.6b-前置分析与分类适配器-IssueAnalyzer/分析与计划触发时机.md)
    - [前端体验](wiki/07-AI编程工具集成/7.6b-前置分析与分类适配器-IssueAnalyzer/前端体验.md)
  - 验收：Issue 创建或进入分析状态后可得到分析结果；分析失败不会阻断人工处理。

- [ ] T14 RequirementDecomposer 需求拆解
  - 目标：把较大需求拆成可执行 Issue，并给用户确认入口。
  - 参考 wiki 小文档：
    - [拆解流程](wiki/07-AI编程工具集成/7.6c-需求自动拆解-RequirementDecomposer/拆解流程.md)
    - [适配器实现](wiki/07-AI编程工具集成/7.6c-需求自动拆解-RequirementDecomposer/适配器实现.md)
    - [用户确认界面](wiki/07-AI编程工具集成/7.6c-需求自动拆解-RequirementDecomposer/用户确认界面.md)
  - 验收：用户可确认、修改或拒绝 AI 拆解结果；确认后批量生成 Issue。

- [ ] T15 AI 执行通信与 Docker 沙箱
  - 目标：打通后端调度、工具 Adapter、沙箱执行环境和执行日志。
  - 参考 wiki 小文档：
    - [7.1b 通信机制](wiki/07-AI编程工具集成/7.1-概述/7.1b-通信机制.md)
    - [7.1c 触发 opencode 开始开发](wiki/07-AI编程工具集成/7.1-概述/7.1c-触发-opencode-开始开发.md)
    - [SandboxManager 实现](wiki/07-AI编程工具集成/7.2-Docker-沙盒执行环境/SandboxManager-实现.md)
    - [Docker 沙箱镜像](wiki/07-AI编程工具集成/7.2-Docker-沙盒执行环境/Docker-沙盒镜像.md)
  - 验收：一个 Issue 可被投递到沙箱执行；执行输出、失败原因和退出状态可查询。

- [ ] T16 OpenCode 与 Codex 适配
  - 目标：先实现两个 AI 工具适配器，验证统一 Adapter 是否足够。
  - 参考 wiki 小文档：
    - [OpenCode 适配方式](wiki/07-AI编程工具集成/7.3-OpenCode-适配/适配方式.md)
    - [Codex 适配方式](wiki/07-AI编程工具集成/7.4-Codex-OpenAI-适配/适配方式.md)
    - [自动选择](wiki/07-AI编程工具集成/7.6-工具选择策略/自动选择.md)
    - [手动选择](wiki/07-AI编程工具集成/7.6-工具选择策略/手动选择.md)
  - 验收：用户可手动选择工具；自动选择策略有可解释结果。

## L5 Git 闭环：克隆、提交、PR

- [ ] T17 Git URL 校验与凭证管理
  - 目标：在项目接入代码仓库前完成 URL 可用性校验和 Git 凭证存储。
  - 参考 wiki 小文档：
    - [校验流程](wiki/08-Git集成方案/8.2b-Git-URL-可用性校验/校验流程.md)
    - [URL 规范化规则](wiki/08-Git集成方案/8.2b-Git-URL-可用性校验/URL-规范化规则.md)
    - [3.2.7 GitCredential Git 凭证表](wiki/03-数据模型/3.2-核心表结构/3.2.7-GitCredential-Git-凭证表.md)
    - [SSH known_hosts 管理](wiki/08-Git集成方案/8.2d-SSH-密钥管理/SSH-knownhosts-管理.md)
  - 验收：错误 URL、无权限仓库、无效 SSH Key 都有明确错误提示。

- [ ] T18 沙箱内 Git Clone、Commit、Push
  - 目标：让 AI 执行任务时在沙箱内完成代码克隆、修改提交和推送。
  - 参考 wiki 小文档：
    - [Clone in Sandbox 实现](wiki/08-Git集成方案/8.2c-代码克隆-Git-Clone/Clone-in-Sandbox-实现.md)
    - [Clone 策略](wiki/08-Git集成方案/8.2c-代码克隆-Git-Clone/Clone-策略.md)
    - [容器内 Commit & Push](wiki/08-Git集成方案/8.2c-代码克隆-Git-Clone/容器内-Commit-&-Push.md)
    - [8.3 分支命名规范](wiki/08-Git集成方案/8.3-分支命名规范.md)
  - 验收：每个 Issue 生成独立分支；提交信息可追踪到 Issue。

- [ ] T19 PR 自动创建与 Provider 抽象
  - 目标：完成 GitHub/GitLab/Gitea Provider 抽象，并在任务完成后创建 PR。
  - 参考 wiki 小文档：
    - [8.6 GitProvider 统一接口](wiki/08-Git集成方案/8.6-GitProvider-统一接口.md)
    - [触发条件](wiki/08-Git集成方案/8.5-PR-自动创建与管理/触发条件.md)
    - [PR 模板](wiki/08-Git集成方案/8.5-PR-自动创建与管理/PR-模板.md)
    - [PR 附加信息](wiki/08-Git集成方案/8.5-PR-自动创建与管理/PR-附加信息.md)
  - 验收：done 状态的 Issue 能自动生成 PR；PR 描述包含任务、测试、日志链接。

## L6 扩展闭环：插件、Skills、测试用例

- [ ] T20 Hook Registry 与工作流钩子
  - 目标：建立钩子注册、执行、失败隔离和 Webhook 签名校验。
  - 参考 wiki 小文档：
    - [钩子注册](wiki/06-插件与钩子体系/6.5-钩子系统-Hook-Registry/钩子注册.md)
    - [钩子执行流程](wiki/06-插件与钩子体系/6.5-钩子系统-Hook-Registry/钩子执行流程.md)
    - [Webhook 签名验证](wiki/06-插件与钩子体系/6.5-钩子系统-Hook-Registry/Webhook-签名验证.md)
    - [5.5 工作流 Hook 点](wiki/05-工作流引擎/5.5-工作流-Hook-点-集成到-FSM-回调/README.md)
  - 验收：核心状态变化可触发钩子；单个钩子失败不影响主流程提交状态。

- [ ] T21 Skills 技能体系
  - 目标：实现 Skill 定义、加载、启停和与 AI 执行的集成点。
  - 参考 wiki 小文档：
    - [Skill 定义格式](wiki/06-插件与钩子体系/6.8-Skills-技能体系/Skill-定义格式.md)
    - [Skill 加载机制](wiki/06-插件与钩子体系/6.8-Skills-技能体系/Skill-加载机制.md)
    - [Skill 与 AI 执行的集成](wiki/06-插件与钩子体系/6.8-Skills-技能体系/Skill-与-AI-执行的集成.md)
    - [4.3.10 Skills 管理](wiki/04-API设计/4.3-接口清单/4.3.10-Skills-管理-apiv1skills.md)
  - 验收：系统能发现 Skill、读取元信息，并在 AI 执行前注入可用 Skill。

- [ ] T22 测试用例自动生成与审核流
  - 目标：根据需求或 Issue 生成测试用例，并通过 TestCase FSM 审核。
  - 参考 wiki 小文档：
    - [7.8 测试用例自动生成](wiki/07-AI编程工具集成/7.8-测试用例自动生成/README.md)
    - [触发机制](wiki/07-AI编程工具集成/7.8-测试用例自动生成/触发机制.md)
    - [API 调用](wiki/07-AI编程工具集成/7.8-测试用例自动生成/API-调用.md)
    - [5.7.1 TestCase FSM 实现](wiki/05-工作流引擎/5.7-TestCase-审核工作流/5.7.1-TestCase-FSM-实现.md)
  - 验收：生成结果可审核、退回、通过；通过后可关联到 Issue 或 Requirement。

## L7 生产闭环：监控、SaaS、部署、测试

- [ ] T23 监控事件接入与自动创建 Issue
  - 目标：接收监控事件，去重、分级，并按策略生成 Issue。
  - 参考 wiki 小文档：
    - [9.6 监控 SDK 事件类型](wiki/09-监控与反馈闭环/9.6-监控-SDK-事件类型.md)
    - [9.3 事件处理管道](wiki/09-监控与反馈闭环/9.3-事件处理管道/README.md)
    - [9.4 自动创建 Issue 策略](wiki/09-监控与反馈闭环/9.4-自动创建-Issue-策略/README.md)
    - [3.2.6 MonitorEvent 监控事件表](wiki/03-数据模型/3.2-核心表结构/3.2.6-MonitorEvent-监控事件表.md)
  - 验收：重复事件不会刷屏；高优先级事件可自动生成 Issue。

- [ ] T24 监控面板与 Issue 双向追踪
  - 目标：提供监控统计、趋势、事件详情，以及事件与 Issue 的互跳。
  - 参考 wiki 小文档：
    - [9.5 监控面板](wiki/09-监控与反馈闭环/9.5-监控面板/README.md)
    - [9.7 反馈入口](wiki/09-监控与反馈闭环/9.7-反馈入口.md)
    - [9.8 监控与 Issue 双向追踪](wiki/09-监控与反馈闭环/9.8-监控与-Issue-双向追溯.md)
  - 验收：用户可以从监控事件进入 Issue，也可以从 Issue 回看来源事件。

- [ ] T25 SaaS 多租户隔离
  - 目标：补齐组织、成员、租户识别和查询隔离，避免跨组织数据访问。
  - 参考 wiki 小文档：
    - [2.4 SaaS 多租户架构](wiki/02-系统架构/2.4-SaaS-多租户架构/README.md)
    - [3.2.0 Organization 组织表](wiki/03-数据模型/3.2-核心表结构/3.2.0-Organization-组织表-SaaS-核心.md)
    - [3.2.0b OrgMember 组织成员表](wiki/03-数据模型/3.2-核心表结构/3.2.0b-OrgMember-组织成员表.md)
    - [4.3.1b 组织管理](wiki/04-API设计/4.3-接口清单/4.3.1b-组织管理-apiv1orgs-SaaS.md)
  - 验收：任意项目、需求、Issue、日志查询都受 orgId 约束；越权访问有测试。

- [ ] T26 部署、健康检查与资源配置
  - 目标：完成 SQLite 入门部署、Docker Compose 生产部署、环境变量和健康检查。
  - 参考 wiki 小文档：
    - [10.3 环境变量清单](wiki/10-部署与运维/10.3-环境变量清单.md)
    - [10.4 一键启动](wiki/10-部署与运维/10.4-一键启动/README.md)
    - [10.5 数据库迁移](wiki/10-部署与运维/10.5-数据库迁移.md)
    - [内置健康端点](wiki/10-部署与运维/10.7-健康检查与监控/内置健康端点.md)
    - [10.8 资源需求](wiki/10-部署与运维/10.8-资源需求/README.md)
  - 验收：新环境可按文档启动；健康检查、日志和迁移流程可复现。

- [ ] T27 测试金字塔与 CI/CD 流水线
  - 目标：为核心链路建立单元、集成、E2E 和 CI/CD 测试策略。
  - 参考 wiki 小文档：
    - [11.1 测试金字塔](wiki/11-测试策略/11.1-测试金字塔.md)
    - [单元测试覆盖目标](wiki/11-测试策略/11.2-单元测试/覆盖目标.md)
    - [FSM 状态机测试](wiki/11-测试策略/11.3-集成测试/FSM-状态机测试.md)
    - [API 契约测试](wiki/11-测试策略/11.3-集成测试/API-契约测试.md)
    - [11.5 CI/CD 测试流水线](wiki/11-测试策略/11.5-CICD-测试流水线.md)
  - 验收：核心 API、FSM、AI 执行、Git/PR、监控自动 Issue 至少各有一条自动化测试链路。
