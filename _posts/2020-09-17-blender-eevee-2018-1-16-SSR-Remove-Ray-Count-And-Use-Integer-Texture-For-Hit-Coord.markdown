---
layout:     post
title:      "blender eevee SSR Remove ray count and use integer texture for hit coord."
subtitle:   ""
date:       2021-4-15 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2018/1/16  *   Eevee : SSR: Remove ray count and use integer texture for hit coord. <br> 

> 
Using GL_RG16I texture for the hit coordinates increase tremendously the precision of the hit.
The sign of the integer is used to 2 flags (has_hit and is_planar).
We do not store the depth and retrieve it from the depth buffer (increasing bandwith by +8bit/px).
The PDF is stored into another GL_R16F texture.

>
We remove the raycount for simplicity and to reduce compilation time (less branching in refraction shader).



> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
优化SSR效果，这里的效果觉得比之前的要好

<br><br>

## 效果

![](/img/Eevee/SSR/10/1.png)
<br><br>
![](/img/Eevee/SSR/10/2.png)

<br><br>
