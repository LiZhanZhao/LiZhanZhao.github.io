---
layout:     post
title:      "blender eevee TAA Reprojection Initial implementation"
subtitle:   ""
date:       2021-4-16 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2018/4/21  *  Eevee :TAA Reprojection Initial implementation. <br> 

> 
This "improve" the viewport experience by reducing the noise from random
sampling effects (SSAO, Contact Shadows, SSR) when moving the viewport or
during playback.

>
This does not do Anti Aliasing because this would conflict with the outline
pass. We could enable AA jittering in "only render" mode though.

>
There are many things to improve but this is a solid basis to build upon.


> SVN : 2018/3/18  vc14 libs: add missing package folder. 


<br><br>

## 作用
实现 TAA Reprojection

<br><br>
