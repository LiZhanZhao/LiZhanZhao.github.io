---
layout:     post
title:      "blender eevee SSS Add Translucency support"
subtitle:   ""
date:       2021-4-12 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/11/22  *   Eevee : SSS : Add Translucency support. <br> 

> 
This adds the possibility to simulate things like red ears with strong backlight or material with high scattering distances.

>
To enable it you need to turn on the "Subsurface Translucency" option in the "Options" tab of the Material Panel (and of course to have "regular" SSS enabled in both render settings and material options).
Since the effect is adding another overhead I prefer to make it optional. But this is open to discussion.

>
Be aware that the effect only works for direct lights (so no indirect/world lighting) that have shadowmaps, and is affected by the "softness" of the shadowmap and resolution.

> Technical notes:
This is inspired by http://www.iryoku.com/translucency/ but goes a bit beyond that.
We do not use a sum of gaussian to apply in regards to the object thickness but we precompute a 1D kernel texture.
This texture stores the light transmited to a point at the back of an infinite slab of material of variying thickness.
We make the assumption that the slab is perpendicular to the light so that no fresnel or diffusion term is taken into account.
The light is considered constant.
If the setup is similar to the one assume during the profile baking, the realtime render matches cycles reference.
Due to these assumptions the computed transmitted light is in most cases too bright for curvy objects.

>
Finally we jitter the shadow map sample per pixel so we can simulate dispersion inside the medium.
Radius of the dispersion is in world space and derived by from the "soft" shadowmap parameter.
Idea for this come from this presentation http://www.iryoku.com/stare-into-the-future (slide 164).


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
实现SSS Translucency 效果
<br><br>

## 效果

![](/img/Eevee/SSS/01/1.png)
![](/img/Eevee/SSS/01/2.png)

<br><br>
