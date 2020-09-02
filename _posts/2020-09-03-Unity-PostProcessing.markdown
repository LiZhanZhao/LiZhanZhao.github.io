---
layout:     post
title:      "Unity Post Processing"
subtitle:   ""
date:       2020-09-03 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Unity
---
## 环境
Unity 2018.4

## 安装
[教程](https://docs.unity3d.com/Packages/com.unity.postprocessing@2.2/manual/Installation.html)
<br>
>
- 打开 Window > Package Manager 菜单，选择 Postprocessing 就可以进行安装

## 源码
[仓库地址](https://github.com/Unity-Technologies/PostProcessing)

## 快速开始
[教程](https://github.com/Unity-Technologies/PostProcessing/wiki/Quick-start)

>
- Post-process Layer, 在Camera上添加 ***Component -> Rendering -> Post-process Layer*** 组件，这个组件需要设置Layer，也可以设置Anti-aliasing, 参数具体可以参考[教程](https://github.com/Unity-Technologies/PostProcessing/wiki/Quick-start)<br><br> 
>
- Post-process Volumes, 在任何一个GameObject上添加 ***Component -> Rendering -> Post-process Volume***, 在 ***Post-process Volume***上新建Profile. <br><br> 
>
- 在Post-processing Profile 进行调整后处理。

