---
layout:     post
title:      "Ninja Ripper + 夜神模拟器"
subtitle:   ""
date:       2020-08-21 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Other
---

## 为什么使用 Ninja Ripper
- GPA 导出的模型没有UV，所以尝试使用 Ninja Ripper 提取 DX游戏资源。

## 环境
环境：Win10, Nox(夜神模拟器)6.2.8.3, Ninja Ripper 5.2.0.0

## 参考
[地址视频](https://www.youtube.com/watch?v=Dp90rbVntb0)

## 下载

[Ninja Ripper](https://gamebanana.com/tools/5638) 
<br>
[noesisv](https://richwhitehouse.com/index.php?content=inc_projects.php)


## 步骤
- 在模拟器中安装游戏（运行夜神模拟器，把游戏的apk包拖入进去自动安装），在右上角的设置中，将显卡渲染模式设置更改为急速模式(DirectX)。关闭模拟器
- 运行Ninja ripper
![](/img/NinjaRipper/1.png)
- 设置工具的参数
![](/img/NinjaRipper/2.png)
- 修改快捷键（防止与模拟器的快捷键冲突）
![](/img/NinjaRipper/3.png)

- 点击Run，会启动夜神模拟器，在模拟器中，运行游戏
- 在模拟器中进入游戏之后，在场景中，按下F3键，导出当前场景渲染的所有贴图，按下F10键，导出当前场景渲染的所有模型。(注意，按一次就卡一下就相当于导出一次了)
- 在导出的目录中，找到 带 ***_Nox.exe_***的目录就可以找到导出的Texture 和 Mesh, Mesh 是.rip 文件 , 
- 用noesisv可以直接查看rip文件, noesisv 可以直接导出fbx，这个fbx是带uv的。
  

## 注意
- Ninja Ripper 对 夜神模拟器的版本会有要求，现在尝试夜神模拟器 5.2.0.0 可以成功导出，但是夜神模拟器 5.2.0.0版本较低，无法运行一些游戏
- 在[地址视频](https://www.youtube.com/watch?v=Dp90rbVntb0) 的评论中发现，In 2020 I noticed that version 6.x of NOX is not working with Ninja Ripper due to some changes ，可以多尝试不同的版本。