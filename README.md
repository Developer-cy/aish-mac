<div align="center">

English | [简体中文](README_CN.md)

---

# AISH for macOS

Empower the Shell to think. Evolve Operations.

> **本仓库基于 [AI-Shell-Team/aish](https://github.com/AI-Shell-Team/aish) 原始项目适配 macOS 平台而来。**
> macOS 适配的详细说明请参阅 [docs/macOS_ADAPTATION.md](docs/macOS_ADAPTATION.md)。

[![Official Website](https://img.shields.io/badge/Website-aishell.ai-blue.svg)](https://www.aishell.ai)
[![GitHub](https://img.shields.io/badge/GitHub-Developer--cy/aish--mac-black.svg)](https://github.com/Developer-cy/aish-mac)
[![Rust Version](https://img.shields.io/badge/rust-1.89+-orange.svg)](https://www.rust-lang.org/)
[![Platform](https://img.shields.io/badge/platform-macOS-lightgrey.svg)](#)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)

![](./docs/images/demo_show.gif)

**A Real AI Shell: Complete PTY + Configurable Security & Risk Control**



</div>

---

## Table of Contents

- [Why Choose AISH](#why-choose-aish)
- [Quick Start](#quick-start)
- [Installation](#installation)
- [Uninstallation](#uninstallation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Security & Risk Control](#security--risk-control)
- [Skills (Plugins)](#skills-plugins)
- [Data & Privacy](#data--privacy)
- [Documentation](#documentation)
- [Community & Support](#community--support)
- [Development & Testing](#development--testing)
- [Contributing](#contributing)
- [License](#license)

---

## Why Choose AISH

- **True Interactive Shell**: Full PTY support, runs interactive programs like `vim` / `ssh` / `top`
- **AI Native Integration**: Describe tasks in natural language, generate, explain and execute commands
- **Secure & Controllable**: AI commands have risk grading and confirmation flow; optional sandbox pre-run for change assessment
- **Extensible**: Skills plugin system with hot loading and priority override
- **Low Migration Cost**: Compatible with regular commands and workflows, everything in terminal by default

---

## Feature Comparison

| Feature | AISH | Claude Code |
|---------|------|-------------|
| 🎯 **Core Positioning** | Ops/System Troubleshooting CLI | Development Coding Assistant |
| 🤖 **Multi-Model Support** | ✅ Fully Open | ⚠️ Mainly Claude |
| 🔧 **Sub-Agent System** | ✅ ReAct Diagnostic Agent | ✅ Multiple Agent Types |
| 🧩 **Skills Support** | ✅ Hot Loading | ✅ |
| 🖥️ **Native Terminal Integration** | ✅ Full PTY Support | ⚠️ Limited Support |
| 🛡️ **Security Risk Assessment** | ✅ Security Confirmation | ✅ Security Confirmation |
| 🌐 **Local Model Support** | ✅ Fully Supported | Fully Supported  |
| 📁 **File Operation Tools** | ✅ Minimal Essential Suppport | ✅ Full Support |
| 💰 **Completely Free** | ✅ Open Source | ❌ Paid Service |
| 📊 **Observability** | ✅ Langfuse Optional | ⚠️ Built-in |
| 🌍 **Multi-language Output** | ✅ Auto Detection | ✅ |

---

## Quick Start

### 1) Install and Launch

#### Option 1: Install from Source (Recommended)

```bash
# 克隆仓库
git clone git@github.com:Developer-cy/aish-mac.git
cd aish-mac

# 安装 Rust 工具链（如已安装可跳过）
brew install rustup
export PATH="/opt/homebrew/opt/rustup/bin:$PATH"

# 编译 Release 版
cargo build --release -p aish-cli

# 安装到 PATH
cp target/release/aish ~/.local/bin/aish
chmod +x ~/.local/bin/aish
```

#### Option 2: Direct Run (Development)

```bash
cargo run --bin aish
```

Then launch:

```bash
aish
```

Note: `aish` without subcommands is equivalent to `aish run`.

### 2) Use Like a Regular Shell

```bash
aish> ls -la
aish> cd /etc
aish> vim hosts
```

### 3) Let AI Do the Work (Start with ;)

Starting with `;` or `；` enters AI mode:

```bash
aish> ;find files larger than 100M in current directory and sort by size
aish> ;explain this command: tar -czf a.tgz ./dir
```

---

## Installation

### Install from Source (macOS)

```bash
# 克隆仓库
git clone git@github.com:Developer-cy/aish-mac.git
cd aish-mac

# 编译 Release 版
cargo build --release -p aish-cli

# 安装到 PATH（~/.local/bin 默认已在 PATH 中）
cp target/release/aish ~/.local/bin/aish
chmod +x ~/.local/bin/aish
```

### Run from Source (Development/Trial)

```bash
cargo run --bin aish
```

---

## Uninstallation

Remove the binary:

```bash
rm ~/.local/bin/aish
```

Optional: Clean configuration files (will clear model/API keys etc.):

```bash
rm -rf ~/.config/aish
```

---

## Configuration

### Configuration File Location

- Default: `~/.config/aish/config.yaml` (or `$XDG_CONFIG_HOME/aish/config.yaml` if `XDG_CONFIG_HOME` is set)

### Priority (High to Low)

1. Command-line arguments
2. Environment variables
3. Configuration file

### Minimal Configuration Example

```yaml
# ~/.config/aish/config.yaml
model: openai/deepseek-chat
api_base: https://openrouter.ai/api/v1
api_key: your_api_key
```

Alternatively via environment variables (more suitable for secrets):

```bash
export AISH_MODEL="openai/deepseek-chat"
export AISH_API_BASE="https://openrouter.ai/api/v1"
export AISH_API_KEY="your_api_key"

```

> Tip: LiteLLM also supports reading vendor-specific environment variables (e.g., `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`).

Interactive configuration (optional):

```bash
aish setup
```

Tool calling compatibility check (confirm selected model/channel supports tool calling):

```bash
aish check-tool-support --model openai/deepseek-chat --api-base https://openrouter.ai/api/v1 --api-key your_api_key
```

Langfuse (optional observability):

1) Enable in configuration:

```yaml
enable_langfuse: true
```

2) Set environment variables:

```bash
export LANGFUSE_PUBLIC_KEY="..."
export LANGFUSE_SECRET_KEY="..."
export LANGFUSE_HOST="https://cloud.langfuse.com"
```

`aish check-langfuse` will run checks when `check_langfuse.py` exists in project root.

---

## Usage

### Common Input Types

| Type | Example | Description |
|:----:|---------|-------------|
| Shell Commands | `ls -la`, `cd /path`, `git status` | Execute regular commands directly |
| AI Requests | `;how to check port usage`, `;find files larger than 100M` | Enter AI mode with `;`/`；` prefix |
| Built-in Commands | `help`, `clear`, `exit`, `quit` | Shell built-in control commands |
| Model Switching | `/model gpt-4` | View or switch model |

### Shell Compatibility (PTY)

```bash
aish> ssh user@host
aish> top
aish> vim /etc/hosts
```

---

## Security & Risk Control

AI Shell only performs security assessment on **AI-generated and ready-to-execute** commands.

### Risk Levels

- **LOW**: Allowed by default
- **MEDIUM**: Confirmation before execution
- **HIGH**: Blocked by default

### Security Policy File Path

Policy files are resolved in this order:
1. `~/.config/aish/security_policy.yaml` (user-level; auto-generated template if not exists)

### Sandbox Pre-run (Optional, Linux Only)

Default policy has sandbox pre-run **disabled**. The sandbox uses Linux mount namespace isolation (`unshare(CLONE_NEWNS)`) and is **not available on macOS**.

On macOS, the sandbox will return `SandboxUnavailable` and agent interactions will fall back to policy rule evaluation (`sandbox_off_action`: BLOCK/CONFIRM/ALLOW). Security assessment still works via glob-matched policy rules.

To enable on Linux:

1) Set in security policy:

```yaml
global:
  enable_sandbox: true
```

2) Start privileged sandbox service (systemd):

```bash
sudo systemctl enable --now aish-sandbox.socket
```

Default socket: `/run/aish/sandbox.sock`.

---

## Skills (Plugins)

Skills extend AI's domain knowledge and workflows, supporting hot loading and priority override.

Default scan directories and priority:
- `~/.config/aish/skills/` (or `$AISH_CONFIG_DIR/skills`)
- `~/.claude/skills/`

---

## Data & Privacy

This project stores the following data locally (for troubleshooting and traceability):

- **Logs**: Default `~/.config/aish/logs/aish.log`
- **Sessions/History**: Default `~/.local/share/aish/sessions.db` (SQLite)
- **Large Output Offload**: Default `~/.local/share/aish/offload/`

Recommendations:
- Don't commit real API keys to repository; prefer environment variables or secret management systems.
- Production environments can combine security policies to limit AI-accessible directory scope.

---

## Documentation

- Configuration Guide: `CONFIGURATION.md`
- Quick Start: `QUICKSTART.md`
- macOS Adaptation Guide: `docs/macOS_ADAPTATION.md`
- Command Correction Mechanism: `docs/command-interaction-correction.md`

---

## Community & Support

| Link | Description |
|------|-------------|
| [Official Website](https://www.aishell.ai) | Project homepage and more information |
| [GitHub Repository](https://github.com/Developer-cy/aish-mac) | macOS adaptation source code |
| [Original Repository](https://github.com/AI-Shell-Team/aish/) | Original Linux project |
| [GitHub Issues](https://github.com/Developer-cy/aish-mac/issues) | Bug reports |
| [Discord](https://discord.com/invite/Pw2mjZt3) | Join the community |

> If you find AISH helpful, please consider giving us a Star ⭐
> It helps more people discover the project and keeps us motivated to continue improving it.


---

## Development & Testing

This repository is the Rust implementation of aish, adapted for macOS. The original project is [AI-Shell-Team/aish](https://github.com/AI-Shell-Team/aish).

```bash
# Build (Release)
cargo build --release -p aish-cli

# Run Rust tests
cargo test --workspace

# Lint Rust code
cargo clippy --all-targets -- -D warnings
```

#### Architecture (14 crates)

| Crate | Description |
|-------|-------------|
| `aish-core` | Error types, shared enums (RiskLevel, MemoryCategory, etc.) |
| `aish-config` | YAML config loading with XDG paths and env overrides |
| `aish-i18n` | Internationalization with 6 locales and embedded fallback |
| `aish-pty` | PTY executor with fork/exec, select()-based I/O, output offload |
| `aish-llm` | OpenAI-compatible HTTP client with SSE streaming and tool calling |
| `aish-session` | SQLite session persistence with WAL mode |
| `aish-context` | Sliding-window context manager with tiktoken token counting |
| `aish-security` | Security policy engine with glob-to-regex pattern matching |
| `aish-skills` | Skill plugin discovery with hot-reload via notify |
| `aish-memory` | Markdown-based long-term memory with relevance scoring |
| `aish-tools` | Built-in tools: bash, WebFetch, fs (read/write/edit), ask_user, memory, skill |
| `aish-scripts` | .aish script system with frontmatter, ai "prompt" syntax, hooks |
| `aish-shell` | Main shell: REPL loop, AI handler, built-in commands, animation |
| `aish-cli` | CLI entry point with clap derive macros |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.
---

## License

`LICENSE` (Apache 2.0)
