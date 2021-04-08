---
layout:     post
title:      "blender eevee Volumetrics Add Volume object support"
subtitle:   ""
date:       2021-4-8 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/10/27  *   Eevee : Volumetrics: Add Volume object support. <br> 

> This is quite basic as it only support boundbing boxes.
But the material can refine the volume shape in anyway the user like.

>
To overcome this limitation, a voxelisation should be done on the mesh (generating a SDF maybe?) and tested against every volumetric cell.

> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
物体可以进行 Volume


## 效果
*想要看到效果, Viewport Samples(TAA Sample) 要先设置为 1*
![](/img/Eevee/Volumetrics+/02/1.png)

<br><br>


