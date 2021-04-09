---
layout:     post
title:      "blender eevee Initial Separable Subsurface Scattering implementation"
subtitle:   ""
date:       2021-4-9 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/10/27  *   Eevee : Initial Separable Subsurface Scattering implementation. <br> 

> How to use:
- Enable subsurface scattering in the render options.
- Add Subsurface BSDF to your shader.
- Check "Screen Space Subsurface Scattering" in the material panel options.

> This initial implementation has a few limitations:
- only supports gaussian SSS.
- Does not support principled shader.
- The radius parameters is baked down to a number of samples and then put into an UBO. This means the radius input socket cannot be used. You need to tweak the default vector directly.
- The "texture blur" is considered as always set to 1


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
实现初级版本的SSS效果
<br><br>

## 效果

![](/img/Eevee/SSS/01/1.png)
![](/img/Eevee/SSS/01/2.png)

<br><br>
