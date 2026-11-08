---
title: "让 RDP 下的 GNOME 会话真正可用（1）：日常使用中哪里出了问题"
date: 2026-11-08
draft: false
summary: "我们把自己的日常桌面搬到了无头 GNOME 会话里的 FreeRDP shadow 服务器上。头两天就暴露出：声音噼啪、屏幕区域残留旧画面、慢速链路上延迟越来越大、两块显示器变成一块大显示器、指针看不见。这个系列记录了每个问题是怎么找到、怎么修的，并附上数字。"
tags: ["rdp", "freerdp", "gnome", "mutter", "pipewire", "wayland", "gnome-rdp-series"]
---

远程桌面只有做到能长期住在里面，才算完成。演示掩盖了八小时工作日里立刻就会暴露的问题：
滚动页面时声音爆音，顶栏的时钟慢了一分钟，酒店 Wi-Fi 上会话越用越慢。

所以我们做了最直接的事：把自己的日常桌面搬到这套方案上，把所有让人烦的地方都记下来。
这个系列就是结果，每个问题一篇。

## 环境

```
 laptop (SDL3 client)                        Linux server
┌──────────────────────┐   RDP / TLS    ┌───────────────────────────────────────────┐
│ sdl-freerdp          │◀──────────────▶│ session manager (sesman)                  │
│  /multimon, sound    │                │  └─ headless GNOME session per user       │
└──────────────────────┘                │      └─ freerdp-shadow-cli, Mutter backend│
                                        │          ScreenCast + RemoteDesktop (EIS) │
                                        │          PipeWire: picture + sound        │
                                        └───────────────────────────────────────────┘
```

- **服务器**是 FreeRDP 的 shadow 服务器，加上一个对接 GNOME 合成器 Mutter 的新后端。它向 Mutter
  请求一个*虚拟显示器*（ScreenCast 的 `RecordVirtual` 调用），再通过 PipeWire 拿到画面、
  damage 区域（damage regions）和光标。输入则通过 libei 送回去。这个后端本身正在上游评审
  （[#14006](https://github.com/FreeRDP/FreeRDP/pull/14006)）。
- **会话管理器**为每个登录的用户启动一个无头 GNOME 会话，所以根本没有物理屏幕：GNOME 只为我们绘制。
- **客户端**是 FreeRDP 的 SDL3 客户端，支持 Linux、macOS 和 Windows。

## 哪里出了问题，在哪里修的

| # | 看到的现象 | 原因 | 文章 |
|---|---|---|---|
| 2 | 屏幕滚动时声音噼啪 | 捕获线程持有声音也需要的锁，同时等着每一帧视频编码完成 | [声音](../gnome-rdp-2-sound/) |
| 3 | 旧的窗口标题、慢了的时钟 | Mutter 的 damage 区域有时会漏掉发生变化的区域 | [残影](../gnome-rdp-3-ghosting/) |
| 4 | 慢速链路上延迟无休止地增长 | 没有流控：发送帧的速度超过了链路的承载能力 | [流控](../gnome-rdp-4-flow-control/) |
| 5 | 一块巨大的显示器，而不是两块 | 服务器把客户端的外包围盒做成了一块虚拟显示器 | [多显示器](../gnome-rdp-5-multi-monitor/) |
| 6 | 指针看不见，点击仍然有效 | Mutter 发来的是空的光标图像，之后再也不发真正的图像 | [自己修，还是上报？](../gnome-rdp-6-fix-or-report/) |

## 我们的工作方式

每个问题都走同样的步骤：

1. **只读观察线上会话，不去动它。** 看线程 CPU 时间、服务器自己的日志、内存映射。
   这个会话是某个人正在用的桌面；重启它是一个决定，而不是调试步骤。
2. **在容器里复现。** 一个无头 GNOME 50 会话、PipeWire、一个测试客户端，以及一套脚本化的负载
   （一个滚动的窗口、一个 1 kHz 的音调、一个每 2 秒打开又关闭的窗口）。每次运行都是一个带版本号的脚本，
   脚本有问题就在旧版本旁边建一个新版本，所以这些文章里的每个数字都能追溯到某次运行。
3. **在同一批运行里测量前后对比。** 主机的负载白天会变，所以我们只比较前后紧挨着跑的结果。
4. **安装之前先在 AddressSanitizer 下运行**，并跑单元测试。
5. **并排安装**，绝不覆盖正在运行的版本，这样回退只需改 systemd drop-in 里的一行。

这些修复都在我们自己构建的 FreeRDP 里，由
[rdp-release-kit](https://github.com/congzhangzh/rdp-release-kit) 发布。其中两个问题出在 GNOME，
而不是 FreeRDP；最后一篇讲的就是：当 bug 在别人的代码里时，该怎么决定。
