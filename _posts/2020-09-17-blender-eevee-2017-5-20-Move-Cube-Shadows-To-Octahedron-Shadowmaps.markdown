---
layout:     post
title:      "blender eevee Move Cube Shadows To Octahedron Shadowmaps"
subtitle:   ""
date:       2020-12-21 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/5/20  * Eevee: Move cube shadows to octahedron shadowmaps.  <br> 
We render linear distance to the light in a R32 texture and store it into an octahedron projection inside a 2D texture array.  <br> 
This render the sampling function much more simpler and without edge artifacts. <br>  

> SVN : 2017/4/5  MSVC 2015 windows x64 (vc140) Alembic 1.7.1



## 作用 




## 编译

- 如果之前用过命令 make.bat full nobuild 2015 生成vs工程 的话，需要重新生成。


## 效果



