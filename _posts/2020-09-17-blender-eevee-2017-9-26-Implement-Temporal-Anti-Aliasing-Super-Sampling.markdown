---
layout:     post
title:      "blender eevee Implement Temporal Anti Aliasing  Super Sampling"
subtitle:   ""
date:       2021-3-29 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/9/26  *   Eevee : Implement Temporal Anti Aliasing  Super Sampling.<br> 

> This adds TAA to eevee. The only thing important to note is that we need to keep the unjittered depth buffer so that the other engines are composited correctly.


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
实现 TAA 渲染流程，目前没有真正实现TAA

<br>
