# =============================================================================
# AnserFlow Agent Sandbox Dockerfile
# 目标：最小体积的 Agent 代码执行沙箱
#
# 构建:  docker build -t anserflow/sandbox:latest -f docker/sandbox/Dockerfile .
# 运行:  docker run --rm -v $(pwd):/workspace anserflow/sandbox:latest
# =============================================================================

# ── Stage 1: 编译 opencode CLI（Go 二进制） ─────────────────────────────────
FROM golang:1.24-alpine AS builder

# 如果 opencode 是独立的 Go 项目，取消注释以下：
# WORKDIR /src
# COPY cmd/opencode/ .
# RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /opencode .

# 如果 opencode 是预编译下载的，直接 COPY（跳过此阶段）
# 此处作为占位，实际使用时替换为真实的 opencode 构建/下载逻辑
RUN echo -e '#!/bin/sh\necho "opencode placeholder"' > /opencode && chmod +x /opencode

# ── Stage 2: 最小运行时 ───────────────────────────────────────────────────
FROM alpine:3.21

LABEL org.opencontainers.image.title="AnserFlow Agent Sandbox"
LABEL org.opencontainers.image.description="Minimal sandbox for AI agent code execution"
LABEL org.opencontainers.image.source="https://github.com/anserflow/anserflow"

# 一次性安装 + 清理，最小化层大小
RUN apk add --no-cache \
    # ── 基础工具 ──
    bash=5.2.37-r0 \
    curl=8.12.1-r1 \
    ca-certificates=20240705-r0 \
    tzdata=2025a-r0 \
    # ── Git ──
    git=2.47.2-r0 \
    git-lfs=3.6.1-r0 \
    openssh-client=9.9_p2-r0 \
    # ── Node.js LTS ──
    nodejs=22.14.0-r0 \
    npm \
    # ── Python 3 ──
    python3=3.12.9-r0 \
    py3-pip=24.8-r0 \
    # ── 编译依赖（C/C++ native addon 等）──
    gcc=14.2.0-r6 \
    g++=14.2.0-r6 \
    make=4.4.1-r2 \
    # ── 常用 CLI 工具 ──
    jq=1.7.1-r0 \
    ripgrep=14.1.1-r0 \
    && \
    # 清理 pip 缓存
    pip cache purge && \
    # 清理 apk 缓存
    rm -rf /var/cache/apk/* /tmp/*

# ── 从 builder 复制 opencode CLI ──
COPY --from=builder /opencode /usr/local/bin/opencode

# ── 预装常用 Python 包（可选，按需添加）──
# 如果 Agent 常用这些库，预装可减少首次执行等待时间
RUN pip install --no-cache-dir \
    requests \
    httpx \
    && pip cache purge

# ── 全局 npm 包（可选）──
RUN npm install -g \
    typescript \
    tsx \
    && npm cache clean --force

# ── 安全加固 ──
# 创建非 root 用户运行 Agent 代码
RUN addgroup -g 1000 sandbox && \
    adduser -D -u 1000 -G sandbox sandbox && \
    # 移除其他用户的写权限（保留执行权限，确保 sandbox 用户可运行命令）
    chmod -R go-w /usr/local/bin /usr/bin

# ── 工作目录 ──
WORKDIR /workspace
RUN chown sandbox:sandbox /workspace

# 切换到非 root 用户
USER sandbox

# ── 环境变量 ──
ENV LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    TZ=Asia/Shanghai \
    NODE_ENV=production \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    # Git 安全目录（挂载的 /workspace 需要）
    GIT_CONFIG_GLOBAL=/tmp/gitconfig

# ── Git 初始化配置 ──
RUN git config --global --add safe.directory /workspace && \
    git config --global init.defaultBranch main

# ── 健康检查 ──
HEALTHCHECK --interval=30s --timeout=3s --retries=1 CMD node -e "1+1" || exit 1

# ── 入口：由 Go Docker SDK 动态指定 ──
# 示例: ["opencode", "run", "--task", "{...}"]
ENTRYPOINT ["/usr/local/bin/opencode"]
CMD ["--help"]
