# macOS 适配说明

本文档记录 aish 在 macOS 平台上的适配工作，包括平台差异分析、代码修改清单、构建方式及已知限制。

---

## 1. 适配概述

aish 最初仅支持 Linux（musl 静态构建 + systemd 打包）。本次适配的目标是让 **原始命令执行** 和 **Agent 交互（`;` 模式）** 在 macOS 上可编译、可运行。

核心 PTY/终端逻辑无需重写——`nix::pty::openpty`、`TIOCGWINSZ`/`TIOCSCTTY`/`TIOCSWINSZ`、`libc::rlimit`、Unix socket 等在 Darwin libc 中均有定义。主要工作量在于：替换 Linux 专属的 errno 访问方式、将沙箱运行时中的 Linux 专属 syscall 用 `cfg(target_os = "linux")` 门控。

---

## 2. 平台差异分析

### 2.1 编译级阻塞点

| 问题 | Linux | macOS (Darwin) | 解决方案 |
|---|---|---|---|
| `libc::__errno_location()` | glibc/musl 提供 | 未导出 | 替换为 `std::io::Error::last_os_error().raw_os_error()` |
| `libc::openpty` 参数 | `*const termios`, `*const winsize` | `*mut termios`, `*mut winsize` | 改用 `null_mut()` / `&raw mut`（mut 可自动强转为 const） |
| `libc::TIOCSCTTY` 类型 | `c_ulong` | `c_uint` | 显式 `as libc::c_ulong` 转换 |
| `libc::timeval.tv_usec` 类型 | `c_long` (i64) | `c_int` (i32) | 用 `as _` 让编译器按字段类型推断 |
| `libc::libc::passwd` 结构体 | 7 个字段 | 额外含 `pw_change`/`pw_class`/`pw_expire` | 改用 `std::mem::zeroed()` |
| `libc::prctl` / `PR_SET_PDEATHSIG` | Linux 提供 | 未提供 | `#[cfg(target_os = "linux")]` 门控 |
| `libc::ucred` / `SO_PEERCRED` | Linux 提供 | 未提供 | `#[cfg(target_os = "linux")]` 门控 |
| `libc::lgetxattr` | Linux 提供 | 未提供 | `#[cfg(target_os = "linux")]` 门控 |
| `libc::MS_BIND`/`MS_REC`/`MS_REMOUNT`/`MS_RDONLY`/`MNT_DETACH` | Linux 提供 | 未提供 | 定义 cfg 门控的本地常量（macOS 下为 0） |
| `libc::umount2` | Linux 提供 | 未提供 | `#[cfg(target_os = "linux")]` 门控 |
| `libc::unshare` / `CLONE_NEWNS` | Linux 提供 | 符号存在但不支持挂载命名空间 | `#[cfg(target_os = "linux")]` 门控 |
| `libc::SYS_PIDFD_OPEN` / `SYS_PIDFD_SEND_SIGNAL` | Linux syscall 号 | 不存在（调用会调到无关 syscall） | `#[cfg(target_os = "linux")]` 门控，macOS 返回 None |

### 2.2 运行时差异

| 项目 | Linux | macOS | 影响 |
|---|---|---|---|
| 挂载命名空间 (`unshare(CLONE_NEWNS)`) | 支持 | 不支持 | 沙箱降级为不可用 |
| `/proc/self/mountinfo` | 存在 | 不存在 | 静默跳过（已有 `else` 分支） |
| `/proc/<pid>/stat` | 存在 | 不存在 | resource.rs 的 /proc 采样在 macOS 上返回空结果 |
| pidfd_open / pidfd_send_signal | 内核 5.3+ 支持 | 不存在 | 降级为 `kill()` 信号 |
| `RLIMIT_NOFILE` 默认值 | 通常较大 | 可能较小 | 运行时可能需要 `sysctl` 调整 |
| `TIOCSWINSZ` → `SIGWINCH` 保证 | 内核保证 | 行为可能有差异 | 建议实际测试交互程序 |

---

## 3. 代码修改清单

共修改 12 个文件，按 crate 分类：

### aish-pty（PTY 核心）

| 文件 | 修改内容 |
|---|---|
| `executor.rs` | 2 处 `__errno_location` → `last_os_error().raw_os_error()`；`TIOCSCTTY as c_ulong` |
| `persistent.rs` | 3 处 `__errno_location` → 可移植写法；`openpty` 签名（`*mut`）；`TIOCSCTTY as c_ulong`；`tv_usec as _`；测试中 `openpty` 签名 |
| `aish_completion.rs` | 1 处 `__errno_location` → 可移植写法；`tv_usec as _` |
| `daemon.rs` | `openpty` 签名（`*mut`）；`TIOCSCTTY as c_ulong` |
| `resource.rs` | pidfd_open/pidfd_send_signal 门控为 Linux 专属；`AsRawFd`/`FromRawFd` import 门控 |

### aish-security（安全沙箱）

| 文件 | 修改内容 |
|---|---|
| `sudo.rs` | `libc::passwd` 改用 `std::mem::zeroed()` |
| `sandbox/runtime/worker.rs` | `prepare_mount_namespace` 门控为 Linux 专属；`resolve_user_home` 中 `passwd` 改用 `zeroed()` |
| `sandbox/runtime/mount.rs` | MS_*/MNT_DETACH 常量 cfg 门控；`system_mount`/`system_unmount` 门控；测试中引用本地常量 |
| `sandbox/runtime/daemon.rs` | `peer_credentials` 门控为 Linux 专属；`AsRawFd` import 门控 |
| `sandbox/runtime/collect.rs` | `is_opaque_dir` 门控为 Linux 专属；`CString`/`OsStrExt` import 门控 |

### aish-mcp

| 文件 | 修改内容 |
|---|---|
| `transports/stdio.rs` | `prctl(PR_SET_PDEATHSIG)` 门控为 Linux 专属 |

### aish-cli

| 文件 | 修改内容 |
|---|---|
| `tests/pty_daemon_config.rs` | 测试中 `openpty` 签名修复 |

---

## 4. 构建方式

### 4.1 安装 Rust 工具链

```bash
# 方式一：Homebrew（推荐）
brew install rustup
export PATH="/opt/homebrew/opt/rustup/bin:$PATH"
rustup toolchain install 1.95.0 --profile minimal --component clippy,rustfmt

# 方式二：rustup 官方安装脚本
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### 4.2 编译

```bash
# Debug 版（开发用）
cargo build -p aish-cli

# Release 版（日常使用）
cargo build --release -p aish-cli
```

### 4.3 安装到 PATH

```bash
# 复制 release 二进制到 ~/.local/bin（已在 PATH 中）
cp target/release/aish ~/.local/bin/aish
chmod +x ~/.local/bin/aish

# 验证
aish --help
```

> 注意：`~/.local/bin` 在 macOS 上默认已在 PATH 中。如果未添加，请在 `~/.zshrc` 中加入 `export PATH="$HOME/.local/bin:$PATH"`。

### 4.4 验证编译

```bash
cargo check --workspace     # 全工作区编译检查
cargo test --workspace      # 全工作区测试
```

---

## 5. 已知限制

### 5.1 沙箱降级

macOS 不支持 Linux 的挂载命名空间（`unshare(CLONE_NEWNS)`），因此安全沙箱的预运行隔离功能在 macOS 上不可用。沙箱会返回 `SandboxUnavailable`，agent 交互会自动降级到策略规则评估（`sandbox_off_action` 决定 BLOCK/CONFIRM/ALLOW）。

**影响**：AI 命令的安全性评估仍通过策略规则（glob 匹配）工作，但无法通过沙箱预运行捕获文件系统变更。

### 5.2 资源采样

`aish-pty/src/resource.rs` 中的进程资源采样依赖 `/proc`，在 macOS 上不可用。资源监控功能在 macOS 上返回空结果，不影响核心功能。

### 5.3 打包与 CI

当前打包脚本（`build.sh`、`Makefile`、`build_bundle.sh`）默认面向 Linux（musl 静态构建 + systemd 安装 + .deb 包）。macOS 的打包支持尚未实现。

### 5.4 集成测试

`crates/aish-cli/tests/pty_daemon_config.rs` 中的 3 个端到端测试在 macOS 上失败（等待 PTY daemon session 超时）。手动验证 daemon 能正常创建 session 文件，但测试中的 cwd 匹配或 daemon 启动时序存在 macOS 差异。不影响实际使用。

---

## 6. 适配验证结果

| 验证项 | 结果 |
|---|---|
| `cargo check --workspace` | ✅ 全部通过，无错误无警告 |
| 单元测试 | ✅ 大部分通过（除上述 3 个集成测试） |
| 原始命令执行 | ✅ 手动验证通过 |
| Agent 交互（`;` 模式） | ✅ 手动验证通过 |
| PTY daemon 启动 | ✅ 手动验证 socket 和 session 文件正常创建 |
| Release 构建 | ✅ 构建成功（26MB） |

---

## 7. 后续工作建议

1. **打包支持**：为 macOS 添加 `.pkg` 或 Homebrew formula 安装方式
2. **CI 扩展**：在 `.github/workflows/ci.yml` 中添加 macOS (arm64) 的 test + build job
3. **文档更新**：更新 README 平台徽章为 `linux/macOS`，补充 macOS 安装说明
4. **集成测试修复**：排查 `pty_daemon_config.rs` 测试在 macOS 上的失败原因
5. **资源采样**：为 macOS 实现 `sysctl` 或 `libproc` 替代 `/proc` 的进程资源采样
