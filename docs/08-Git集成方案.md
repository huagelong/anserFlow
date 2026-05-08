# 08 - Git 集成方案

## 8.1 概述

flowcode 不内置 Git 托管，而是通过 **GitProvider 接口** 对接外部 Git 平台（GitHub / GitLab / Gitea）。系统支持从远程仓库拉取代码（HTTP / SSH），供 AI 工具在 Docker 沙盒中执行编码任务。AI 编码完成后，系统自动在对应平台创建分支、提交代码、发起 Pull Request / Merge Request。

> **职责边界**：Issue 由 flowcode 独立管理，不在 Git 平台创建。Git 平台仅负责 PR/MR 生命周期 + Webhook 事件回调。Issue 单向导入为**纯手动操作**（Web 控制台按钮 / CLI 命令），不做自动同步。所有三个平台 SDK（go-github / go-gitlab / go-gitea）均为必需依赖。

### 8.1b Git SDK 配置清单

#### 项目 Git 地址

| 字段 | 来源 | 说明 |
|------|------|------|
| `gitRepoUrl` | Project 表 | 仓库地址，支持 `https://` 和 `git@` 两种协议 |
| `gitCredentialId` | Project 表 FK→GitCredential | **项目选用的凭证**，从已配置的凭证列表中选取 |

> 一个项目绑定一个仓库地址。先配置凭证（`POST git/connect`），再在项目设置中选择要使用的凭证。创建项目时可暂不选凭证，后续通过更新接口绑定。

#### Git 凭证（GitCredential 表）

一个项目可配置多条凭证（如对接多个仓库、多个平台），每条凭证通过 `name` 区分。支持 HTTP（PAT/OAuth）和 SSH 两种认证方式：

| 配置项 | HTTP 模式 | SSH 模式 | 说明 |
|--------|:--------:|:--------:|------|
| `name` | ✅ | ✅ | 凭证名称，如 "生产仓库"、"GitLab 备份" |
| `provider` | ✅ | ✅ | `github` / `gitlab` / `gitea` |
| `authMethod` | `http` | `ssh` | 认证方式 |
| `accessToken` | ✅ | — | PAT 或 OAuth Token，AES-256-GCM 加密存储 |
| `tokenType` | `pat` / `oauth` | — | Token 类型 |
| `sshPrivateKey` | — | ✅ | SSH 私钥，AES-256-GCM 加密存储 |
| `sshPassphrase` | — | 可选 | 私钥密码短语，加密存储 |
| `expiresAt` | ✅ | ✅ | 过期时间 |
| `lastUsedAt` | ✅ | ✅ | 最后使用时间（自动更新） |

#### GitProvider 工厂参数

```go
// internal/adapter/git/factory.go

const (
    ProviderGitHub = "github"  // go-github SDK，无需 baseURL
    ProviderGitLab = "gitlab"  // go-gitlab SDK，支持自建实例 baseURL
    ProviderGitea  = "gitea"   // go-gitea SDK，支持自建实例 baseURL
)

func NewGitProvider(pt ProviderType, accessToken, baseURL string) (GitProvider, error)
```

| 平台 | SDK | baseURL 必填 | 示例 |
|------|-----|:---:|------|
| GitHub | go-github | 否 | `NewGitHubProvider("ghp_xxx")` |
| GitLab | go-gitlab | 自建时必填 | `NewGitLabProvider("glpat-xxx", "https://gitlab.example.com")` |
| Gitea | go-gitea | 自建时必填 | `NewGiteaProvider("token", "https://gitea.example.com")` |

> GitHub 始终使用 `api.github.com`；GitLab/Gitea 的 SaaS 版（gitlab.com / gitea.com）baseURL 传空字符串即可，SDK 自动使用默认 API 端点。

#### 配置方式

| 方式 | 命令 / 端点 |
|------|------------|
| CLI | `flowcode project git-connect --name "生产仓库" --provider github --token ghp_xxx` |
| CLI | `flowcode project set-credential --id cred_xxx`（为当前项目选择凭证） |
| API | `POST /api/v1/projects/:pid/git/connect`（新增凭证） |
| API | `GET /api/v1/projects/:pid/git/credentials`（凭证列表） |
| API | `DELETE /api/v1/projects/:pid/git/credentials/:id`（删除凭证） |
| API | `PUT /api/v1/projects/:id`（更新 `gitCredentialId` 选择凭证） |
| Web | 项目设置 → Git 连接 → 选择平台 → 粘贴 PAT / SSH 密钥 |

**API 请求示例**：

```bash
# HTTP 方式（PAT）—— 新增凭证
curl -X POST /api/v1/projects/proj_1/git/connect \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "GitHub 主仓库",
    "authMethod": "http",
    "accessToken": "ghp_xxxxxxxxxxxxxxxxxxxx",
    "tokenType": "pat",
    "provider": "github"
  }'

# SSH 方式
curl -X POST /api/v1/projects/proj_1/git/connect \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "公司 GitLab",
    "authMethod": "ssh",
    "sshPrivateKey": "-----BEGIN OPENSSH PRIVATE KEY-----\n...\n-----END OPENSSH PRIVATE KEY-----",
    "provider": "gitlab"
  }'

# 查看凭证列表
curl -X GET /api/v1/projects/proj_1/git/credentials \
  -H "Authorization: Bearer $TOKEN"

# 删除凭证
curl -X DELETE /api/v1/projects/proj_1/git/credentials/cred_1 \
  -H "Authorization: Bearer $TOKEN"
```

## 8.2 整体流程

```
创建项目 → Git URL 校验
                │
        ┌───────┴────────┐
        ▼                ▼
   [HTTP 模式]      [SSH 模式]
   验证 PAT Token    验证 SSH 密钥
        │                │
        └───────┬────────┘
                ▼
        项目创建成功
                │
Issue 进入编码阶段 (approved → 分析完成 → 计划完成 → running)
        │
        ▼
┌───────────────────────────────┐
│  SessionService (沙盒编排)     │
│  ┌───────────────────────────┐ │
│  │ 1. 初始化沙盒             │ │
│  │ 2. Git Clone 拉取代码     │ │
│  │    - HTTP: token@URL      │ │
│  │    - SSH: 临时 SSH 密钥   │ │
│  │ 3. 创建 flowcode/ 分支    │ │
│  │ 4. AI 工具按计划执行编码   │ │
│  │ 5. Git Add & Commit       │ │
│  │ 6. Git Push 到远程        │ │
│  └───────────────────────────┘ │
└──────────┬────────────────────┘
           │
           ▼
┌───────────────────┐
│  GitService        │
│  ┌───────────────┐ │
│  │ 1. 选择Provider│ │
│  │ 2. 获取凭证    │ │
│  │ 3. 创建 PR     │ │
│  │ 4. 保存 PR URL │ │
│  └───────────────┘ │
└──────────┬────────┘
           │
           ▼
┌───────────────────┐      ┌───────────────────┐
│   GitHub API      │      │   Gitea API       │
│   (github.com)    │      │   (self-hosted)   │
└───────────────────┘      └───────────────────┘
           │                        │
           ▼                        ▼
     PR Created               PR Created
           │                        │
           ▼                        ▼
┌───────────────────────────────────────────┐
│         Webhook 回调 flowcode                          │
│  - PR merged  → issue 触发 deploy → close             │
│  - PR closed  → issue 触发 reopen → reopened → approved │
└───────────────────────────────────────────┘
```

## 8.2b Git URL 可用性校验

添加项目时，系统必须先校验 Git 仓库地址是否可达，防止保存无效配置。

### 校验流程

```go
// internal/service/git/url_validator.go

type GitURLValidator struct {
    credentialRepo *CredentialRepository
}

// ValidateURL 校验 Git 地址是否可达
func (v *GitURLValidator) ValidateURL(ctx context.Context, gitURL string, protocol string, credentialID string) (*ValidateResult, error) {
    // 1. 解析并规范化 URL
    parsed, err := v.parseAndNormalize(gitURL)
    if err != nil {
        return nil, fmt.Errorf("invalid git URL: %w", err)
    }

    // 2. 根据协议校验
    switch protocol {
    case "http":
        return v.validateHTTP(ctx, parsed, credentialID)
    case "ssh":
        return v.validateSSH(ctx, parsed, credentialID)
    default:
        return nil, fmt.Errorf("unsupported protocol: %s", protocol)
    }
}

func (v *GitURLValidator) validateHTTP(ctx context.Context, parsed *GitURL, credentialID string) (*ValidateResult, error) {
    // 获取凭证
    cred, err := v.credentialRepo.GetByID(ctx, credentialID)
    if err != nil {
        return nil, fmt.Errorf("credential not found: %w", err)
    }

    // 发送 git ls-remote 以验证访问权限和仓库存在性
    // 使用 token 构建认证 URL: https://token@github.com/user/repo.git
    authURL := fmt.Sprintf("%s://%s@%s/%s.git",
        parsed.Scheme, cred.AccessToken, parsed.Host, parsed.RepoPath)

    cmd := exec.CommandContext(ctx, "git", "ls-remote", "--heads", authURL)
    cmd.Env = append(os.Environ(), "GIT_TERMINAL_PROMPT=0") // 禁止交互
    output, err := cmd.CombinedOutput()
    if err != nil {
        return nil, &ValidateError{
            Code:    "GIT_UNREACHABLE",
            Message: fmt.Sprintf("无法访问仓库: %s", strings.TrimSpace(string(output))),
            Err:     err,
        }
    }

    // 解析默认分支
    defaultBranch := "main"
    for _, line := range strings.Split(strings.TrimSpace(string(output)), "\n") {
        if strings.HasSuffix(line, "refs/heads/main") {
            defaultBranch = "main"
            break
        } else if strings.HasSuffix(line, "refs/heads/master") {
            defaultBranch = "master"
            break
        }
    }

    return &ValidateResult{
        Reachable:     true,
        DefaultBranch: defaultBranch,
        Branches:      parseRefs(string(output)),
    }, nil
}

func (v *GitURLValidator) validateSSH(ctx context.Context, parsed *GitURL, credentialID string) (*ValidateResult, error) {
    cred, err := v.credentialRepo.GetByID(ctx, credentialID)
    if err != nil {
        return nil, fmt.Errorf("credential not found: %w", err)
    }

    // 解密 SSH 私钥
    privateKey, err := v.decryptSSHKey(cred.SSHPrivateKey)
    if err != nil {
        return nil, fmt.Errorf("failed to decrypt SSH key: %w", err)
    }

    // 写入临时 SSH 密钥文件
    tmpKeyFile, err := v.writeTempKeyFile(privateKey, cred.SSHPassphrase)
    if err != nil {
        return nil, fmt.Errorf("failed to setup SSH key: %w", err)
    }
    defer os.Remove(tmpKeyFile)

    // 使用 GIT_SSH_COMMAND 指定临时密钥进行校验
    sshCmd := fmt.Sprintf("ssh -i %s -o StrictHostKeyChecking=accept-new -o PasswordAuthentication=no", tmpKeyFile)
    cmd := exec.CommandContext(ctx, "git", "ls-remote", "--heads", gitURL)
    cmd.Env = append(os.Environ(),
        "GIT_TERMINAL_PROMPT=0",
        "GIT_SSH_COMMAND="+sshCmd,
    )

    output, err := cmd.CombinedOutput()
    if err != nil {
        return nil, &ValidateError{
            Code:    "GIT_UNREACHABLE",
            Message: fmt.Sprintf("SSH 连接失败: %s", strings.TrimSpace(string(output))),
            Err:     err,
        }
    }

    defaultBranch := detectDefaultBranch(string(output))
    return &ValidateResult{
        Reachable:     true,
        DefaultBranch: defaultBranch,
        Branches:      parseRefs(string(output)),
    }, nil
}

type ValidateResult struct {
    Reachable     bool     `json:"reachable"`
    DefaultBranch string   `json:"defaultBranch"`
    Branches      []string `json:"branches"`
    RepoID        string   `json:"repoId"`        // 如 "user/repo"
}

type ValidateError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Err     error  `json:"-"`
}
```

### URL 规范化规则

| 输入格式 | HTTP 规范化 | SSH 规范化 |
|---------|-----------|----------|
| `github.com/user/repo` | `https://github.com/user/repo.git` | `git@github.com:user/repo.git` |
| `https://github.com/user/repo.git` | 不变 | — |
| `git@github.com:user/repo.git` | — | 不变 |
| `https://gitea.example.com/owner/repo` | `https://gitea.example.com/owner/repo.git` | `git@gitea.example.com:owner/repo.git` |

### 校验时机与缓存

- **创建项目时**：必须校验通过才能保存
- **更新 Git 配置时**：修改 URL 或凭证后需重新校验
- **校验结果缓存**：Redis 缓存 5 分钟，`validate` 参数可强制刷新
- **后台巡检**：定时任务每天对所有项目的 Git 地址做可达性检查，失败则告警

## 8.2c 代码克隆 (Git Clone)

AI 工具执行编码任务前，系统需要在 Docker 沙盒容器内克隆目标仓库。克隆支持 HTTP 和 SSH 两种协议。

### Clone in Sandbox 实现

克隆操作不再在宿主机本地文件系统执行，而是在 Docker 容器内通过 `SandboxManager.Exec()` 完成：

```go
// internal/service/sandbox/prepare.go

// PrepareWorkspace 在容器内准备代码工作区
func (d *DockerSandbox) PrepareWorkspace(ctx context.Context, sb *Sandbox, project *model.Project, cred *model.GitCredential) error {
    // 1. 根据协议构建 clone URL 和环境变量
    var cloneCmd []string

    switch cred.AuthMethod {
    case "http":
        // HTTP: token 嵌入 URL
        cloneURL := buildHTTPCloneURL(project.GitRepoUrl, cred.AccessToken)
        cloneCmd = []string{"git", "clone", "--depth", "1", cloneURL, "/workspace/repo"}

    case "ssh":
        // SSH: 将解密后的私钥写入容器内临时文件
        sshKeyPath := "/tmp/flowcode_ssh_key"
        privateKey := decryptAES256GCM(cred.SSHPrivateKey, os.Getenv("FLOWCODE_ENCRYPTION_KEY"))

        // 先写入私钥到容器
        d.Exec(ctx, sb, []string{"sh", "-c", fmt.Sprintf(
            "echo '%s' > %s && chmod 600 %s",
            escapeShell(privateKey), sshKeyPath, sshKeyPath,
        )}, nil)

        cloneCmd = []string{"sh", "-c", fmt.Sprintf(
            "GIT_SSH_COMMAND='ssh -i %s -o StrictHostKeyChecking=accept-new' git clone --depth 1 %s /workspace/repo",
            sshKeyPath, project.GitRepoUrl,
        )}
    }

    // 2. 容器内执行 clone
    result, err := d.Exec(ctx, sb, cloneCmd, nil)
    if err != nil {
        return fmt.Errorf("clone in sandbox: %w, output: %s", err, result.Output)
    }

    // 3. 注入 Skills prompt
    skillsContent := d.loadSkills(ctx, project.ID)
    writeSkillsCmd := []string{"sh", "-c", fmt.Sprintf(
        "cat > /workspace/.flowcode_prompt.md << 'EOSKILL'\n%s\nEOSKILL",
        skillsContent,
    )}
    d.Exec(ctx, sb, writeSkillsCmd, nil)

    return nil
}

// buildHTTPCloneURL 将 token 嵌入 URL
func buildHTTPCloneURL(repoURL string, token string) string {
    u, _ := url.Parse(repoURL)
    u.User = url.User(token)
    return u.String()
}
```

### Clone 策略

| 参数 | 说明 |
|------|------|
| `--depth 1` | 浅克隆，仅拉取最新提交 |
| 超时 | 120 秒 |
| 重试 | 失败自动重试 1 次 |
| SSH 密钥 | 容器内临时文件，执行完 `rm` 清理 |
| 工作目录 | 容器内 `/workspace/repo`，宿主机通过 Docker Volume 访问 |

### 容器内 Commit & Push

```go
// internal/service/sandbox/commit.go

func (d *DockerSandbox) CommitAndPush(ctx context.Context, sb *Sandbox, issue *model.Issue) (string, error) {
    cmds := [][]string{
        {"git", "-C", "/workspace/repo", "config", "user.email", "flowcode@flowcode.dev"},
        {"git", "-C", "/workspace/repo", "config", "user.name", "flowcode"},
        {"git", "-C", "/workspace/repo", "add", "-A"},
        {"git", "-C", "/workspace/repo", "commit", "-m", issue.Title},
        {"git", "-C", "/workspace/repo", "push", "origin", issue.GitBranch},
    }
    for _, cmd := range cmds {
        if _, err := d.Exec(ctx, sb, cmd, nil); err != nil {
            return "", fmt.Errorf("git cmd '%s': %w", strings.Join(cmd, " "), err)
        }
    }

    // 获取最新 commit SHA
    result, _ := d.Exec(ctx, sb, []string{"git", "-C", "/workspace/repo", "rev-parse", "HEAD"}, nil)
    return strings.TrimSpace(result.Output), nil
}
```

> **安全性**：容器内的 git push 使用的是容器中注入的凭证（HTTP token 或 SSH key），不会泄露到宿主机。容器销毁后凭证随之清除。

## 8.2d SSH 密钥管理

SSH 协议适用于自托管 Gitea/GitLab 及无法使用 PAT 的场景。

### 密钥格式要求

| 格式 | 支持 | 备注 |
|------|------|------|
| RSA (>=2048 bit) | ✅ 推荐 | 兼容所有平台 |
| ED25519 | ✅ 推荐 | 更高安全性 |
| ECDSA | ✅ | — |
| DSA | ❌ | 已弃用 |

### 密钥安全存储

```go
// internal/service/git/ssh_key_manager.go

type SSHKeyManager struct {
    encryptionKey []byte // 来自 FLOWCODE_ENCRYPTION_KEY
}

func (m *SSHKeyManager) ValidatePublicKey(privateKeyPEM string, passphrase string) (fingerprint string, err error) {
    // 1. 解析私钥
    var block *pem.Block
    if passphrase != "" {
        block, _ = pem.Decode([]byte(privateKeyPEM))
        decrypted, err := x509.DecryptPEMBlock(block, []byte(passphrase))
        if err != nil {
            return "", fmt.Errorf("私钥密码错误: %w", err)
        }
        block = &pem.Block{Type: block.Type, Bytes: decrypted}
    } else {
        block, _ = pem.Decode([]byte(privateKeyPEM))
    }

    // 2. 解析为 crypto.Signer
    var key interface{}
    switch block.Type {
    case "RSA PRIVATE KEY":
        key, err = x509.ParsePKCS1PrivateKey(block.Bytes)
    case "EC PRIVATE KEY":
        key, err = x509.ParseECPrivateKey(block.Bytes)
    case "OPENSSH PRIVATE KEY":
        key, err = ssh.ParseRawPrivateKey([]byte(privateKeyPEM))
    default:
        return "", fmt.Errorf("不支持的密钥类型: %s", block.Type)
    }
    if err != nil {
        return "", fmt.Errorf("密钥解析失败: %w", err)
    }

    // 3. 生成指纹 (SHA256)
    publicKey := getPublicKey(key)
    sshPublicKey, _ := ssh.NewPublicKey(publicKey)
    fingerprint = ssh.FingerprintSHA256(sshPublicKey)

    return fingerprint, nil
}

func (m *SSHKeyManager) EncryptAndStore(privateKeyPEM string, passphrase string) (encryptedKey string, encryptedPass string, err error) {
    encryptedKey, err = crypto.EncryptAES256GCM(privateKeyPEM, m.encryptionKey)
    if err != nil {
        return "", "", err
    }
    if passphrase != "" {
        encryptedPass, err = crypto.EncryptAES256GCM(passphrase, m.encryptionKey)
        if err != nil {
            return "", "", err
        }
    }
    return encryptedKey, encryptedPass, nil
}
```

### SSH known_hosts 管理

首次连接时自动添加到 `~/.ssh/known_hosts`（通过 `StrictHostKeyChecking=accept-new`），或由管理员预配置常用 Git 平台的 Host Key。

## 8.3 分支命名规范

```
flowcode/<issue-id>-<short-slug>

示例:
  flowcode/abc123-implement-login
  flowcode/def456-fix-pagination-bug
  flowcode/ghi789-refactor-auth-middleware
```

分支命名规则可自定义（见 `Project.settings.branchNaming`）。

## 8.4 Commit Message 生成

AI 工具执行完成后，系统从以下来源拼装 Commit Message：

1. **工具解析**：调用 `adapter.parseCommitMessage(result)` 从 AI 输出中提取
2. **规则生成**：基于 Issue 分类自动添加前缀

```
格式:
  <type>(<scope>): <description>

  Closes #<issue-flowcode-id>
  Ref: <original-issue-url>

类型映射:
  feature  → feat
  bug      → fix
  refactor → refactor
  docs     → docs
  infra    → chore
```

### 示例

```
feat(auth): implement JWT-based login endpoint

- Add POST /api/v1/auth/login route
- Add JWT sign and verify utilities
- Add login rate limiting
- Add unit tests for auth flow

Closes #vf-abc123
Ref: https://flowcode.local/projects/xxx/issues/abc123
```

## 8.5 PR 自动创建与管理

### Issue 与 PR 的职责边界

| 概念 | 管理方 | 说明 |
|------|--------|------|
| **Issue** | flowcode | 需求录入、优先级、分类、状态流转均在 flowcode 内部 |
| **PR / MR** | Git 平台 | 代码审查、合并、讨论在 GitHub/GitLab/Gitea 上 |
| **关联** | flowcode | PR URL 写入 Issue.prUrl；平台 PR body 中引用 flowcode Issue 链接 |

### 触发条件

- `Project.Settings.autoCreatePR === true`（默认开启）
- Issue 状态为 `done` 或 `pr_ready`
- Git Provider 凭证有效

### GitHub Flow 分支策略

```
main ────────────────────────────────────────────── (始终可部署)
  │
  ├── flowcode/abc123-implement-login ──→ PR ──→ merge
  ├── flowcode/def456-fix-pagination ──→ PR ──→ merge
  └── flowcode/ghi789-refactor-auth ────→ PR ──→ merge
```

- `main` 分支始终保护，不可直接推送
- 所有 AI 编码在 `flowcode/<issue-id>-<slug>` 分支进行
- 通过 PR 合并回 `main`
- PR 合并后触发 GitHub Actions 自动构建 Docker 镜像

### PR 模板

```go
// internal/service/git/pr_builder.go

func BuildPRBody(issue *model.Issue, log *model.ExecutionLog) string {
    return fmt.Sprintf(`
## Description
%s

## Changes Made by %s
- Files changed: %d
- Execution time: %s
- Attempt: #%d

## Changed Files
%s

## AI Execution Summary
`+"```"+`
%s
`+"```"+`

---
> This PR was automatically created by [flowcode](https://flowcode.dev)
> Issue: %s
> AI Tool: %s
`, issue.Description, log.ToolName, len(log.FilesChanged),
        formatDuration(log.Duration), log.Attempt,
        formatFilesChanged(log.FilesChanged),
        truncate(log.Stdout, 2000),
        issue.Title, log.ToolName)
}
```

### PR 附加信息

| 字段 | 值 |
|------|----|
| Labels | 映射 Issue 标签 + `ai-generated` 标签 |
| Reviewers | `Project.settings.defaultReviewers` |
| Draft | Issue.priority === 'p2' 或 'p3' 时设为 draft |

## 8.6 GitProvider 统一接口

所有 Git 平台的 PR/MR 操作通过统一接口抽象，三个平台均为必需依赖：

```go
// internal/adapter/git/provider.go

type GitProvider interface {
    // ── 凭证验证 ──
    ValidateCredentials(ctx context.Context, accessToken, repoURL string) error

    // ── 分支操作 ──
    CreateBranch(ctx context.Context, opts BranchOptions) error

    // ── PR/MR 操作 ──
    CreatePR(ctx context.Context, opts PROptions) (*PRResult, error)
    GetPR(ctx context.Context, repoID string, prNumber int) (*PRResult, error)
    ListPRs(ctx context.Context, repoID string, status string) ([]*PRResult, error)
    MergePR(ctx context.Context, repoID string, prNumber int, method string) error

    // ── Webhook 注册 ──
    RegisterWebhook(ctx context.Context, repoID, callbackURL, secret string) error
    UnregisterWebhook(ctx context.Context, repoID string, webhookID int64) error
    ParseWebhookPayload(r *http.Request) (*WebhookEvent, error)

    // ── Issue 导入 ──
    ListIssues(ctx context.Context, repoID string, opts ListIssueOptions) ([]*RemoteIssue, error)
    GetIssue(ctx context.Context, repoID string, issueNumber int) (*RemoteIssue, error)
}

// ── 公共类型 ──

type BranchOptions struct {
    Name       string  // 新分支名, 如 flowcode/abc123-implement-login
    BaseBranch string  // 源分支, 如 main
}

type PROptions struct {
    Title      string
    Body       string
    HeadBranch string
    BaseBranch string
    Draft      bool
    Labels     []string
    Reviewers  []string
}

type PRResult struct {
    ID        int64     `json:"id"`
    Number    int       `json:"number"`
    URL       string    `json:"url"`
    Title     string    `json:"title"`
    Status    string    `json:"status"`     // open / closed / merged
    Merged    bool      `json:"merged"`
    Mergeable bool      `json:"mergeable"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type WebhookEvent struct {
    Platform    string  // github / gitea / gitlab
    Event       string  // pull_request / merge_request
    Action      string  // opened / closed / merged / updated
    RepoFullName string
    PRNumber    int
    PRMerged    bool
    HeadRef     string
    Raw         map[string]any  // 原始 payload
}
```

> Issue 的创建/查询/更新由 flowcode 自身管理，仅支持从 Git 平台**单向导入**。Git 平台负责 PR/MR 生命周期 + Webhook 事件。

## 8.7 GitHub Provider（go-github SDK）

```go
// internal/adapter/git/github.go

import "github.com/google/go-github/v68/github"

type GitHubProvider struct {
    client *github.Client
}

func NewGitHubProvider(accessToken string) *GitHubProvider {
    return &GitHubProvider{
        client: github.NewClient(nil).WithAuthToken(accessToken),
    }
}

func (p *GitHubProvider) ValidateCredentials(ctx context.Context, _, _ string) error {
    _, _, err := p.client.Users.Get(ctx, "")
    return err
}

func (p *GitHubProvider) CreateBranch(ctx context.Context, opts BranchOptions) error {
    owner, repo := parseOwnerRepo(ctx)
    ref, _, err := p.client.Git.GetRef(ctx, owner, repo, "heads/"+opts.BaseBranch)
    if err != nil {
        return err
    }
    _, _, err = p.client.Git.CreateRef(ctx, owner, repo, &github.Reference{
        Ref:    github.String("refs/heads/" + opts.Name),
        Object: &github.GitObject{SHA: ref.Object.SHA},
    })
    return err
}

// ── PR 操作 ──

func (p *GitHubProvider) CreatePR(ctx context.Context, opts PROptions) (*PRResult, error) {
    owner, repo := parseOwnerRepo(ctx)
    pr, _, err := p.client.PullRequests.Create(ctx, owner, repo, &github.NewPullRequest{
        Title: github.String(opts.Title),
        Body:  github.String(opts.Body),
        Head:  github.String(opts.HeadBranch),
        Base:  github.String(opts.BaseBranch),
        Draft: github.Bool(opts.Draft),
    })
    if err != nil {
        return nil, err
    }
    // 设置 Labels 和 Reviewers
    if len(opts.Labels) > 0 {
        p.client.Issues.AddLabelsToIssue(ctx, owner, repo, pr.GetNumber(), opts.Labels)
    }
    if len(opts.Reviewers) > 0 {
        p.client.PullRequests.RequestReviewers(ctx, owner, repo, pr.GetNumber(),
            github.ReviewersRequest{Reviewers: opts.Reviewers})
    }
    return toPRResult(pr), nil
}

func (p *GitHubProvider) GetPR(ctx context.Context, repoID string, prNumber int) (*PRResult, error) {
    owner, repo := parseOwnerRepo(ctx)
    pr, _, err := p.client.PullRequests.Get(ctx, owner, repo, prNumber)
    if err != nil {
        return nil, err
    }
    return toPRResult(pr), nil
}

func (p *GitHubProvider) ListPRs(ctx context.Context, repoID string, status string) ([]*PRResult, error) {
    owner, repo := parseOwnerRepo(ctx)
    opts := &github.PullRequestListOptions{State: status} // open / closed / all
    prs, _, err := p.client.PullRequests.List(ctx, owner, repo, opts)
    if err != nil {
        return nil, err
    }
    var results []*PRResult
    for _, pr := range prs {
        results = append(results, toPRResult(pr))
    }
    return results, nil
}

func (p *GitHubProvider) MergePR(ctx context.Context, repoID string, prNumber int, method string) error {
    owner, repo := parseOwnerRepo(ctx)
    _, _, err := p.client.PullRequests.Merge(ctx, owner, repo, prNumber,
        "auto-merged by flowcode",
        &github.PullRequestOptions{MergeMethod: method}) // merge / squash / rebase
    return err
}

// ── Webhook ──

func (p *GitHubProvider) RegisterWebhook(ctx context.Context, repoID, callbackURL, secret string) error {
    owner, repo := parseOwnerRepo(ctx)
    _, _, err := p.client.Repositories.CreateHook(ctx, owner, repo, &github.Hook{
        Name:   github.String("web"),
        Active: github.Bool(true),
        Events: []string{"pull_request"},
        Config: map[string]any{
            "url":          callbackURL,
            "content_type": "json",
            "secret":       secret,
        },
    })
    return err
}

func (p *GitHubProvider) UnregisterWebhook(ctx context.Context, repoID string, webhookID int64) error {
    owner, repo := parseOwnerRepo(ctx)
    _, err := p.client.Repositories.DeleteHook(ctx, owner, repo, webhookID)
    return err
}

func (p *GitHubProvider) ParseWebhookPayload(r *http.Request) (*WebhookEvent, error) {
    payload, err := github.ValidatePayload(r, []byte(os.Getenv("GITHUB_WEBHOOK_SECRET")))
    if err != nil {
        return nil, err
    }
    event, _ := github.ParseWebHook(r.Header.Get("X-GitHub-Event"), payload)
    if prEvent, ok := event.(*github.PullRequestEvent); ok {
        return &WebhookEvent{
            Platform:     "github",
            Event:        "pull_request",
            Action:       prEvent.GetAction(),
            RepoFullName: prEvent.GetRepo().GetFullName(),
            PRNumber:     prEvent.GetPullRequest().GetNumber(),
            PRMerged:     prEvent.GetPullRequest().GetMerged(),
            HeadRef:      prEvent.GetPullRequest().GetHead().GetRef(),
        }, nil
    }
    return nil, fmt.Errorf("unsupported event: %s", r.Header.Get("X-GitHub-Event"))
}

func toPRResult(pr *github.PullRequest) *PRResult {
    return &PRResult{
        ID:     pr.GetID(),
        Number: pr.GetNumber(),
        URL:    pr.GetHTMLURL(),
        Title:  pr.GetTitle(),
        Status: pr.GetState(),
        Merged: pr.GetMerged(),
        Mergeable: pr.GetMergeable(),
        CreatedAt: pr.GetCreatedAt(),
        UpdatedAt: pr.GetUpdatedAt(),
    }
}
```

## 8.8 GitLab Provider（go-gitlab SDK）

GitLab 的概念叫 Merge Request (MR)，接口适配为统一的 PR 语义：

```go
// internal/adapter/git/gitlab.go

import "github.com/xanzy/go-gitlab"

type GitLabProvider struct {
    client   *gitlab.Client
    baseURL  string   // 自建 GitLab 地址
}

func NewGitLabProvider(accessToken, baseURL string) *GitLabProvider {
    opts := []gitlab.ClientOptionFunc{}
    if baseURL != "" {
        opts = append(opts, gitlab.WithBaseURL(baseURL))
    }
    client, _ := gitlab.NewClient(accessToken, opts...)
    return &GitLabProvider{client: client, baseURL: baseURL}
}

func (p *GitLabProvider) ValidateCredentials(ctx context.Context, _, _ string) error {
    _, _, err := p.client.Users.CurrentUser()
    return err
}

func (p *GitLabProvider) CreateBranch(ctx context.Context, opts BranchOptions) error {
    pid := getProjectID(ctx)
    _, _, err := p.client.Branches.CreateBranch(pid, &gitlab.CreateBranchOptions{
        Branch: gitlab.Ptr(opts.Name),
        Ref:    gitlab.Ptr(opts.BaseBranch),
    })
    return err
}

// ── MR 操作（适配为 PR 语义）──

func (p *GitLabProvider) CreatePR(ctx context.Context, opts PROptions) (*PRResult, error) {
    pid := getProjectID(ctx)
    mr, _, err := p.client.MergeRequests.CreateMergeRequest(pid, &gitlab.CreateMergeRequestOptions{
        Title:              gitlab.Ptr(opts.Title),
        Description:        gitlab.Ptr(opts.Body),
        SourceBranch:       gitlab.Ptr(opts.HeadBranch),
        TargetBranch:       gitlab.Ptr(opts.BaseBranch),
        RemoveSourceBranch: gitlab.Ptr(true),
        Labels:             &opts.Labels,
        AssigneeIDs:        toAssigneeIDs(opts.Reviewers),
    })
    if err != nil {
        return nil, err
    }
    return &PRResult{
        ID:     int64(mr.IID),
        Number: mr.IID,
        URL:    mr.WebURL,
        Title:  mr.Title,
        Status: mr.State,
        Merged: mr.State == "merged",
        CreatedAt: *mr.CreatedAt,
        UpdatedAt: *mr.UpdatedAt,
    }, nil
}

func (p *GitLabProvider) GetPR(ctx context.Context, repoID string, mrIID int) (*PRResult, error) {
    pid := getProjectID(ctx)
    mr, _, err := p.client.MergeRequests.GetMergeRequest(pid, mrIID, nil)
    if err != nil {
        return nil, err
    }
    return &PRResult{
        ID:     int64(mr.IID),
        Number: mr.IID,
        URL:    mr.WebURL,
        Title:  mr.Title,
        Status: mr.State,
        Merged: mr.State == "merged",
        CreatedAt: *mr.CreatedAt,
        UpdatedAt: *mr.UpdatedAt,
    }, nil
}

func (p *GitLabProvider) ListPRs(ctx context.Context, repoID string, status string) ([]*PRResult, error) {
    pid := getProjectID(ctx)
    mrs, _, err := p.client.MergeRequests.ListProjectMergeRequests(pid,
        &gitlab.ListProjectMergeRequestsOptions{State: gitlab.Ptr(status)})
    if err != nil {
        return nil, err
    }
    var results []*PRResult
    for _, mr := range mrs {
        results = append(results, &PRResult{
            ID:     int64(mr.IID),
            Number: mr.IID,
            URL:    mr.WebURL,
            Title:  mr.Title,
            Status: mr.State,
            Merged: mr.State == "merged",
            CreatedAt: *mr.CreatedAt,
            UpdatedAt: *mr.UpdatedAt,
        })
    }
    return results, nil
}

func (p *GitLabProvider) MergePR(ctx context.Context, repoID string, mrIID int, method string) error {
    pid := getProjectID(ctx)
    _, _, err := p.client.MergeRequests.AcceptMergeRequest(pid, mrIID,
        &gitlab.AcceptMergeRequestOptions{
            MergeWhenPipelineSucceeds: gitlab.Ptr(true),
            Squash:                    gitlab.Ptr(method == "squash"),
        })
    return err
}

// ── Webhook ──

func (p *GitLabProvider) RegisterWebhook(ctx context.Context, repoID, callbackURL, secret string) error {
    pid := getProjectID(ctx)
    _, _, err := p.client.Projects.AddProjectHook(pid, &gitlab.AddProjectHookOptions{
        URL:                 gitlab.Ptr(callbackURL),
        Token:               gitlab.Ptr(secret),
        MergeRequestsEvents: gitlab.Ptr(true),
        PushEvents:          gitlab.Ptr(false),
    })
    return err
}

func (p *GitLabProvider) UnregisterWebhook(ctx context.Context, repoID string, webhookID int64) error {
    pid := getProjectID(ctx)
    _, err := p.client.Projects.DeleteProjectHook(pid, int(webhookID))
    return err
}

func (p *GitLabProvider) ParseWebhookPayload(r *http.Request) (*WebhookEvent, error) {
    payload, _ := io.ReadAll(r.Body)
    event, err := gitlab.ParseWebhook(gitlab.EventType(r.Header.Get("X-Gitlab-Event")), payload)
    if err != nil {
        return nil, err
    }
    if mrEvent, ok := event.(*gitlab.MergeEvent); ok {
        return &WebhookEvent{
            Platform:     "gitlab",
            Event:        "merge_request",
            Action:       mrEvent.ObjectAttributes.Action,
            RepoFullName: mrEvent.Project.PathWithNamespace,
            PRNumber:     mrEvent.ObjectAttributes.IID,
            PRMerged:     mrEvent.ObjectAttributes.State == "merged",
            HeadRef:      mrEvent.ObjectAttributes.SourceBranch,
        }, nil
    }
    return nil, fmt.Errorf("unsupported event")
}
```

## 8.9 Gitea Provider（go-gitea SDK）

```go
// internal/adapter/git/gitea.go

import "code.gitea.io/sdk/gitea"

type GiteaProvider struct {
    client  *gitea.Client
    baseURL string
}

func NewGiteaProvider(accessToken, baseURL string) *GiteaProvider {
    client, _ := gitea.NewClient(baseURL,
        gitea.SetToken(accessToken),
        gitea.SetHTTPClient(&http.Client{Timeout: 30 * time.Second}),
    )
    return &GiteaProvider{client: client, baseURL: baseURL}
}

func (p *GiteaProvider) ValidateCredentials(ctx context.Context, _, _ string) error {
    _, _, err := p.client.GetMyUserInfo()
    return err
}

func (p *GiteaProvider) CreateBranch(ctx context.Context, opts BranchOptions) error {
    owner, repo := parseOwnerRepo(ctx)
    _, _, err := p.client.CreateBranch(owner, repo, gitea.CreateBranchOption{
        BranchName:    opts.Name,
        OldBranchName: opts.BaseBranch,
    })
    return err
}

// ── PR 操作 ──

func (p *GiteaProvider) CreatePR(ctx context.Context, opts PROptions) (*PRResult, error) {
    owner, repo := parseOwnerRepo(ctx)
    pr, _, err := p.client.CreatePullRequest(owner, repo, gitea.CreatePullRequestOption{
        Title: opts.Title,
        Body:  opts.Body,
        Head:  opts.HeadBranch,
        Base:  opts.BaseBranch,
        Labels: opts.Labels,
    })
    if err != nil {
        return nil, err
    }
    return &PRResult{
        ID:        pr.ID,
        Number:    int(pr.Index),
        URL:       pr.HTMLURL,
        Title:     pr.Title,
        Status:    string(pr.State),
        Merged:    pr.HasMerged,
        Mergeable: pr.Mergeable,
        CreatedAt: pr.Created,
        UpdatedAt: pr.Updated,
    }, nil
}

func (p *GiteaProvider) GetPR(ctx context.Context, repoID string, prIndex int) (*PRResult, error) {
    owner, repo := parseOwnerRepo(ctx)
    pr, _, err := p.client.GetPullRequest(owner, repo, int64(prIndex))
    if err != nil {
        return nil, err
    }
    return &PRResult{
        ID:     pr.ID,
        Number: int(pr.Index),
        URL:    pr.HTMLURL,
        Title:  pr.Title,
        Status: string(pr.State),
        Merged: pr.HasMerged,
        CreatedAt: pr.Created,
        UpdatedAt: pr.Updated,
    }, nil
}

func (p *GiteaProvider) ListPRs(ctx context.Context, repoID string, status string) ([]*PRResult, error) {
    owner, repo := parseOwnerRepo(ctx)
    prs, _, err := p.client.ListRepoPullRequests(owner, repo,
        gitea.ListPullRequestsOptions{State: gitea.StateType(status)})
    if err != nil {
        return nil, err
    }
    var results []*PRResult
    for _, pr := range prs {
        results = append(results, &PRResult{
            ID:     pr.ID,
            Number: int(pr.Index),
            URL:    pr.HTMLURL,
            Title:  pr.Title,
            Status: string(pr.State),
            Merged: pr.HasMerged,
            CreatedAt: pr.Created,
            UpdatedAt: pr.Updated,
        })
    }
    return results, nil
}

func (p *GiteaProvider) MergePR(ctx context.Context, repoID string, prIndex int, method string) error {
    owner, repo := parseOwnerRepo(ctx)
    _, _, err := p.client.MergePullRequest(owner, repo, int64(prIndex),
        gitea.MergePullRequestOption{
            Style:   gitea.MergeStyle(method),
            Title:   "auto-merged by flowcode",
        })
    return err
}

// ── Webhook ──

func (p *GiteaProvider) RegisterWebhook(ctx context.Context, repoID, callbackURL, secret string) error {
    owner, repo := parseOwnerRepo(ctx)
    _, _, err := p.client.CreateRepoHook(owner, repo, gitea.CreateHookOption{
        Type:   "gitea",
        Config: map[string]string{
            "url":          callbackURL,
            "content_type": "json",
            "secret":       secret,
        },
        Events: []string{"pull_request"},
        Active: true,
    })
    return err
}

func (p *GiteaProvider) UnregisterWebhook(ctx context.Context, repoID string, webhookID int64) error {
    owner, repo := parseOwnerRepo(ctx)
    _, err := p.client.DeleteRepoHook(owner, repo, webhookID)
    return err
}

func (p *GiteaProvider) ParseWebhookPayload(r *http.Request) (*WebhookEvent, error) {
    payload, _ := io.ReadAll(r.Body)
    event, err := gitea.ParseWebhook(gitea.HookEvent(r.Header.Get("X-Gitea-Event")), payload)
    if err != nil {
        return nil, err
    }
    if prEvent, ok := event.(*gitea.PullRequestPayload); ok {
        return &WebhookEvent{
            Platform:     "gitea",
            Event:        "pull_request",
            Action:       string(prEvent.Action),
            RepoFullName: prEvent.Repository.FullName,
            PRNumber:     int(prEvent.Index),
            PRMerged:     prEvent.PullRequest.HasMerged,
            HeadRef:      prEvent.PullRequest.Head.Name,
        }, nil
    }
    return nil, fmt.Errorf("unsupported event")
}
```

## 8.10 GitProvider 工厂

```go
// internal/adapter/git/factory.go

type ProviderType string

const (
    ProviderGitHub ProviderType = "github"
    ProviderGitLab ProviderType = "gitlab"
    ProviderGitea  ProviderType = "gitea"
)

func NewGitProvider(pt ProviderType, accessToken, baseURL string) (GitProvider, error) {
    switch pt {
    case ProviderGitHub:
        return NewGitHubProvider(accessToken), nil
    case ProviderGitLab:
        return NewGitLabProvider(accessToken, baseURL), nil
    case ProviderGitea:
        return NewGiteaProvider(accessToken, baseURL), nil
    default:
        return nil, fmt.Errorf("unsupported provider: %s", pt)
    }
}
```

## 8.11 Webhook 统一路由

Git 平台的 Webhook 用于感知 PR 合并/关闭事件：

### 多 Provider 路由

```
POST /api/v1/git/webhook/github  ← GitHub 回调
POST /api/v1/git/webhook/gitlab  ← GitLab 回调
POST /api/v1/git/webhook/gitea   ← Gitea 回调
```

### 统一处理器

```go
// internal/handler/git_webhook.go

func (h *GitHandler) HandleWebhook(provider ProviderType) gin.HandlerFunc {
    return func(c *gin.Context) {
        p := h.registry.Get(provider)
        if p == nil {
            c.JSON(400, gin.H{"error": "unknown provider"})
            return
        }

        event, err := p.ParseWebhookPayload(c.Request)
        if err != nil {
            // 非 PR 事件（如 issues、issue_comment 等），接收但不处理
            // Issue 导入为手动操作，不做自动同步
            if strings.Contains(err.Error(), "unsupported") {
                c.JSON(200, gin.H{"received": true})
                return
            }
            c.JSON(400, gin.H{"error": err.Error()})
            return
        }

        // 根据 PR 状态触发 Issue 状态流转
        switch {
        case event.Action == "closed" && event.PRMerged:
            h.onPRMerged(c.Request.Context(), event)
        case event.Action == "closed" && !event.PRMerged:
            h.onPRClosedWithoutMerge(c.Request.Context(), event)
        }

        c.JSON(200, gin.H{"received": true})
    }
}

func (h *GitHandler) onPRMerged(ctx context.Context, event *WebhookEvent) {
    issue, _ := h.svc.FindIssueByRepoAndBranch(ctx, event.RepoFullName, event.HeadRef)
    if issue != nil {
        h.issueSvc.Transition(ctx, issue.ID, "deploy")
        h.issueSvc.Transition(ctx, issue.ID, "close")
        h.hookRegistry.Emit(ctx, "issue:pr:merged", map[string]any{
            "issue":    issue,
            "platform": event.Platform,
            "prNumber": event.PRNumber,
        })
    }
}

func (h *GitHandler) onPRClosedWithoutMerge(ctx context.Context, event *WebhookEvent) {
    issue, _ := h.svc.FindIssueByRepoAndBranch(ctx, event.RepoFullName, event.HeadRef)
    if issue != nil {
        h.issueSvc.Transition(ctx, issue.ID, "reopen")
    }
}
```

## 8.12 认证方式

| 平台 | 推荐方式 | 配置方式 |
|------|---------|---------|
| GitHub | Fine-grained PAT | Settings > Developer settings > PAT |
| Gitea | Access Token | Settings > Applications > Generate Token |
| GitHub (OAuth) | flowcode OAuth App | 后续版本支持 |

凭证在数据库中 AES-256-GCM 加密存储。加密密钥通过环境变量 `FLOWCODE_ENCRYPTION_KEY` 注入。

## 8.12b Issue 单向导入（手动）

> ⚠️ **仅手动导入，不做自动同步**。导入由用户在 Web 控制台点击按钮或执行 CLI 命令触发，不监听 Git 平台 Webhook、不设定时任务。导入后 Issue 由 flowcode 独立管理，不回写 Git 平台。

从 Git 平台（GitHub / GitLab / Gitea）将已有 Issue 单向导入 flowcode。

### 平台接口方法

```go
// internal/adapter/git/provider.go

type ListIssueOptions struct {
    State  string    // open / closed / all，默认 open
    Since  time.Time // 只拉取此时间之后的（增量导入）
    Page   int       // 分页，1-based
    PerPage int      // 每页数量，默认 30，最大 100
}

type RemoteIssue struct {
    Number    int       `json:"number"`     // 平台侧 Issue 编号
    Title     string    `json:"title"`
    Body      string    `json:"body"`       // Markdown
    State     string    `json:"state"`      // open / closed
    Labels    []string  `json:"labels"`
    Assignee  string    `json:"assignee"`   // 用户名
    Milestone string    `json:"milestone"`
    URL       string    `json:"url"`        // https://github.com/.../issues/123
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}
```

`GitProvider` 主接口已包含 `ListIssues` / `GetIssue`，这里仅补充导入相关参数与返回结构。

### 字段映射

| Git 平台字段 | flowcode Issue 字段 | 映射规则 |
|-------------|-------------------|---------|
| Number | sourceIssueId | 平台原始编号 |
| Title | title | 直接映射 |
| Body | description | Markdown 原样保留 |
| State=open | status | → `draft`（待评审） |
| State=closed | status | → `done` |
| Labels[0] | category | 尝试匹配 keyword 规则，未命中则留空待 AI 分类 |
| Labels | tags | 全部保留为 JSON 数组 |
| Assignee | assigneeId | 按用户名/邮箱匹配本地用户，未匹配则为空 |
| URL | sourceIssueUrl | 原始链接 |
| — | sourcePlatform | `github` / `gitlab` / `gitea` |
| CreatedAt | createdAt | 保留原始时间 |

### 去重策略

按 `projectId + sourcePlatform + sourceIssueId` 唯一约束去重：

```sql
CREATE UNIQUE INDEX idx_issue_source
ON issues(project_id, source_platform, source_issue_id)
WHERE source_platform IS NOT NULL;
```

- 已导入的 Issue 再次触发导入时**跳过**（幂等）
- 不更新已导入 Issue 的内容（单向导入，后续由 flowcode 独立管理）

### API

```
POST   /api/v1/projects/:pid/git/import-issues  # 触发导入
GET    /api/v1/projects/:pid/git/import-status   # 查询导入结果
```

### 导入流程

```go
// internal/service/git/issue_importer.go

func (s *IssueImporter) Import(ctx context.Context, projectID string, opts ImportOptions) (*ImportResult, error) {
    // 1. 获取项目和凭证
    project, _ := s.projectRepo.GetByID(ctx, projectID)
    cred, _ := s.credRepo.GetByID(ctx, project.GitCredentialId)

    // 2. 创建 GitProvider
    provider, _ := NewGitProvider(cred.Provider, cred.AccessToken, "")

    // 3. 分页拉取远程 Issue 列表
    var imported, skipped int
    for page := 1; ; page++ {
        issues, _ := provider.ListIssues(ctx, project.GitRepoId, ListIssueOptions{
            State:   opts.State,   // open / all
            Since:   opts.Since,   // 增量导入用
            Page:    page,
            PerPage: 100,
        })
        if len(issues) == 0 { break }

        for _, ri := range issues {
            // 4. 去重检查
            exists, _ := s.issueRepo.ExistsBySource(ctx, projectID, cred.Provider, ri.Number)
            if exists { skipped++; continue }

            // 5. 字段映射 + 创建
            issue := s.mapToIssue(project, ri, cred.Provider)
            s.issueRepo.Create(ctx, issue)
            imported++
        }
    }

    return &ImportResult{Imported: imported, Skipped: skipped}, nil
}
```

### 导入结果

```json
{
  "code": 0,
  "data": {
    "imported": 12,
    "skipped": 3,
    "total": 15,
    "details": [
      { "number": 1, "title": "Add login page", "action": "imported", "flowcodeId": "iss_xxx" },
      { "number": 2, "title": "Fix navbar",    "action": "skipped",  "reason": "already_exists" }
    ]
  },
  "message": "ok"
}
```

### CLI

```bash
# 全量导入（open 状态）
flowcode project import-issues --state open

# 增量导入（仅导入指定日期之后的）
flowcode project import-issues --since 2026-01-01

# 导入所有状态（含已关闭的）
flowcode project import-issues --state all
```

> **注意事项**：
> - 导入前需先配置 Git 凭证并绑定到项目
> - Issue 标签中的分类关键词命中规则时自动设置 category，未命中则留空由后续 AI 分类
> - 导入不自动触发 opencode 前置分析与执行计划生成（避免大量调用），可手动批量触发
> - GitLab 的 Issue 在 SDK 中对应 `gitlab.Issue`（非 MR），Gitea 同理

## 8.13 安全考虑

1. **最小权限原则**：Git PAT 仅需 `repo` 和 `pull_requests` 权限
2. **凭证加密**：数据库存储加密，日志脱敏
3. **操作审计**：所有 Git 操作自动写入 `audit_logs` 表（action=`git.*`），可追溯
4. **分支保护**：AI 创建的分支带有 `flowcode/` 前缀，不触碰 `main`/`master`
5. **可回滚**：PR 关闭不合并时，Issue 触发 `reopen`，随后经 `reopened` 自动回到 `approved` 入口
6. **SSH 密钥安全**：
   - 私钥仅存储在数据库中（AES-256-GCM 加密），使用后立即清理临时文件
   - `GIT_TERMINAL_PROMPT=0` 禁止任何交互式密码输入
   - 临时密钥文件权限 `0600`，操作完成后通过 defer 删除
7. **Git URL 安全**：
   - 仅允许 HTTP/HTTPS 和 SSH 协议，禁止 `file://`、`git://` 等
   - 校验时设置 15 秒超时，防止 SSRF 和内网探测
   - URL 白名单域名校验（可选：限制仅允许 github.com / gitea.example.com 等）
8. **沙盒隔离**：Clone 的代码仅存在于容器/临时目录，Issue 完成后清理
