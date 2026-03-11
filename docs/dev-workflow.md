# Development Workflow

本文档面向 OpenFang 开发者，涵盖日常开发中的编译运行、调试、热重载、局域网访问等实践。

## 目录

- [前置条件](#前置条件)
- [快速启动（开发模式）](#快速启动开发模式)
- [配置 LLM 提供商](#配置-llm-提供商)
- [局域网访问 Web Dashboard](#局域网访问-web-dashboard)
- [日志与调试](#日志与调试)
- [快速编译检查](#快速编译检查)
- [自动重载（cargo-watch）](#自动重载cargo-watch)
- [常用环境变量速查](#常用环境变量速查)
- [完整开发启动示例](#完整开发启动示例)
- [防火墙配置](#防火墙配置)
- [故障排查](#故障排查)

---

## 前置条件

- **Rust 1.75+**（通过 [rustup](https://rustup.rs/) 安装）
- **Git**
- 至少一个 LLM 提供商的 API Key（或使用 Ollama 本地模型，无需 Key）

首次克隆后无需手动编译，`cargo run` 会自动处理。

---

## 快速启动（开发模式）

开发阶段**不需要**手动执行 `cargo build`，直接使用 `cargo run` 即可自动增量编译并运行 debug 版本：

```bash
# 初始化配置（仅首次需要）
cargo run -p openfang-cli -- init

# 启动 daemon（自动编译 + 运行）
cargo run -p openfang-cli -- start
```

> **说明**：`cargo run` 每次只重新编译有变更的代码（增量编译），首次编译约需几分钟（SQLite、Wasmtime 等依赖），后续通常数秒内完成。

修改代码后，`Ctrl+C` 停止 daemon，再次 `cargo run -p openfang-cli -- start` 即可。

---

## 配置 LLM 提供商

### 方式 1：使用 .env 文件（推荐，一次配置永久生效）

将 API Key 写入 `~/.openfang/.env`，程序启动时自动加载：

```bash
# 创建 .env 文件
mkdir -p ~/.openfang
cat > ~/.openfang/.env << 'EOF'
GROQ_API_KEY=gsk_your_key_here
EOF
```

也可以使用 CLI 命令管理 Key：

```bash
cargo run -p openfang-cli -- config set-key groq
```

### 方式 2：命令行传入环境变量

```bash
GROQ_API_KEY="gsk_..." cargo run -p openfang-cli -- start
```

### 支持的提供商

| 提供商 | 环境变量 | 说明 |
|--------|----------|------|
| Groq | `GROQ_API_KEY` | 快速推理，有免费额度 |
| OpenAI | `OPENAI_API_KEY` | GPT 系列 |
| Anthropic | `ANTHROPIC_API_KEY` | Claude 系列 |
| DeepSeek | `DEEPSEEK_API_KEY` | DeepSeek 系列 |
| Gemini | `GEMINI_API_KEY` | Google Gemini |
| Ollama | （无需 Key） | 本地模型 |

完整列表见 [docs/providers.md](providers.md)。

---

## 局域网访问 Web Dashboard

默认绑定地址是 `127.0.0.1:4200`（仅本机可访问）。要从局域网其他设备访问，需改为 `0.0.0.0`。

### 方式 A：修改 config.toml（推荐）

编辑 `~/.openfang/config.toml`：

```toml
api_listen = "0.0.0.0:4200"
```

### 方式 B：通过环境变量（不修改配置文件）

```bash
OPENFANG_LISTEN="0.0.0.0:4200" cargo run -p openfang-cli -- start
```

启动后，在局域网内任意设备浏览器访问：

```
http://<服务器IP>:4200/
```

Dashboard 包含 Chat、Overview、Analytics、Logs、Sessions、Channels、Workflows、Scheduler、Skills、Settings 等页面。

---

## 日志与调试

OpenFang 使用 `tracing` 框架，通过 `RUST_LOG` 环境变量控制日志级别。默认级别为 `info`。

### 日志级别

| 级别 | 说明 |
|------|------|
| `error` | 仅错误 |
| `warn` | 警告 + 错误 |
| `info` | 常规运行信息（默认） |
| `debug` | 调试信息，包含请求细节 |
| `trace` | 最详细，包含请求/响应完整内容 |

### 常用示例

```bash
# 全局 debug 日志
RUST_LOG=debug cargo run -p openfang-cli -- start

# 仅 API 层 debug（减少噪音）
RUST_LOG=openfang_api=debug cargo run -p openfang-cli -- start

# 仅 LLM driver 层 trace（调试 LLM 请求/响应）
RUST_LOG=openfang_runtime=trace cargo run -p openfang-cli -- start

# 组合：全局 info，API 层 debug，runtime trace
RUST_LOG=info,openfang_api=debug,openfang_runtime=trace cargo run -p openfang-cli -- start

# 只看 kernel 调度和 agent 生命周期
RUST_LOG=openfang_kernel=debug cargo run -p openfang-cli -- start
```

### 各 crate 的 RUST_LOG 过滤名

| 过滤名 | 对应内容 |
|--------|----------|
| `openfang_api` | HTTP 路由、WebSocket、SSE、Dashboard |
| `openfang_runtime` | Agent 循环、LLM 驱动、工具执行 |
| `openfang_kernel` | 配置加载、调度、工作流、预算 |
| `openfang_memory` | SQLite 读写、向量嵌入、会话管理 |
| `openfang_channels` | 消息渠道适配器（Telegram/Discord 等） |
| `openfang_wire` | OFP P2P 网络 |
| `openfang_extensions` | MCP 服务器、凭证保险库 |

---

## 快速编译检查

开发过程中想快速验证代码正确性而不启动 daemon：

```bash
# 类型检查（最快，不生成二进制）
cargo check --workspace

# 编译所有库 crate（跳过 CLI 二进制，避免被运行中的 daemon 锁住）
cargo build --workspace --lib

# 运行所有测试（1744+）
cargo test --workspace

# 运行单个 crate 的测试
cargo test -p openfang-kernel
cargo test -p openfang-api

# Clippy lint 检查（CI 要求零警告）
cargo clippy --workspace --all-targets -- -D warnings

# 代码格式化
cargo fmt --all
```

---

## 自动重载（cargo-watch）

[cargo-watch](https://github.com/watchexec/cargo-watch) 可以在文件变更时自动重新编译并重启 daemon，免去手动 `Ctrl+C` + 重新运行的循环：

```bash
# 安装（仅需一次）
cargo install cargo-watch

# 文件变更自动重启 daemon
cargo watch -x 'run -p openfang-cli -- start'

# 带环境变量
RUST_LOG=debug OPENFANG_LISTEN="0.0.0.0:4200" \
  cargo watch -x 'run -p openfang-cli -- start'

# 后台持续做编译检查（适合开发时另开一个终端）
cargo watch -x 'check --workspace'

# 文件变更自动跑测试
cargo watch -x 'test --workspace'
```

---

## 常用环境变量速查

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `RUST_LOG` | 日志级别 | `info` |
| `OPENFANG_LISTEN` | API 绑定地址（覆盖 config.toml） | `127.0.0.1:4200` |
| `OPENFANG_HOME` | 数据/配置目录 | `~/.openfang` |
| `OPENFANG_API_KEY` | HTTP API 认证 Key | 空（无认证） |
| `OPENFANG_VAULT_KEY` | 凭证保险库加密密钥 | — |
| `GROQ_API_KEY` | Groq 提供商 Key | — |
| `OPENAI_API_KEY` | OpenAI 提供商 Key | — |
| `ANTHROPIC_API_KEY` | Anthropic 提供商 Key | — |
| `DEEPSEEK_API_KEY` | DeepSeek 提供商 Key | — |
| `GEMINI_API_KEY` | Google Gemini Key | — |

---

## 完整开发启动示例

```bash
# 一行命令：局域网可访问 + debug 日志 + Groq 作为 LLM
RUST_LOG=debug \
OPENFANG_LISTEN="0.0.0.0:4200" \
GROQ_API_KEY="gsk_your_key" \
cargo run -p openfang-cli -- start
```

启动成功后终端输出：

```
OpenFang API server listening on http://0.0.0.0:4200
```

局域网浏览器访问 `http://<服务器IP>:4200/` 即可使用 Web Dashboard。

### 验证服务是否正常

```bash
# 健康检查
curl http://localhost:4200/api/health

# 列出 Agent
curl http://localhost:4200/api/agents

# 发送消息（替换 <agent-id>）
curl -X POST http://localhost:4200/api/agents/<agent-id>/message \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello"}'
```

---

## 防火墙配置

如果局域网设备无法访问 4200 端口，检查服务器防火墙：

```bash
# ufw (Ubuntu/Debian)
sudo ufw allow 4200/tcp

# firewalld (CentOS/Fedora)
sudo firewall-cmd --add-port=4200/tcp --permanent && sudo firewall-cmd --reload

# iptables
sudo iptables -A INPUT -p tcp --dport 4200 -j ACCEPT
```

---

## 故障排查

### 端口被占用

```bash
# 查看占用 4200 端口的进程
lsof -i :4200
# 或
ss -tlnp | grep 4200

# 换一个端口
OPENFANG_LISTEN="0.0.0.0:4201" cargo run -p openfang-cli -- start
```

### 编译时二进制被锁

如果 daemon 正在运行，编译可能报文件锁定错误。先停止 daemon：

```bash
# 通过 CLI 停止
cargo run -p openfang-cli -- stop

# 或直接 kill
pkill -f openfang
```

或使用 `--lib` 跳过二进制编译：

```bash
cargo build --workspace --lib
```

### 检查配置是否正确

```bash
cargo run -p openfang-cli -- doctor
```

`doctor` 命令会检查配置文件、.env 文件、API Key、端口等是否就绪。
