---
layout:     post
title:      "RenderDoc 截帧 夜神模拟器"
subtitle:   ""
date:       2023-1-29 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - renderdoc
---

## 参考
[这里](https://zhuanlan.zhihu.com/p/403453085)

## 步骤
- 先下载 [renderdoc](https://renderdoc.org) 和 [夜神模拟器](https://www.yeshen.com/)
<br><br>
- 夜神模拟器 先设置渲染模式 为 DirectX, 性能设置 -> 基础模式 DirectX, 记得不是opengl
<br><br>
- 在 RenderDoc Tools->Settings->General 里面找到 Allow global process hooking 并勾选。
<br><br>
- 找到模拟器的 NoxVMHandle.exe, 例如路径在  C:\Program Files (x86)\Bignox\BigNoxVM\RT 中，可以打开模拟器，然后运行一个游戏，然后任务管理器里面按照 CPU 使用排序，排在最前面的就是，右键点击之，选择打开文件所在位置。
<br><br>
- 在 RenderDoc 的 Launch Application 页面里面。Executable Path 选择刚才找到模拟器的 NoxVMHandle.exe
<br><br>
- 然后在下面 Global Process Hook 里面点 Enable Global Hook，如果提示需要 Administrator 启动就确定以后再点 Enable Global Hook 按钮
<br><br>
- 退掉所有模拟器，然后重新启动模拟器，这时候应该能看到模拟器画面左上角已经显示 RenderDoc 的信息了。
<br><br>
注意：模拟器一定要退干净，有时候模拟器界面关掉了，核心还在后台运行。可以在任务管理器里面查看模拟器的核心是否还在运行，还在运行的话用任务管理器杀掉。
- RenderDoc File 菜单 Attach to Running Instance , 在 localhost 下面可以看到模拟器核心程序，选中并点击 Connect to app ，之后就正常抓帧即可

