---
title: Archlinux+Niri
published: 2026-02-11
description: ''
image: ''
tags: [Arch, Niri]
category: 'Linux'
draft: false
---

# ArchLinux + Niri 自用配置

都说Archlinux好，得学，但是身边没多少人用。恰逢Windows11现在被阿三程序员狠狠蹂躏，所以我还是折腾一下操作系统鄙视链顶端的Arch吧！我会把使用的经验一步步补上来。

### 娱乐软件

#### 音乐播放：Biu

Biu是一个使用Bilibili作为音乐源的播放工具。其AUR包可以直接用以下命令进行安装：

```bash
paru -S Biu
```

#### 视频播放：MPV

视频播放使用MPV，轻量、支持多种编码格式。

```bash
paru -S mpv
```

#### 壁纸与管理：waypaper与awww

均为使用paru安装，后续补充安装流程。

### 办公软件

#### Typora

这个是史无前例的强无敌Markdown编辑器，一直被模仿，但从未被超越。遗憾的是，它现在从免费转为收费了，但其实影响并不大，因为有脚本可以破除这一限制。Anyway，安装与配置流程如下：
```bash
paru -S typora
```

选第一个就好。如果你是使用Niri的话，那么会遇到两个问题：

1. 打开Typora失败。这个是因为Electron的原因，你可以通过添加启动参数`--ozone-platform-hint=auto`得到解决。

2. 中文输入法无效。这个是因为Niri和Fcitx5的兼容问题导致的。

   ```bash
   --ozone-platform-hint=auto
   --enable-wayland-ime
   --wayland-text-input-version=3
   ```

   都加上这些后应该就好了。

