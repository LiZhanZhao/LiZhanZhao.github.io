---
layout:     post
title:      "blender eevee Transparency Add support for blend ADD and MULTIPLY."
subtitle:   ""
date:       2021-2-21 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/10  *  Eevee: Transparency: Add support for blend ADD and MULTIPLY .<br> 

> This introduces a new transparency pass.
It bypass the radial distance encoding in alpha for the transparent shaders.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/BlendAddAndMultiply/1.png)
![](/img/Eevee/BlendAddAndMultiply/2.png)
![](/img/Eevee/BlendAddAndMultiply/3.png)


## 作用
blend ADD 和 MULTIPLY 的实现

## 编译
- 重新生成SLN
- git 定位到  2017/7/10  * Eevee: Transparency: Add support for blend ADD and MULTIPLY .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

