---
layout:     post
title:      "blender eevee Refactor Shadow System"
subtitle:   ""
date:       2021-3-21 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/9/1  *   Eevee : Refactor Shadow System <br> 

> 
- Use only one 2d texture array to store all shadowmaps.
- Allow to change shadow maps resolution.
- Do not output radial distance when rendering shadowmaps. This will allow fast rendering of shadowmaps when we will drop the use of geometry shaders.


> SVN : 2017/8/17  [windows] add numpy headers to python include dir, required for D2716


## 效果
![](/img/Eevee/Shadow-2/1.png)

## 作用
重构影子系统
