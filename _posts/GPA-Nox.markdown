---
layout:     post
title:      "Intel GPA + 夜神模拟器"
subtitle:   ""
date:       2019-10-2 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Other
---




## 环境
环境：Win10，Intel GPA 2019R2，Nox(夜神模拟器)6.2.8.3

## 下载
夜神模拟器下载地址 : [这个]( https://www.bignox.com/)



## 步骤
- 先运行Nox，在设置中将显卡渲染模式改成DirectX并保存.(确保在GPA的浮动框显示的API是DX)
<br>
<br>

- 打开GraphicsMonitor，在下方选择Nox程序的位置，然后打开设置面板，将Auto-detect launched applications切换为On。然后点击右下按钮运行Nox。Auto-detect lanucned applications每次打开都默认都为Off，因此每次一定要手动切换为On，否则GPA无法识别应用。（Auto-detect launched applications切换为On，在GraphicsMonitor启动 Nox.exe , 会有NoxVMHandler.exe）
<br>
<br>

- 开始选择需要截帧的游戏
<br>
<br>

- 打开System Analyzer，点击Connect后，选择NoxVMHandler.exe. 
<br>
<br>

- 之后就进入到了分析界面，接着在你想要抓帧的地方点击照相的按钮。
<br>
<br>

- 抓取完成后，最后打开Graphics Frame Analyzer，你应该能在主界面看到刚刚抓帧的信息
<br>
<br>



## 补充
NoxVMHandle.exe 存在地方 和 GPA启动NoxVMHandle.exe的命令行：

"C:\Program Files (x86)\Bignox\BigNoxVM\RT\NoxVMHandle.exe" --comment nox --startvm 00000000-f2f7-4ca2-a15b-000000000051 --vrde config --romx 0 --romy 0 --romwidth 1280 --romheight 720 --renderport 55001 --callbackport 54001 --rendertype directx --hwnd 396362 --renderpath libdx.dll --passthrough NOXa.dll



  

