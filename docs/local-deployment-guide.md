# Claude Context 本地化部署指南

本指南介绍如何在不依赖云服务的情况下，完全本地化部署 Claude Context。

## 目录

- [概述](#概述)
- [方案一：本地 Milvus + OpenAI 嵌入](#方案一本地-milvus--openai-嵌入)
- [方案二：完全本地化（Milvus + Ollama）](#方案二完全本地化milvus--ollama)
- [方案对比](#方案对比)
- [故障排除](#故障排除)

---

## 概述

Claude Context 默认使用 Zilliz Cloud 作为向量数据库，但它完全支持本地部署的 Milvus。本指南提供两种本地化方案：

| 方案 | 向量数据库 | 嵌入模型 | 需要云服务 | 适用场景 |
|------|-----------|---------|-----------|----------|
| 方案一 | 本地 Milvus | OpenAI API | ✅ OpenAI API | 追求嵌入质量 |
| 方案二 | 本地 Milvus | Ollama 本地模型 | ❌ 无 | 完全离线/隐私敏感 |

---

## 方案一：本地 Milvus + OpenAI 嵌入

此方案使用本地 Milvus 存储向量，但仍使用 OpenAI API 生成嵌入向量。

### 前置要求

- Docker Desktop 或 Podman
- OpenAI API Key
- Node.js 18+

### 步骤 1：启动本地 Milvus

#### 使用 Docker

```bash
# 创建并启动 Milvus 容器
docker run -d --name milvus-standalone \
  -p 19530:19530 \
  -p 9091:9091 \
  -v milvus-data:/var/lib/milvus \
  milvusdb/milvus:latest \
  milvus run standalone
```

#### 使用 Podman

Podman 命令与 Docker 几乎完全兼容，只需将 `docker` 替换为 `podman`：

```bash
# 创建并启动 Milvus 容器
podman run -d --name milvus-standalone \
  -p 19530:19530 \
  -p 9091:9091 \
  -v milvus-data:/var/lib/milvus \
  docker.io/milvusdb/milvus:latest \
  milvus run standalone
```

> 💡 **Podman 注意事项**：
> - Podman 默认以 rootless 模式运行，无需 sudo
> - 需要使用完整镜像路径 `docker.io/milvusdb/milvus:latest`
> - 如果遇到权限问题，可以添加 `--privileged` 参数
> - **Windows/macOS 用户必须先初始化并启动 Podman Machine**（见下方说明）

#### Windows/macOS 首次使用 Podman

在 Windows 和 macOS 上，Podman 需要运行一个 Linux 虚拟机。首次使用前必须初始化：

```powershell
# 1. 初始化 Podman Machine（只需执行一次，需要几分钟下载镜像）
podman machine init

# 2. 启动 Podman Machine
podman machine start

# 3. 验证连接
podman system connection list
```

> ⚠️ **注意**：每次重启电脑后，需要重新运行 `podman machine start`

**自定义虚拟机配置**（可选，用于大型代码库）：

```powershell
# 分配更多资源：4 CPU、8GB 内存、50GB 磁盘
podman machine init --cpus 4 --memory 8192 --disk-size 50
```

#### 验证 Milvus 是否启动成功

```bash
# Docker
docker ps | grep milvus

# Podman
podman ps | grep milvus

# 检查健康状态（通用）
curl http://localhost:9091/healthz
```

### 步骤 2：配置 Claude Code MCP

#### 方式 A：使用命令行配置

```bash
claude mcp add claude-context \
  -e OPENAI_API_KEY=sk-your-openai-api-key \
  -e MILVUS_ADDRESS=localhost:19530 \
  -- npx @zilliz/claude-context-mcp@latest
```

#### 方式 B：编辑配置文件

编辑 Claude Code 的 MCP 配置文件（通常位于 `~/.claude/mcp.json` 或项目根目录的 `.mcp.json`）：

```json
{
  "mcpServers": {
    "claude-context": {
      "command": "npx",
      "args": ["@zilliz/claude-context-mcp@latest"],
      "env": {
        "OPENAI_API_KEY": "sk-your-openai-api-key",
        "MILVUS_ADDRESS": "localhost:19530"
      }
    }
  }
}
```

> ⚠️ **注意**：本地 Milvus 不需要设置 `MILVUS_TOKEN`，只有 Zilliz Cloud 才需要。

### 步骤 3：验证配置

在 Claude Code 中运行：

```
> Index this codebase
```

如果看到索引进度，说明配置成功。

---

## 方案二：完全本地化（Milvus + Ollama）

此方案完全不依赖任何云服务，使用本地 Milvus + Ollama 嵌入模型。

### 前置要求

- Docker Desktop 或 Podman
- [Ollama](https://ollama.ai)
- Node.js 18+

### 步骤 1：安装并配置 Ollama

#### Windows

1. 从 [ollama.ai](https://ollama.ai) 下载并安装 Ollama
2. 安装完成后，Ollama 会自动在后台运行

#### macOS

```bash
brew install ollama
ollama serve  # 启动服务
```

#### Linux

```bash
curl -fsSL https://ollama.ai/install.sh | sh
ollama serve  # 启动服务
```

### 步骤 2：拉取嵌入模型

```bash
# 推荐：通用嵌入模型
ollama pull nomic-embed-text

# 可选：更大更准确的模型（需要更多内存）
ollama pull mxbai-embed-large

# 可选：专为代码优化的模型
ollama pull codellama:7b
```

**模型对比：**

| 模型名称 | 向量维度 | 内存需求 | 特点 |
|---------|---------|---------|------|
| `nomic-embed-text` | 768 | ~500MB | 轻量，通用 |
| `mxbai-embed-large` | 1024 | ~1.2GB | 更准确 |
| `snowflake-arctic-embed` | 1024 | ~1GB | 检索优化 |

### 步骤 3：启动本地 Milvus

#### 使用 Docker

```bash
docker run -d --name milvus-standalone \
  -p 19530:19530 \
  -p 9091:9091 \
  -v milvus-data:/var/lib/milvus \
  milvusdb/milvus:latest \
  milvus run standalone
```

#### 使用 Podman

```bash
podman run -d --name milvus-standalone \
  -p 19530:19530 \
  -p 9091:9091 \
  -v milvus-data:/var/lib/milvus \
  docker.io/milvusdb/milvus:latest \
  milvus run standalone
```

### 步骤 4：配置 Claude Code MCP

#### 方式 A：使用命令行配置

```bash
claude mcp add claude-context \
  -e EMBEDDING_PROVIDER=Ollama \
  -e OLLAMA_MODEL=nomic-embed-text \
  -e OLLAMA_HOST=http://127.0.0.1:11434 \
  -e MILVUS_ADDRESS=localhost:19530 \
  -- npx @zilliz/claude-context-mcp@latest
```

#### 方式 B：编辑配置文件

```json
{
  "mcpServers": {
    "claude-context": {
      "command": "npx",
      "args": ["@zilliz/claude-context-mcp@latest"],
      "env": {
        "EMBEDDING_PROVIDER": "Ollama",
        "OLLAMA_MODEL": "nomic-embed-text",
        "OLLAMA_HOST": "http://127.0.0.1:11434",
        "MILVUS_ADDRESS": "localhost:19530"
      }
    }
  }
}
```

### 步骤 5：验证配置

```bash
# 1. 确认 Ollama 运行中
curl http://127.0.0.1:11434/api/tags

# 2. 确认 Milvus 运行中
curl http://localhost:9091/healthz

# 3. 在 Claude Code 中测试
> Index this codebase
```

---

## 方案对比

### 性能对比

| 指标 | 方案一 (OpenAI) | 方案二 (Ollama) |
|------|----------------|-----------------|
| 嵌入质量 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 索引速度 | 受网络影响 | 受 CPU/GPU 影响 |
| 隐私性 | 代码发送到 OpenAI | 完全本地 |
| 成本 | 按 Token 收费 | 免费 |
| 离线使用 | ❌ | ✅ |

### 推荐选择

- **选择方案一**：如果你追求最佳嵌入质量，且代码不涉及敏感信息
- **选择方案二**：如果你需要完全离线使用，或处理敏感代码

---

## 环境变量参考

| 变量名 | 说明 | 默认值 | 示例 |
|--------|------|--------|------|
| `MILVUS_ADDRESS` | Milvus 服务地址 | (Zilliz Cloud) | `localhost:19530` |
| `MILVUS_TOKEN` | Milvus 认证令牌 | - | 本地部署不需要 |
| `EMBEDDING_PROVIDER` | 嵌入模型提供者 | `OpenAI` | `Ollama`, `VoyageAI`, `Gemini` |
| `OPENAI_API_KEY` | OpenAI API 密钥 | - | `sk-...` |
| `OLLAMA_HOST` | Ollama 服务地址 | `http://127.0.0.1:11434` | - |
| `OLLAMA_MODEL` | Ollama 嵌入模型 | `nomic-embed-text` | `mxbai-embed-large` |

---

## 故障排除

### 问题 1：无法连接到 Milvus

**症状**：`Connection refused` 或 `ECONNREFUSED`

**解决方案**：

```bash
# 检查 Milvus 容器是否运行
docker ps | grep milvus

# 如果没有运行，查看日志
docker logs milvus-standalone

# 重启容器
docker restart milvus-standalone
```

### 问题 2：Ollama 模型未找到

**症状**：`model not found` 错误

**解决方案**：

```bash
# 确认模型已下载
ollama list

# 如果没有，重新拉取
ollama pull nomic-embed-text
```

### 问题 3：索引速度很慢

**可能原因**：
- 使用 Ollama 时 CPU 性能不足
- 代码库文件过多

**解决方案**：
- 使用 GPU 加速 Ollama（需要 NVIDIA GPU）
- 在 `.gitignore` 中排除不需要索引的目录
- 创建 `.contextignore` 文件排除特定文件

### 问题 4：Windows 上 Docker 网络问题

**症状**：`localhost` 无法访问

**解决方案**：

```bash
# 尝试使用 host.docker.internal
# 或者查找容器 IP
docker inspect milvus-standalone | grep IPAddress
```

### 问题 5：Podman 特有问题

#### 5.0 无法连接到 Podman（Windows/macOS）

**症状**：
```
Cannot connect to Podman. Please verify your connection to the Linux system using `podman system connection list`, or try `podman machine init` and `podman machine start` to manage a new Linux VM
Error: unable to connect to Podman socket: failed to connect
```

**原因**：Podman Machine 未初始化或未启动

**解决方案**：

```powershell
# 步骤 1：初始化 Podman Machine（首次使用，只需执行一次）
podman machine init

# 步骤 2：启动 Podman Machine
podman machine start

# 步骤 3：验证连接
podman system connection list
podman info
```

**如果 `podman machine init` 失败**：

```powershell
# 检查是否已存在
podman machine list

# 如果存在但状态异常，删除后重新创建
podman machine rm podman-machine-default
podman machine init

# 如果下载缓慢，可以使用代理
$env:HTTP_PROXY = "http://your-proxy:port"
$env:HTTPS_PROXY = "http://your-proxy:port"
podman machine init
```

**如果 `podman machine start` 失败**：

```powershell
# 检查 WSL 状态（Windows）
wsl --status

# 如果 WSL 未安装或版本过低
wsl --install
wsl --update

# 重启后再试
podman machine start
```

#### 5.0.1 Machine 显示运行中但无法连接（Windows）

**症状**：
```
PS> podman machine start
Error: unable to start "podman-machine-default": already running

PS> podman run ...
Cannot connect to Podman. Please verify your connection to the Linux system...
Error: unable to connect to Podman socket: failed to connect: dial tcp 127.0.0.1:8206: connectex: No connection could be made because the target machine actively refused it.
```

**原因**：Podman Machine 与 WSL 状态不同步，通常是因为 WSL 被单独启动或关闭导致的。

**解决方案**：

```powershell
# 方案 A：重启 WSL 和 Podman Machine（推荐）
wsl --shutdown
podman machine stop
podman machine start

# 方案 B：完全重置（如果方案 A 无效）
wsl --shutdown
podman machine stop
podman machine rm podman-machine-default -f
podman machine init
podman machine start

# 验证
podman info
```

**预防措施**：
- 避免在 Podman Machine 运行时手动操作 WSL
- 如果需要关闭 WSL，先运行 `podman machine stop`
- 建议将 `podman machine start` 添加到开机启动脚本

#### 5.1 镜像拉取失败

**症状**：`Error: short-name resolution enforced`

**解决方案**：

```bash
# 使用完整的镜像路径
podman pull docker.io/milvusdb/milvus:latest

# 或者配置 unqualified-search-registries
# 编辑 /etc/containers/registries.conf (Linux)
# 或 ~/.config/containers/registries.conf
# 添加: unqualified-search-registries = ["docker.io"]
```

#### 5.2 Rootless 模式下端口绑定失败

**症状**：`Error: rootlessport cannot expose privileged port 19530`

**解决方案**：

```bash
# 方案 A：使用更高的端口号
podman run -d --name milvus-standalone \
  -p 19530:19530 \
  -p 9091:9091 \
  docker.io/milvusdb/milvus:latest \
  milvus run standalone

# 方案 B：允许非 root 用户绑定低端口（Linux）
sudo sysctl net.ipv4.ip_unprivileged_port_start=0
```

#### 5.3 Podman Machine 网络问题（macOS/Windows）

**症状**：容器内服务无法从主机访问

**解决方案**：

```bash
# 确保 Podman Machine 正在运行
podman machine start

# 检查端口转发是否正确
podman machine ssh -- ss -tlnp | grep 19530

# 如果使用 WSL2 (Windows)，可能需要配置端口转发
# 或者使用 podman machine 的 IP 地址
podman machine inspect | grep IPAddress
```

#### 5.4 SELinux 权限问题（RHEL/Fedora）

**症状**：`Permission denied` 访问卷

**解决方案**：

```bash
# 添加 :Z 标签让 Podman 自动处理 SELinux 上下文
podman run -d --name milvus-standalone \
  -p 19530:19530 \
  -p 9091:9091 \
  -v milvus-data:/var/lib/milvus:Z \
  docker.io/milvusdb/milvus:latest \
  milvus run standalone
```

---

## 快速启动脚本

### Windows (PowerShell)

创建 `start-local-context.ps1`：

```powershell
# 启动 Milvus
$milvusRunning = docker ps --filter "name=milvus-standalone" --format "{{.Names}}"
if (-not $milvusRunning) {
    Write-Host "Starting Milvus..."
    docker start milvus-standalone 2>$null
    if ($LASTEXITCODE -ne 0) {
        docker run -d --name milvus-standalone `
            -p 19530:19530 `
            -p 9091:9091 `
            -v milvus-data:/var/lib/milvus `
            milvusdb/milvus:latest `
            milvus run standalone
    }
}

# 确认 Ollama 运行中
$ollamaRunning = Get-Process ollama -ErrorAction SilentlyContinue
if (-not $ollamaRunning) {
    Write-Host "Please start Ollama manually"
}

Write-Host "Local Claude Context environment is ready!"
Write-Host "Milvus: localhost:19530"
Write-Host "Ollama: localhost:11434"
```

### macOS/Linux (Bash)

创建 `start-local-context.sh`：

```bash
#!/bin/bash

# 启动 Milvus
if ! docker ps | grep -q milvus-standalone; then
    echo "Starting Milvus..."
    docker start milvus-standalone 2>/dev/null || \
    docker run -d --name milvus-standalone \
        -p 19530:19530 \
        -p 9091:9091 \
        -v milvus-data:/var/lib/milvus \
        milvusdb/milvus:latest \
        milvus run standalone
fi

# 启动 Ollama（如果未运行）
if ! pgrep -x "ollama" > /dev/null; then
    echo "Starting Ollama..."
    ollama serve &
    sleep 2
fi

echo "Local Claude Context environment is ready!"
echo "Milvus: localhost:19530"
echo "Ollama: localhost:11434"
```

### Windows (PowerShell) - Podman 版本

创建 `start-local-context-podman.ps1`：

```powershell
# 确保 Podman Machine 正在运行
$machineRunning = podman machine inspect 2>$null
if ($LASTEXITCODE -ne 0) {
    Write-Host "Starting Podman Machine..."
    podman machine start
}

# 启动 Milvus
$milvusRunning = podman ps --filter "name=milvus-standalone" --format "{{.Names}}"
if (-not $milvusRunning) {
    Write-Host "Starting Milvus..."
    podman start milvus-standalone 2>$null
    if ($LASTEXITCODE -ne 0) {
        podman run -d --name milvus-standalone `
            -p 19530:19530 `
            -p 9091:9091 `
            -v milvus-data:/var/lib/milvus `
            docker.io/milvusdb/milvus:latest `
            milvus run standalone
    }
}

# 确认 Ollama 运行中
$ollamaRunning = Get-Process ollama -ErrorAction SilentlyContinue
if (-not $ollamaRunning) {
    Write-Host "Please start Ollama manually"
}

Write-Host "Local Claude Context environment is ready!"
Write-Host "Milvus: localhost:19530"
Write-Host "Ollama: localhost:11434"
```

### macOS/Linux (Bash) - Podman 版本

创建 `start-local-context-podman.sh`：

```bash
#!/bin/bash

# 检测容器运行时
CONTAINER_CMD="podman"

# 确保 Podman Machine 正在运行（macOS）
if [[ "$OSTYPE" == "darwin"* ]]; then
    if ! podman machine inspect &>/dev/null; then
        echo "Starting Podman Machine..."
        podman machine start
    fi
fi

# 启动 Milvus
if ! $CONTAINER_CMD ps | grep -q milvus-standalone; then
    echo "Starting Milvus..."
    $CONTAINER_CMD start milvus-standalone 2>/dev/null || \
    $CONTAINER_CMD run -d --name milvus-standalone \
        -p 19530:19530 \
        -p 9091:9091 \
        -v milvus-data:/var/lib/milvus:Z \
        docker.io/milvusdb/milvus:latest \
        milvus run standalone
fi

# 启动 Ollama（如果未运行）
if ! pgrep -x "ollama" > /dev/null; then
    echo "Starting Ollama..."
    ollama serve &
    sleep 2
fi

echo "Local Claude Context environment is ready!"
echo "Milvus: localhost:19530"
echo "Ollama: localhost:11434"
```

---

## Docker vs Podman 对比

| 特性 | Docker | Podman |
|------|--------|--------|
| 守护进程 | 需要 Docker Daemon | 无守护进程（daemonless） |
| 权限模式 | 默认需要 root | 默认 rootless |
| 兼容性 | - | 与 Docker 命令兼容 |
| 许可证 | Docker Desktop 商业使用需许可 | 完全开源免费 |
| Windows/macOS | Docker Desktop | Podman Machine |
| SELinux | 需要额外配置 | 原生支持（:Z 标签） |
| 企业环境 | 可能需要许可证 | 推荐使用 |

---

## 参考链接

- [Milvus 官方文档](https://milvus.io/docs)
- [Ollama 官方网站](https://ollama.ai)
- [Claude Context GitHub](https://github.com/zilliztech/claude-context)
- [Podman 官方文档](https://podman.io/docs)
- [Podman 安装指南](https://podman.io/getting-started/installation)
