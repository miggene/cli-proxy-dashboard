# cli-proxy-dashboard

[中文](#中文) | [English](#english)

---

## 中文

一个将 **[CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)** 与 **[CPA-Dashboard](https://github.com/dongshuyan/CPA-Dashboard)** 整合在一起、开箱即用的 Docker Compose 部署方案。

CLIProxyAPI 提供多账户 AI 代理服务，CPA-Dashboard 提供可视化管理界面，两者通过 Docker 内部网络无缝通信。

### 📋 项目构成

| 子模块 | 技术 | 默认端口 | 说明 |
|--------|------|----------|------|
| [CLIProxyAPI](./CLIProxyAPI) | Go | `8317` | AI 代理服务核心，支持 Gemini / Antigravity / Claude / Codex / Qwen / iFlow / Kimi 多账户轮询，提供 OpenAI 兼容 API |
| [CPA-Dashboard](./CPA-Dashboard) | Python / Flask | `5000` | Web 管理面板，用于账户监控、配额刷新、OAuth 登录、服务状态查看 |

### 🏗️ 架构说明

```
宿主机                       Docker 内部网络 (cpa-network)
                            ┌─────────────────────────────────┐
:5000 ──────────────────▶  │  cpa-dashboard (Flask :5000)    │
                            │       │ http://cli-proxy-api:8317│
:8317 ──────────────────▶  │  cli-proxy-api (Go :8317)       │
:8085 / :1455 / :54545      │       │                          │
:51121 (OAuth 回调端口)     │       ▼                          │
                            │  共享 Volume: auths/ config.yaml │
                            └─────────────────────────────────┘
```

CPA-Dashboard 通过容器内部 hostname `cli-proxy-api` 与 CLIProxyAPI 通信，无需暴露 `127.0.0.1`。

---

### ✨ 功能特性

**CLIProxyAPI**
- OpenAI / Gemini / Claude / Codex 兼容 API 端点
- 多账户轮询负载均衡（Gemini、Claude、Codex、Antigravity、Qwen、iFlow、Kimi）
- OAuth 登录支持（无需手动填写 API Key）
- 流式 / 非流式响应、多模态输入、函数调用
- Management API（可选密钥保护）

**CPA-Dashboard**
- 账户列表总览（邮箱、会员等级 ULTRA/PRO/FREE、账户状态）
- 实时 / 静态配额显示，支持单账户及批量并行刷新
- OAuth 登录界面（无需命令行，点击即可添加账户）
- 失效账户批量删除
- 类型 & 状态组合筛选
- API 使用示例（cURL / Python / OpenAI SDK）
- 服务启停控制与日志查看（非 Docker 模式）

---

### 📦 前置要求

- [Docker](https://docs.docker.com/get-docker/) ≥ 24
- [Docker Compose](https://docs.docker.com/compose/install/) ≥ 2.20（通常随 Docker Desktop 一起安装）
- 可访问 GitHub 的网络环境（拉取镜像）

---

### 🚀 快速开始

#### 1. 克隆仓库（含子模块）

```bash
git clone --recurse-submodules git@github.com:miggene/cli-proxy-dashboard.git
cd cli-proxy-dashboard
```

> 如果已经 clone 但忘了加 `--recurse-submodules`，补救方法：
> ```bash
> git submodule update --init --recursive
> ```

#### 2. 准备 CLIProxyAPI 配置文件

```bash
cp CLIProxyAPI/config.example.yaml CLIProxyAPI/config.yaml
```

然后按需编辑 `config.yaml`，**最少需要关注以下几项**：

```yaml
# 服务端口（保持 8317 即可）
port: 8317

# 认证文件目录（容器内固定为 /root/.cli-proxy-api，不需要修改）
auth-dir: "~/.cli-proxy-api"

# Management API 密钥（Dashboard 通过此密钥调用 Management API）
# 留空则 Management API 完全禁用，Dashboard 改用「本地文件模式」
remote-management:
  allow-remote: true   # Docker 部署时需要允许非 localhost 访问
  secret-key: "your-strong-password-here"

# API Keys（供 AI 客户端访问代理服务使用）
api-keys:
  - "your-api-key-1"
  - "your-api-key-2"
```

#### 3. 准备环境变量

```bash
cp .env.example .env
```

编辑 `.env`：

```dotenv
# 与 config.yaml 中 remote-management.secret-key 保持一致
CPA_MANAGEMENT_KEY=your-strong-password-here
```

> **提示**：如果你不设置 `secret-key` 和 `CPA_MANAGEMENT_KEY`，Dashboard 会自动切换到**本地文件模式**，直接读取 `auths/` 目录中的文件，功能不受影响，只是无法通过 Management API 获取数据。

#### 4. 启动服务

```bash
docker compose up -d
```

首次启动约需 1–2 分钟（拉取镜像 + 构建 Dashboard）。

#### 5. 访问服务

| 服务 | 地址 |
|------|------|
| **CPA-Dashboard（管理面板）** | http://localhost:5000 |
| **CLIProxyAPI（AI 代理）** | http://localhost:8317 |

---

### 📖 使用说明

#### 添加 AI 账户（OAuth 登录）

1. 打开 Dashboard → http://localhost:5000
2. 切换到**账户管理**页面
3. 点击「添加账户」，选择 Provider（Antigravity / Gemini / Codex / Claude / Qwen / iFlow / Kimi）
4. 按页面提示完成 OAuth 流程
5. 认证成功后账户自动出现在列表

| Provider | 说明 | OAuth 回调端口 |
|----------|------|----------------|
| Antigravity | Google Antigravity | 51121 |
| Gemini CLI | Google Gemini CLI | 8085 |
| Codex | OpenAI Codex | 1455 |
| Claude | Anthropic Claude Code | 54545 |
| Qwen | 通义千问（设备码） | — |
| iFlow | iFlow | 55998 |
| Kimi | Moonshot Kimi（设备码） | — |

> **远程服务器部署**：OAuth 回调需要在本地浏览器完成。使用 SSH 端口转发：
> ```bash
> # 以 Antigravity 为例
> ssh -L 51121:localhost:51121 user@your-server
> ```

#### 配置 AI 客户端使用代理

在任何支持 OpenAI 兼容 API 的工具中，将 Base URL 设置为：

```
http://localhost:8317/v1
```

API Key 使用 `config.yaml` 中 `api-keys` 列表里的任意一个。

#### 查看配额

在 Dashboard 账户页面，点击单个账户的「刷新」按钮，或使用「刷新所有配额」批量查询。

---

### ⚙️ 高级配置

#### 修改端口

编辑 `.env`：

```dotenv
# 修改 Dashboard 对外端口（默认 5000）
DASHBOARD_PORT=8080

# 修改 CLIProxyAPI 对外端口（默认 8317）
CPA_API_PORT=9000
```

#### 批量刷新并发数

在 `config.yaml` 中添加：

```yaml
quota-refresh-concurrency: 8  # 默认 4，范围 1–32
```

或通过 `.env` 设置：

```dotenv
# 在 docker-compose.yml cpa-dashboard 的 environment 中添加
CPA_QUOTA_REFRESH_CONCURRENCY=8
```

#### 环境变量参考

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `CLI_PROXY_IMAGE` | CLIProxyAPI 镜像 | `eceasy/cli-proxy-api:latest` |
| `CLI_PROXY_CONFIG_PATH` | `config.yaml` 路径（宿主机） | `./CLIProxyAPI/config.yaml` |
| `CLI_PROXY_AUTH_PATH` | 认证文件目录（宿主机） | `./auths` |
| `CLI_PROXY_LOG_PATH` | 日志目录（宿主机） | `./logs` |
| `CPA_API_PORT` | CLIProxyAPI 对外端口 | `8317` |
| `CPA_MANAGEMENT_KEY` | Management API 密钥 | 空（本地模式） |
| `DASHBOARD_PORT` | Dashboard 对外端口 | `5000` |
| `WEBUI_DEBUG` | Flask 调试模式 | `false` |

---

### 🔧 常用运维命令

```bash
# 查看运行状态
docker compose ps

# 查看实时日志
docker compose logs -f

# 只看某个服务的日志
docker compose logs -f cli-proxy-api
docker compose logs -f cpa-dashboard

# 重启某个服务
docker compose restart cpa-dashboard

# 停止所有服务
docker compose down

# 更新镜像后重新启动
docker compose pull cli-proxy-api
docker compose up -d --build
```

---

### 📁 目录结构

```
cli-proxy-dashboard/
├── docker-compose.yml        # 服务编排配置
├── .env.example              # 环境变量模板
├── .env                      # 用户私有配置（不提交 git）
├── .gitmodules               # 子模块配置
├── auths/                    # OAuth 认证文件（两个服务共享）
├── logs/                     # CLIProxyAPI 日志
├── CLIProxyAPI/              # 子模块：Go 代理服务
│   ├── config.example.yaml   # 配置模板
│   └── config.yaml           # 用户配置（不提交 git）
└── CPA-Dashboard/            # 子模块：Python Web 面板
    ├── Dockerfile
    └── app.py
```

---

### 🐛 常见问题

**Q: Dashboard 显示「无法连接到 CLIProxyAPI」**

A: CLIProxyAPI 启动需要数秒，Dashboard 依赖 healthcheck 等待其就绪。如果长时间无法连接：

```bash
docker compose logs cli-proxy-api   # 检查 API 服务日志
docker compose ps                   # 确认 health 状态是否为 healthy
```

**Q: OAuth 登录时浏览器无法打开回调页面**

A: 确认对应的 OAuth 回调端口已在 `docker-compose.yml` 中 `ports` 列表里暴露，且本地防火墙未拦截。

**Q: config.yaml 修改后不生效**

A: `config.yaml` 通过 volume 挂载，修改后需重启 CLIProxyAPI：

```bash
docker compose restart cli-proxy-api
```

**Q: 如何在不设置 Management API 密钥的情况下使用？**

A: `config.yaml` 中 `remote-management.secret-key` 留空，`.env` 中 `CPA_MANAGEMENT_KEY` 留空，Dashboard 自动切换为「本地文件模式」，直接读取 `auths/` 目录，所有功能正常可用（服务控制除外）。

---

### 📄 许可证

本项目为两个子模块的 Docker 部署配置，遵循各子模块自身的许可证：

- CLIProxyAPI：[MIT License](./CLIProxyAPI/LICENSE)
- CPA-Dashboard：见 [CPA-Dashboard 仓库](https://github.com/dongshuyan/CPA-Dashboard)

---

## English

A ready-to-use Docker Compose setup that integrates **[CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)** and **[CPA-Dashboard](https://github.com/dongshuyan/CPA-Dashboard)** into a unified deployment.

CLIProxyAPI provides a multi-account AI proxy service; CPA-Dashboard provides a visual management UI. They communicate through Docker's internal network.

### 📋 Components

| Submodule | Stack | Default Port | Description |
|-----------|-------|--------------|-------------|
| [CLIProxyAPI](./CLIProxyAPI) | Go | `8317` | AI proxy core — multi-account round-robin for Gemini / Antigravity / Claude / Codex / Qwen / iFlow / Kimi with OpenAI-compatible API |
| [CPA-Dashboard](./CPA-Dashboard) | Python / Flask | `5000` | Web management panel — account monitoring, quota refresh, OAuth login, service status |

### 🏗️ Architecture

```
Host                         Docker Internal Network (cpa-network)
                            ┌──────────────────────────────────────┐
:5000 ──────────────────▶  │  cpa-dashboard  (Flask :5000)        │
                            │       │ http://cli-proxy-api:8317     │
:8317 ──────────────────▶  │  cli-proxy-api  (Go :8317)           │
:8085 / :1455 / :54545      │                                      │
:51121 (OAuth callbacks)    │  Shared Volume: auths/ config.yaml   │
                            └──────────────────────────────────────┘
```

### 📦 Prerequisites

- [Docker](https://docs.docker.com/get-docker/) ≥ 24
- [Docker Compose](https://docs.docker.com/compose/install/) ≥ 2.20

### 🚀 Quick Start

#### 1. Clone with submodules

```bash
git clone --recurse-submodules git@github.com:miggene/cli-proxy-dashboard.git
cd cli-proxy-dashboard
```

#### 2. Prepare config

```bash
cp CLIProxyAPI/config.example.yaml CLIProxyAPI/config.yaml
```

Key settings in `config.yaml`:

```yaml
port: 8317

remote-management:
  allow-remote: true
  secret-key: "your-strong-password-here"

api-keys:
  - "your-api-key-1"
```

#### 3. Set environment variables

```bash
cp .env.example .env
# Edit .env: set CPA_MANAGEMENT_KEY to match remote-management.secret-key
```

#### 4. Start services

```bash
docker compose up -d
```

#### 5. Access

| Service | URL |
|---------|-----|
| **CPA-Dashboard** | http://localhost:5000 |
| **CLIProxyAPI** | http://localhost:8317 |

### Using the AI Proxy

Point any OpenAI-compatible client to:

```
Base URL: http://localhost:8317/v1
API Key:  <any key from api-keys in config.yaml>
```

### Common Commands

```bash
docker compose ps                      # status
docker compose logs -f                 # logs
docker compose restart cpa-dashboard  # restart dashboard
docker compose down                    # stop all
docker compose pull && docker compose up -d --build  # update
```

### License

Deployment configuration — see individual submodule licenses:
- CLIProxyAPI: [MIT License](./CLIProxyAPI/LICENSE)
- CPA-Dashboard: [CPA-Dashboard repository](https://github.com/dongshuyan/CPA-Dashboard)
