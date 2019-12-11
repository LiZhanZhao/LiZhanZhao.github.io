---
layout:     post
title:      "Separable Subsurface Scattering"
subtitle:   ""
date:       2019-12-10 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Unity
---


> Separable Subsurface Scattering  
简称4S, 主要用来实现次表面散射效果，这个4S技术在各种游戏引擎都会使用到，例如Unreal4, Marmoset Toolbag 3, Blender Eevee 等。
  
## 收集学习资料！

### 入门资料
在google直接搜索**Separable Subsurface Scattering**关键字都会搜索到很多资料，但是个人觉得下面的的比较好。  
<br>
[Separable Subsurface Scattering入门](http://www.iryoku.com/separable-sss/)  
<br>
上面的链接可以直接下载到<br><br>[理论PDF](http://www.iryoku.com/separable-sss/downloads/Separable-Subsurface-Scattering.pdf)<br><br>[github上的Demo源码](https://github.com/iryoku/separable-sss)<br><br>可以阅读它的Demo代码来学习**Separable Subsurface Scattering**的实现思路，非常不错。

### 进阶资料
当你学习完入门的资料的时候，高阶一点的资料就十分必要了，因为上面的资料只是模拟了皮肤的次表面散射效果，假如我们模拟牛奶，玉石的效果，就不知道怎么做了。  
<br>[Separable Subsurface Scattering进阶 ](https://users.cg.tuwien.ac.at/zsolnai/gfx/separable-subsurface-scattering-with-activision-blizzard/)给出来对应的解决方案。<br>    
同样的在这个网站上的Resource部分，可以下载到很多资源，包括  
<br> [demo的源码](https://users.cg.tuwien.ac.at/zsolnai/wp/wp-content/uploads/2014/12/s4_cpp_source.zip)  
<br> 然后就可以愉快地玩耍了。

### 注意事项
入门资料和进阶资料可以先下载他们编译好的demo看看效果，然后再下载源码编译，目前使用vs2012可以成功编译他们的demo源码。

> 记录编译<Separable Subsurface Scattering demo>工程遇到的问题
> 1. Separable Subsurface Scattering demo 可以从 https://github.com/iryoku/separable-sss 下载下来，但是默认的sln 是 vs2010 版本的。
> 2. 解压，记得不要有中文路径，我这边直接安装了vs2012，直接打开，提示升级，然后点击升级，或者直接去 属性-> 配置属性-> 常规-> 平台工具集 , 设置 v110
> 3. 这个时候可以正常运行，但是最后会提示 Failed creating the Direct3D derive ，通过调式，可以发现 log 是这样的：
> D3D11CreateDevice: Flags (0x2) were specified which require the D3D11 SDK Layers for Windows 10, but they are not present on the system.
These flags must be removed, or the Windows 10 SDK must be installed.
Flags include: D3D11_CREATE_DEVICE_DEBUG
> 4. 奔溃报错直接定位在 DXUT.cpp  第3837行 不使用 pNewDeviceSettings->d3d10.CreateFlags，改为0，就可以直接不提示报错。

## 在Unity3D实现
学习完以上的资料的时候，最喜欢的就是重复造轮子，按照自己的理解在Unity里面实现一遍**Separable Subsurface Scattering**。下面是Unity的效果 :
<br> 
<br> 
> 整体效果
![](/img/SSSS/1.gif)  


上面的效果也实现了软阴影  

> 软阴影效果
![](/img/SSSS/2.gif)  


对比没有SSSS和有SSSS效果
> 对比效果
![](/img/SSSS/3-1.png)  
![](/img/SSSS/3-2.png)  

## 总结
- 模拟次表面散射效果，可以优先考虑使用SSSS。
- 计算透射的厚度需要光源的Shadowmap，光源越多，涉及到的Shadowmap越多，性能消耗越大。
- SSSS的是屏幕后期效果，最影响效果的是后期的两个模糊操作(不是普通的模糊，具体的可以看Demo的代码)。