---
layout:     post
title:      "Substance Painter Subsurface Scattering 注意事项"
subtitle:   ""
date:       2020-01-10 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - SubstancePainter
---
## Subsurface Scattering 效果
![](/img/SP/Jade.png)

## 官网资料
[官网文档](https://docs.substance3d.com/spdoc/subsurface-scattering-172818701.html)有比较详细的说明，下面简单记录一下。

## 流程
- In the Texture Set Settings add a Scattering channel if not already present. (在Texture Set Settings添加Scattering通道)  
<br>
- Enable the main Subsurface scattering setting in the Display Settings.(在Display Settings打开Subsurface scattering选项)  
<br>
- In the Shader Settings window with default shaders can be found a "SSS Parameters" group with two settings.
Change the scale and the color to fit the target material.(在Shader面板中找到Scatting相关的参数进行调节)  
<br>
- The Subsurface scattering effect works well but may look strange if alone. 
Enabling shadow can help the final look in the viewport and improve the realism of the final material.(在Display Settings中打开阴影)  

## 注意

### 模型大小
模型的大小会影响到Scattering的效果，可以拿SP官方材质球模型在3dmax中对对大小。

### Curvature Map 和 Thickness Map 生成
这两个图非常影响Scattering的效果，一定要注意，模型的UV不能重叠，除非是那种对称的模型，例如脸。

## SP 实时 Subsurface Scattering 

### 相关技术
[这里](https://docs.substance3d.com/spdoc/subsurface-parameters-172818973.html)有提到Substance Painter real-time subsurface 是用屏幕后期实现的。

> Substance Painter real-time subsurface implementation is a screen-space subsurface scattering effect. The parameters to control it are explained in this page.
The current implementation is based on the "Approximate Reflectance Profiles for Efficient Subsurface Scattering" method [published by PIXAR](http://graphics.pixar.com/library/ApproxBSSRDF/).


因为很多实时Subsurface Scattering都会存在那么一点点噪点的问题，这里就是SP处理噪点的方法。

> The amount of noise can also be reduced by enabling [Temporal Anti-Aliasing](https://docs.substance3d.com/spdoc/camera-settings-172818743.html) without increasing the amount of samples.
