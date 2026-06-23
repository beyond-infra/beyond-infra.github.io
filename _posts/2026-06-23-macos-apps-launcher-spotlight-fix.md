---
title: "macOS 26 App 启动器打不开？可能是你关了 Spotlight"
date: 2026-06-23 23:30:00 +0800
categories: [Mac, 系统优化]
tags: [macOS, Spotlight, launchctl, 启动器, 调试]
---

## 引言

macOS 26 把 App 启动器从 Launchpad 换成了 `Apps.app`（`com.apple.apps.launcher`），点击弹窗展示所有应用。平时不用这个功能，一直关着 Spotlight 系列服务相安无事，直到某天随手点了一下——没反应。

这篇文章记录排查过程，结论很简短：**禁用 `com.apple.Spotlight` 会导致 App 启动器无法打开**。

## 问题现象

点击 Dock 上的 Apps 图标，没有任何窗口弹出。`open -b com.apple.apps.launcher` 返回 0，但进程瞬间退出，不留 crash log。

```bash
$ open -b com.apple.apps.launcher
$ pgrep -l Apps
1288 ManagedAppsSubs   # 这不是 Apps.app
```

## 排查过程

### 第一回合：ResetLaunchPad 标记

查看 Dock 配置，发现 `ResetLaunchPad = 1` 卡住了：

```bash
$ defaults read com.apple.dock | grep ResetLaunchPad
ResetLaunchPad = 1;
```

这个标记本应由 Dock 处理后自动清零，但 Dock 没有处理它。删除标记并重启 Dock：

```bash
$ defaults delete com.apple.dock ResetLaunchPad
$ killall Dock
```

标记清除了，问题依旧。

### 第二回合：Dock 数据目录

发现 `~/Library/Application Support/Dock/` 整个目录不存在（之前用 CleanMyMac 清理过），导致 Dock 无法写 Launchpad 数据库：

```bash
$ mkdir -p ~/Library/Application\ Support/Dock
$ killall Dock
```

目录恢复了，问题依旧。

### 第三回合：日志里的关键线索

开始盯 Dock 日志：

```bash
$ sudo log stream --predicate 'process == "Dock"' --style compact
```

每次尝试打开 Apps.app 时，Dock 都会输出：

```
[com.apple.dock:spotlight] No session to send message with
```

Dock 通过 XPC 跟 Spotlight 通信来启动 Apps.app，但 Spotlight 没在跑。回顾之前在 launchctl 里做过的禁用——果然，`com.apple.Spotlight` 被 disable 了。

### 第四回合：确认根因

从 Dock 二进制文件里能直接看到它依赖 Spotlight：

```bash
$ strings /System/Library/CoreServices/Dock.app/Contents/MacOS/Dock | grep -i spotlight
com.apple.dock.spotlight
com.apple.private.dock.spotlight
SpotlightXPCClientMessage
SpotlightXPCListener
showSpotlightBasedLaunchpad
```

`showSpotlightBasedLaunchpad` 这个符号说明：macOS 26 的 App 启动器是"基于 Spotlight 的 Launchpad"。

### 验证修复

启用 `com.apple.Spotlight`，重启 Dock：

```bash
$ launchctl enable gui/501/com.apple.Spotlight
$ killall Dock
```

日志立刻出现 XPC 握手：

```
Dock[10564] [com.apple.dock:spotlight] Activating listener
Dock[10564] Accepting connection
Dock[10564] spotlight window set to <private>
Dock[10564] Sending .launchAppsBrowsing
Dock[10564] Received message .activated
```

App 启动器正常弹出。

### 反向验证

重新禁用 `com.apple.Spotlight`，重启 Dock，App 启动器再次打不开。**一个服务，一开一关，问题完全复现。**

## 根因

macOS 26 的 App 启动器（`Apps.app`）依赖 `com.apple.Spotlight` 用户级 LaunchAgent（`/System/Library/LaunchAgents/com.apple.Spotlight.plist`）。

流程是：Dock → XPC → com.apple.Spotlight → 渲染应用列表窗口。这个中间人没了，Apps.app 起来就走了。

## 影响范围

之前在这台机器上禁用了以下 Spotlight 相关服务：

| 服务 | 对 Apps.app 的影响 |
|------|-------------------|
| `com.apple.Spotlight` | **直接导致打不开** |
| `com.apple.metadata.mds` | 未直接影响（但 Spotlight 的上游依赖） |
| `com.apple.corespotlightd` | 未直接影响 |
| `com.apple.spotlightknowledged` | 未直接影响 |
| `com.apple.corespotlightservice` | 未直接影响 |

唯一关键的是 `com.apple.Spotlight` 这一个用户代理。

## 总结

如果你也想关 Spotlight 省资源但保留 App 启动器：

1. 可以关闭 `mds`、`corespotlightd` 等底层索引服务
2. **不要禁用 `com.apple.Spotlight`**——这是 Dock 和 App 启动器之间的桥梁
3. 如果已经关了，恢复命令：

```bash
launchctl enable gui/$(id -u)/com.apple.Spotlight
killall Dock
```

下次遇到 macOS 系统组件莫名其妙打不开，先 `log stream` 盯一眼，日志往往比你猜的准。
