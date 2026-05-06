# 08 - Git 集成方案

## 8.1 概述

flowcode 不内置 Git 托管，而是通过 Provider 模式对接外部 Git 平台（GitHub / Gitea / GitLab）。系统支持从远程仓库拉取代码（HTTP / SSH），供 AI 工具在本地沙盒中执行编码任务。AI 编码完成后，系统自动在对应平台创建分支、提交代码、发起 Pull Request。

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
Issue 触发执行 (approved → running)
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
│  │ 4. AI 工具执行编码        │ │
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
│         Webhook 回调 flowcode              │
│  - PR merged  → issue.status = 'deployed' │
│  - PR closed  → issue.status = 'reopened' │
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

AI 工具执行编码任务前，系统需要将目标仓库克隆到本地沙盒目录。克隆支持 HTTP 和 SSH 两种协议。

### Clone 服务实现

```go
// internal/service/git/clone_service.go

type CloneService struct {
    credentialRepo *CredentialRepository
    sandboxRoot    string // /tmp/flowcode-sandbox
}

func (s *CloneService) Clone(ctx context.Context, project *model.Project, sessionID string) (string, error) {
    // 1. 获取项目凭证
    cred, err := s.credentialRepo.GetByProject(ctx, project.ID)
    if err != nil {
        return "", fmt.Errorf("no credential configured for project %s: %w", project.ID, err)
    }

    // 2. 构建沙盒工作目录
    workDir := filepath.Join(s.sandboxRoot, sessionID, "repo")
    if err := os.MkdirAll(workDir, 0755); err != nil {
        return "", fmt.Errorf("failed to create workdir: %w", err)
    }

    // 3. 根据协议选择克隆方式
    var cloneURL string
    switch cred.AuthMethod {
    case "http":
        cloneURL = s.buildHTTPCloneURL(project.GitRepoUrl, cred.AccessToken)
    case "ssh":
        cloneURL = project.GitRepoUrl
    }

    // 4. 执行 git clone
    if err := s.runClone(ctx, cloneURL, workDir, cred); err != nil {
        return "", fmt.Errorf("clone failed: %w", err)
    }

    return workDir, nil
}

// buildHTTPCloneURL 将 token 嵌入 URL 用于免交互认证
func (s *CloneService) buildHTTPCloneURL(repoURL string, token string) string {
    // https://github.com/user/repo.git → https://token@github.com/user/repo.git
    u, _ := url.Parse(repoURL)
    u.User = url.User(token)
    return u.String()
}

// runClone 执行 git clone，支持超时和重试
func (s *CloneService) runClone(ctx context.Context, cloneURL string, workDir string, cred *model.GitCredential) error {
    var cmd *exec.Cmd

    if cred.AuthMethod == "ssh" {
        // SSH 模式：解密私钥，写入临时文件，通过 GIT_SSH_COMMAND 指定
        keyFile, cleanup, err := s.prepareSSHKey(ctx, cred)
        if err != nil {
            return err
        }
        defer cleanup()

        sshCmd := fmt.Sprintf(
            "ssh -i %s -o StrictHostKeyChecking=accept-new -o PasswordAuthentication=no",
            keyFile,
        )
        cmd = exec.CommandContext(ctx, "git", "clone", "--depth", "1", cloneURL, workDir)
        cmd.Env = append(os.Environ(),
            "GIT_TERMINAL_PROMPT=0",
            "GIT_SSH_COMMAND="+sshCmd,
        )
    } else {
        // HTTP 模式：token 已嵌入 URL
        cmd = exec.CommandContext(ctx, "git", "clone", "--depth", "1", cloneURL, workDir)
        cmd.Env = append(os.Environ(), "GIT_TERMINAL_PROMPT=0")
    }

    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    return cmd.Run()
}

// prepareSSHKey 解密 SSH 私钥并写入临时文件
func (s *CloneService) prepareSSHKey(ctx context.Context, cred *model.GitCredential) (keyFile string, cleanup func(), err error) {
    // 解密私钥
    decrypted, err := crypto.DecryptAES256GCM(cred.SSHPrivateKey, os.Getenv("FLOWCODE_ENCRYPTION_KEY"))
    if err != nil {
        return "", nil, fmt.Errorf("failed to decrypt SSH key: %w", err)
    }

    // 写入临时文件
    tmpFile, err := os.CreateTemp("", "flowcode-ssh-key-*")
    if err != nil {
        return "", nil, err
    }
    tmpFile.WriteString(decrypted)
    tmpFile.Chmod(0600)
    tmpFile.Close()

    cleanup = func() { os.Remove(tmpFile.Name()) }
    return tmpFile.Name(), cleanup, nil
}
```

### Clone 策略

| 参数 | 说明 |
|------|------|
| `--depth 1` | 浅克隆，仅拉取最新提交，减少传输量 |
| `--single-branch` | 仅克隆默认分支 |
| 超时 | 默认 120 秒，大仓库可调整 |
| 重试 | 失败自动重试 1 次 |
| 缓存 | 同一仓库 + 同一分支的克隆结果缓存在 `/tmp/flowcode-sandbox/.cache/` |

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

- Add POST /api/auth/login route
- Add JWT sign and verify utilities
- Add login rate limiting
- Add unit tests for auth flow

Closes #vf-abc123
Ref: https://flowcode.local/projects/xxx/issues/abc123
```

## 8.5 PR 自动创建

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

## 8.6 GitHub Provider 实现

```go
// internal/adapter/git/github.go

type GitHubProvider struct {
    config GitProviderConfig
}

func NewGitHubProvider() *GitHubProvider {
    return &GitHubProvider{
        config: GitProviderConfig{
            Name:        "github",
            DisplayName: "GitHub",
            AuthMethods: []string{"oauth", "pat"},
        },
    }
}

func (p *GitHubProvider) Config() GitProviderConfig { return p.config }

func (p *GitHubProvider) ValidateCredentials(ctx GitContext) error {
    client := github.NewClient(nil).WithAuthToken(ctx.AccessToken)
    _, _, err := client.Users.Get(context.Background(), "")
    return err
}

func (p *GitHubProvider) CreateBranch(ctx GitContext, opts BranchOptions) error {
    client := github.NewClient(nil).WithAuthToken(ctx.AccessToken)
    owner, repo := splitRepoID(ctx.RepoID)
    
    ref, _, err := client.Git.GetRef(context.Background(), owner, repo,
        "heads/"+opts.BaseBranch)
    if err != nil {
        return err
    }
    
    _, _, err = client.Git.CreateRef(context.Background(), owner, repo, &github.Reference{
        Ref:    github.String("refs/heads/" + opts.Name),
        Object: &github.GitObject{SHA: ref.Object.SHA},
    })
    return err
}

func (p *GitHubProvider) CreatePullRequest(ctx GitContext, opts PROptions) (*PRResult, error) {
    client := github.NewClient(nil).WithAuthToken(ctx.AccessToken)
    owner, repo := splitRepoID(ctx.RepoID)
    
    pr, _, err := client.PullRequests.Create(context.Background(), owner, repo, &github.NewPullRequest{
        Title: github.String(opts.Title),
        Body:  github.String(opts.Body),
        Head:  github.String(opts.HeadBranch),
        Base:  github.String(opts.BaseBranch),
        Draft: github.Bool(opts.Draft),
    })
    if err != nil {
        return nil, err
    }
    
    return &PRResult{
        ID:     strconv.FormatInt(pr.GetID(), 10),
        Number: pr.GetNumber(),
        URL:    pr.GetHTMLURL(),
        Status: "open",
    }, nil
}
```

## 8.7 Gitea Provider 实现

```go
// internal/adapter/git/gitea.go

type GiteaProvider struct {
    config GitProviderConfig
}

func NewGiteaProvider() *GiteaProvider {
    return &GiteaProvider{
        config: GitProviderConfig{
            Name:        "gitea",
            DisplayName: "Gitea",
            AuthMethods: []string{"pat"},
        },
    }
}

func (p *GiteaProvider) Config() GitProviderConfig { return p.config }

// getBaseURL 从 repoUrl 解析 Gitea 实例地址
// 例如 https://gitea.example.com/owner/repo → https://gitea.example.com
func (p *GiteaProvider) getBaseURL(ctx GitContext) string {
    u, _ := url.Parse(ctx.RepoURL)
    return fmt.Sprintf("%s://%s", u.Scheme, u.Host)
}

func (p *GiteaProvider) request(ctx GitContext, method, path string, body any) (*http.Response, error) {
    baseURL := p.getBaseURL(ctx)
    fullURL := fmt.Sprintf("%s/api/v1%s", baseURL, path)

    var bodyReader io.Reader
    if body != nil {
        data, _ := json.Marshal(body)
        bodyReader = bytes.NewReader(data)
    }

    req, _ := http.NewRequest(method, fullURL, bodyReader)
    req.Header.Set("Authorization", "token "+ctx.AccessToken)
    req.Header.Set("Content-Type", "application/json")

    return http.DefaultClient.Do(req)
}

func (p *GiteaProvider) CreateBranch(ctx GitContext, opts BranchOptions) error {
    parts := strings.SplitN(ctx.RepoID, "/", 2)
    owner, repo := parts[0], parts[1]

    // Gitea API: 获取 base 分支信息
    resp, err := p.request(ctx, "GET",
        fmt.Sprintf("/repos/%s/%s/branches/%s", owner, repo, opts.BaseBranch))
    if err != nil {
        return err
    }
    resp.Body.Close()

    // Gitea API: 创建新分支
    createResp, err := p.request(ctx, "POST",
        fmt.Sprintf("/repos/%s/%s/branches", owner, repo),
        map[string]string{
            "branch_name":     opts.Name,
            "old_branch_name": opts.BaseBranch,
        })
    if err != nil {
        return err
    }
    defer createResp.Body.Close()
    return nil
}

func (p *GiteaProvider) CreatePullRequest(ctx GitContext, opts PROptions) (*PRResult, error) {
    parts := strings.SplitN(ctx.RepoID, "/", 2)
    owner, repo := parts[0], parts[1]

    resp, err := p.request(ctx, "POST",
        fmt.Sprintf("/repos/%s/%s/pulls", owner, repo),
        map[string]string{
            "title": opts.Title,
            "body":  opts.Body,
            "head":  opts.HeadBranch,
            "base":  opts.BaseBranch,
        })
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var pr struct {
        ID     int    `json:"id"`
        Number int    `json:"number"`
        URL    string `json:"html_url"`
    }
    json.NewDecoder(resp.Body).Decode(&pr)

    return &PRResult{
        ID:     strconv.Itoa(pr.ID),
        Number: pr.Number,
        URL:    pr.URL,
        Status: "open",
    }, nil
}

// ValidateCredentials / CommitChanges / GetPRStatus 等方法实现类似
```

## 8.8 Webhook 回调处理

Git 平台的 Webhook 用于感知 PR 合并/关闭事件：

### GitHub Webhook 处理

```go
// internal/handler/git_webhook.go

func (h *GitHandler) HandleGitHubWebhook(c *gin.Context) {
    event := c.GetHeader("X-GitHub-Event")
    var payload map[string]any
    c.ShouldBindJSON(&payload)

    if event != "pull_request" {
        c.JSON(200, gin.H{"received": true})
        return
    }

    action := payload["action"].(string)
    pr := payload["pull_request"].(map[string]any)
    repoFullName := payload["repository"].(map[string]any)["full_name"].(string)
    headRef := pr["head"].(map[string]any)["ref"].(string)

    issue, err := h.svc.FindIssueByRepoAndBranch(c.Request.Context(), repoFullName, headRef)
    if err != nil {
        c.JSON(404, gin.H{"error": "issue not found"})
        return
    }

    merged := pr["merged"].(bool)
    if action == "closed" && merged {
        h.issueSvc.Transition(c.Request.Context(), issue.ID, "deployed")
        h.hookRegistry.Emit(c.Request.Context(), "issue:pr:merged", map[string]any{
            "issue": issue,
            "pr":    pr,
        })
    } else if action == "closed" && !merged {
        h.issueSvc.Transition(c.Request.Context(), issue.ID, "reopened")
    }

    c.JSON(200, gin.H{"received": true})
}
```

### 多 Provider Webhook 路由

```
GitHub:  POST /api/git/webhook/github
Gitea:   POST /api/git/webhook/gitea
GitLab:  POST /api/git/webhook/gitlab  (后续)
```

## 8.9 认证方式

| 平台 | 推荐方式 | 配置方式 |
|------|---------|---------|
| GitHub | Fine-grained PAT | Settings > Developer settings > PAT |
| Gitea | Access Token | Settings > Applications > Generate Token |
| GitHub (OAuth) | flowcode OAuth App | 后续版本支持 |

凭证在数据库中 AES-256-GCM 加密存储。加密密钥通过环境变量 `FLOWCODE_ENCRYPTION_KEY` 注入。

## 8.10 安全考虑

1. **最小权限原则**：Git PAT 仅需 `repo` 和 `pull_requests` 权限
2. **凭证加密**：数据库存储加密，日志脱敏
3. **操作审计**：记录所有 Git 操作日志
4. **分支保护**：AI 创建的分支带有 `flowcode/` 前缀，不触碰 `main`/`master`
5. **可回滚**：PR 关闭不合并时，Issue 回到 `reopened` 状态
6. **SSH 密钥安全**：
   - 私钥仅存储在数据库中（AES-256-GCM 加密），使用后立即清理临时文件
   - `GIT_TERMINAL_PROMPT=0` 禁止任何交互式密码输入
   - 临时密钥文件权限 `0600`，操作完成后通过 defer 删除
7. **Git URL 安全**：
   - 仅允许 HTTP/HTTPS 和 SSH 协议，禁止 `file://`、`git://` 等
   - 校验时设置 15 秒超时，防止 SSRF 和内网探测
   - URL 白名单域名校验（可选：限制仅允许 github.com / gitea.example.com 等）
8. **沙盒隔离**：Clone 的代码仅存在于容器/临时目录，Issue 完成后清理
