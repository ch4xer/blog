---
title: Wayland下Fcitx5切换窗口焦点导致无法输入问题
author: ch4ser
date: 2025-10-13T17:10:55+08:00
categories:
  - 技术
---

自己使用Linux作为日常操作系统这么久了，有一个问题一直困扰着我，那就是有的时候fcitx5会无法输入中文，这个问题非常的恼人，时间长了后我发现此问题基本出现在窗口切换的时候，特别是electron应用窗口。一个触发案例是这样的（以下窗口均为wayland窗口，桌面环境为plasma wayland）:

1. 打开chrome和kitty
2. 在kitty中使用fcitx5输入中文
3. 切换到chrome窗口
4. 在chrome中使用fcitx5输入中文
5. 切换到kitty窗口，发现无法输入中文
6. 尝试在kitty中切换输入法，发现无法切换

在英语世界里，基本上无法找到和我类似的问题，导致这个问题一直困扰着我。直到有一天在arch linux cn论坛上看到一个[帖子](https://forum.archlinuxcn.org/t/topic/14640)，我惊奇地发现这个问题所描述的症状和我遇到的完全一致，然而作者的问题源自于xwayland和wayland之间的切换，而我使用的是纯wayland环境，不过作者和我说可能是text-input协议的问题，我尝试将chrome和其他electron应用的text-input协议切换到v1之后问题消失了！因为目前大部分应用都还在使用text-input-v1协议，而我的chrome在使用text-input-v3协议，所以导致了这个问题。

解决这个问题的方法是将electron应用的text-input协议切换到v1。

```
--enable-wayland-ime
--wayland-text-input-version=1
```
