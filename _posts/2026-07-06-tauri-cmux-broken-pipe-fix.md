---
title: Tauri .app 调用 cmux CLI 报 Broken Pipe 的根因与解决
date: 2026-07-06 16:00:00 +0800
categories: [桌面开发]
tags: [Tauri, macOS, cmux, 终端, IPC]
---

我在 ChatVault（一个 Tauri 桌面应用）里实现了"恢复会话"功能：点击按钮，在 cmux 中打开一个新的终端窗口并执行恢复命令。`pnpm tauri dev` 下一切正常，但打包成 `.app` 安装到 `/Applications` 后，点击按钮报错：

```
Error: Failed to write to socket (Broken pipe, errno 32)
```

本文梳理整个过程：尝试了什么、为什么失败、最终怎么绕过去的。

## 背景

`launch_cmux` 函数做的事情很简单：fork 一个子进程，执行 cmux CLI 二进制文件。

```rust
fn launch_cmux(command: &str, cwd: Option<&str>) -> Result<(), String> {
    let cmux_bin = "/Applications/cmux.app/Contents/Resources/bin/cmux";
    Command::new(cmux_bin)
        .args(["workspace", "create", "--command", command])
        .output()?;
    Ok(())
}
```

dev 模式下（`pnpm tauri dev`）这条路径一直正常。`tauri dev` 本质上是在终端里直接跑 Rust 二进制，子进程继承了完整的 shell 环境，cmux CLI 能找到 daemon 的 Unix socket 并正常通信。打包成 `.app` 后，app 由 macOS LaunchServices 启动，环境是干净的——PATH 只有 `/usr/bin:/bin:/usr/sbin:/sbin`，没有 `CMUX_SOCKET_PATH`。

我加上了显式 socket 路径后发现，socket 能连上，但 cmux daemon 在连接后**立刻断开**（Broken pipe）。

## 尝试过的方案

### 1. 显式传入 `--socket`

cmux CLI 支持 `--socket` 参数和 `CMUX_SOCKET_PATH` 环境变量，daemon 把当前 socket 路径写在 `/tmp/cmux-last-socket-path`。

```rust
let socket = std::fs::read_to_string("/tmp/cmux-last-socket-path")?;
cmd.arg("--socket").arg(socket.trim());
```

结果：**同样的 Broken pipe**。

### 2. `open -a cmux --args`

用 `open -a` 让系统级唤起 cmux，后面跟 `--args` 传递参数给 cmux 的 `main()`。

```rust
Command::new("open")
    .args(["-a", "cmux", "--args", "workspace", "create", "--command", command])
    .output()?;
```

结果：`open` 返回 exit 0，但 **cmux 里没有创建 workspace**。cmux 的单实例模型会在已有 daemon 进程时把参数**转发**给 daemon，同样触发 Broken pipe，静默失败。

### 3. URL Scheme `cmux://open`

cmux 注册了 `cmux://` URL scheme。我用 `url::Url` 构造带 query 参数的 URL，然后 `open "cmux://open?command=..."`。

```rust
let url = format!("cmux://open?command={}", encoded_command);
Command::new("open").arg(&url).output()?;
```

结果：**不创建 workspace**。即使从终端执行同样无效——URL scheme 的参数格式不对，或者 cmux 根本没有实现 `command` query 参数。

### 4. `bash -l -c`

猜测是 shell 环境差异——打包 app 的 PATH 里没有 cmux 的路径。用 `bash -l -c` 套一层 login shell。

```rust
Command::new("/bin/bash")
    .args(["-l", "-c", &format!("exec {cmux_bin} workspace create --command '{command}'")])
    .output()?;
```

结果：Terminal 里能跑（workspace 创建成功），但打包 app 里**同样的 Broken pipe**。不是环境变量的问题。

### 5. 直连 Unix Socket

排除了 CLI 层面的所有因素后，问题必然在 cmux daemon 的**连接认证**上。既然 cmux CLI 也是通过 Unix socket 与 daemon 通信，直接发 JSON 试试。

```rust
let payload = json!({"method": "workspace.create", "params": {"command": command}});
let mut stream = UnixStream::connect(&socket)?;
serde_json::to_writer(&mut stream, &payload)?;
```

用 `nc -U` 从终端直连同一个 socket 发 JSON 能创建 workspace。但从打包 app 里做同样的事，daemon 返回：

```
ERROR: Access denied — only processes started inside cmux can connect
```

**根因确认：cmux daemon 校验连接来源进程的"血缘"。** daemon 只接受由 cmux 自身 fork 出来的子进程的连接。打包 app（由 LaunchServices 启动）不在这个信任域内。

## 最终方案：`.command` 文件

既然直接调用 cmux CLI 不被 daemon 信任，那让 macOS 把任务**路由给 cmux 自己的 handler**。macOS 上 `.command` 文件有特殊含义——双击它会在终端中执行。而 cmux.app 的 `application:openFile:` delegate 也会处理 `.command` 文件，在 cmux 内部打开一个受信任的 shell 并执行。

```rust
fn launch_cmux(command: &str, cwd: Option<&str>) -> Result<(), String> {
    let path = std::env::temp_dir()
        .join(format!("chatvault_{}.command", std::process::id()));
    let mut script = String::from("#!/bin/sh\n");
    if let Some(dir) = cwd {
        script.push_str(&format!("cd '{}'\n", dir));
    }
    script.push_str(command);
    script.push('\n');

    std::fs::write(&path, &script)?;
    std::fs::set_permissions(&path, std::fs::Permissions::from_mode(0o755))?;

    Command::new("open")
        .args(["-a", "cmux", &path.to_string_lossy()])
        .output()?;
    Ok(())
}
```

流程：写 `.command` 文件到临时目录 → `chmod +x` → `open -a cmux` 打开。cmux.app 内部 handler 读取这个文件，在 cmux 进程树下创建一个 workspace 并执行脚本。脚本在 cmux 的 shell 里运行，daemon 自然信任。

## 原理

cmux daemon 的进程认证逻辑大概是这样：

- **终端里的 cmux CLI**：进程树是 `terminal → cmux CLI`，daemon 可以追溯到 `terminal → cmux.app → cmux daemon`，通过签名校验确认链条
- **打包 app 里的 cmux CLI**：进程树是 `LaunchServices → chatvault → cmux CLI`，daemon 看到的 parent chain 里没有 cmux.app，拒绝连接
- **`.command` 文件方式**：`LaunchServices → cmux.app（处理文件打开事件）`，文件在 cmux 进程树下执行，daemon 信任这个上下文

本质上用的是 **cmux 自己的特权边界**——让它自己执行脚本，而不是从外部进程调用它的 CLI。

## 小结

这个问题从 Broken pipe 报错出发，最终落到 macOS 的进程模型和 cmux 的认证机制上。如果一开始就意识到 cmux daemon 有进程血缘校验，可能少绕很多路。但排查过程中试过的几个方案（`--socket`、`bash -l`、直连 socket）最终都指向了同一个根因。

如果 cmux 未来提供 `--allow-untrusted` 之类的参数，就可以直接用 CLI 了。在那之前，`.command` 文件是成本最低的绕过方式——不需要修改 cmux、不需要额外权限、不需要 entitlements。
