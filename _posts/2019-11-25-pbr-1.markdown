---
layout:     post
title:      "PBR Shader"
subtitle:   ""
date:       2019-11-25 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Unity
---


> 什么是PBR (Physically Based Rendering)？
  PBR是一种着色和渲染技术，用于更精确的描述光如何与物体表面互动。PBR的优势：（1）方法论和算法基于精确的计算公式，免除创作表面的猜想过程。
（2）在任何光照环境都能表现出正确的结果（3）为不同的艺术家，提供统一的工作流程



## 写一个PBR Shader！
工作以来几乎每一天都在跟PBR打交道，如果不深入了解什么是PBR，自己干起活来有点虚，如果不知道原理，想改写里面的实现都改不动，所以打算通过自己写一个PBR Shader来进一步了解PBR的。

### 找资源 (Sketchfab)
在[Sketchfab网站](https://sketchfab.com/feed)中可以看到很多很有趣的效果，几乎所有的效果都是基于PBR的，而且[Sketchfab网站](https://sketchfab.com/feed)可以免费提供模型进行下载，随便找到一个模型[模型](https://sketchfab.com/3d-models/dredd-74a05141476d4f6f8ebf83d9636923c5)进行下载，就可以得到Obj和所有的需要用到贴图，十分的方便。可以看看这个模型的效果 :![](/img/PBR-1/1.png)

### 找参考 (Filament)
想要深入学习PBR，可以直接阅读Unity的Standard Shader 源码，也可以直接阅读虚幻4的关于PBR的Shader代码，前两个的复杂度不是一般地大，本人更加倾向于一些轻度一点渲染框架，这个时候主角出现了，[Filament](https://github.com/google/filament) .  
<br>
可以看到github中对Filament的介绍 : **Filament is a real-time physically based rendering engine for Android, iOS, Linux, macOS, Windows, and WebGL. It is designed to be as small as possible and as efficient as possible on Android.**  
<br>
比较吸引我的就是Filament的效果, 很PBR ,并且filament是**开源**的
![](/img/PBR-1/2.jpg)


### 在Unity3D实现
通过阅读的[Filament](https://github.com/google/filament)源码，学习了解到[Filament](https://github.com/google/filament)的PBR的框架，所以打算直接在Unity中重复造一次轮子，按照自己的理解在Unity里面实现一遍PBR。下面是Unity的PBR效果 :  
![](/img/PBR-1/4.gif)  
*以上 PBR Shader 没有依赖过Unity的任何东西*


## 总结
- [Filament文档](https://google.github.io/filament/Filament.html) 是个好东西，可以仔细看看加深自己对PBR的原理的理解。
- All contribution = Indirect contribution + Direct contribution  
  Indirect contribution = Diffuse indirect + Specular indirect  
  Direct contribution = Diffuse direct + Specular direct  
  

