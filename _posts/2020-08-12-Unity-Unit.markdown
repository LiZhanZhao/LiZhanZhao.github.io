---
layout:     post
title:      "Unity 3DMax Blender 单位同步"
subtitle:   ""
date:       2020-08-12 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Unity
---

## Unity
单位为 *米*


## 3DMax 与 Unity 单位一致步骤

### 3Dmax 设置
- 设置3Dmax的单位为米，菜单 Customize -> Units Setup 设置
![](/img/Unity/Unit/1.png)

- 设置3Dmax的网格，菜单 Tools -> Grids and Snaps -> Grids and Snaps Settings
![](/img/Unity/Unit/2.png)


## Blender 与 Unity 单位一致步骤
- 把单位设为米
![](/img/Unity/Unit/3.png)

- 发现在unity里你的模型缩放不是1，1，1, 那么应如上图中这样操作
![](/img/Unity/Unit/4.png)

- 导出
![](/img/Unity/Unit/5.png)
