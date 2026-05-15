# 桌面端发布与页面结构

### 3.6 desktop-release.yml — 桌面端发布

> 完整工作流见「Tauri 桌面端补充说明 → GitHub Actions 自动发布」章节。此处摘要关键步骤：

| 步骤 | 说明 |
|------|------|
| 触发 | tag `desktop-v*` |
| 矩阵构建 | windows/macos-x64/macos-arm64/linux |
| 签名 | `TAURI_SIGNING_PRIVATE_KEY` 环境变量 |
| 产物 | MSI / DMG / AppImage + `latest.json` |
| 公证 | macOS 需 Apple Developer 证书 |

### 3.7 GitHub Secrets 清单

所有 CI/CD Secrets 需在 `Repository Settings → Secrets and variables → Actions` 中配置：

| Secret | 用途 | 触发工作流 |
|--------|------|-----------|
| `GITHUB_TOKEN` | 自动提供，无需手动配置 | 所有 |
| `TAURI_PRIVATE_KEY` | Tauri updater 签名私钥 | `desktop-release` |
| `TAURI_KEY_PASSWORD` | 私钥密码（可选） | `desktop-release` |
| `APPLE_CERTIFICATE` | macOS 签名证书 (base64) | `desktop-release` |
| `APPLE_CERTIFICATE_PASSWORD` | 证书密码 | `desktop-release` |
| `APPLE_ID` | Apple Developer 账号 | `desktop-release` |
| `APPLE_PASSWORD` | App 专用密码 | `desktop-release` |
| `APPLE_TEAM_ID` | Team ID | `desktop-release` |

### 3.8 分支保护规则

在 GitHub Repository Settings → Branches 中配置 `main` 分支保护：

| 规则 | 值 |
|------|-----|
| Require a pull request before merging | ✅ |
| Require approvals | 1 |
| Require status checks to pass | ✅ `ci.yml` (go / admin / desktop) |
| Require conversation resolution | ✅ |
| Do not allow bypassing | ✅ (包括 admins) |

---

### 11.2 客户端（Tauri）

PC 桌面 + Android + iOS 共用 Next.js 前端，Tauri 打包：

- **WebView 内嵌** Next.js 构建产物
- **核心路由**：`/dashboard`、`/projects/:id`、`/chat`、`/invite/:token`
- **原生能力**：系统通知、文件系统、Git 代理（Tauri Command）
- **分阶段交付**：先桌面端，移动端作为 Phase 2

---
