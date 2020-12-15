---
layout:     post
title:      "blender eevee Initial Depth Of Fiedld"
subtitle:   ""
date:       2020-12-14 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/5/10  * Eevee: Initial Depth Of Field commit. <br>  

> SVN : 2017/4/5  MSVC 2015 windows x64 (vc140) Alembic 1.7.1





## 作用 
Bloom 景深效果



## 编译

- 直接编译


## 效果展示

![](/img/Eevee/DepthOfField/Dof.png)


## 算法理论基础

### 1. lens equation (成像公式)
参考 [成像公式推导](https://baike.baidu.com/item/%E6%88%90%E5%83%8F%E5%85%AC%E5%BC%8F)

推到凸透镜的成像规律 : 1/u+1/v=1/f（即 : 物距的倒数与像距的倒数之和等于焦距的倒数。）

![](/img/Eevee/DepthOfField/lens_equation.png)

【题】如右图 ，用几何法证明1/u+1/v=1/f。<br>  
【解】∵△ABO∽△A'B'O				<br>  
∴AB:A'B'=u:v					<br>  
∵△COF∽△A'B'F					<br>  
∴CO:A'B'=f:(v-f)					<br>  
∵四边形ABOC为矩形					<br>  
∴AB=CO								<br>  
∴AB:A'B'=f:(v-f)					<br>  
∴u:v=f:(v-f)						<br>  
∴u(v-f)=vf							<br>  
∴uv-uf=vf							<br>  
∵uvf≠0							<br>  
∴(uv/uvf)-(uf/uvf)=vf/uvf			<br>  
∴1/f-1/v=1/u					<br>  
即：1/u+1/v=1/f					<br>  


### 2. Determining a circle of confusion diameter from the object field 

参考 [Circle of confusion](https://en.wikipedia.org/wiki/Circle_of_confusion#Determining_a_circle_of_confusion_diameter_from_the_object_field)

![](/img/Eevee/DepthOfField/coc.png)

To calculate the diameter(直径) of the circle of confusion in the image plane for an out-of-focus subject, one method is to first calculate the diameter of the blur circle in a virtual image in the object plane, which is simply done using similar triangles, and then multiply by the magnification(放大倍数) of the system, which is calculated with the help of the lens equation.

The blur circle, of diameter C, in the focused object plane at distance S1, is an unfocused virtual image of the object at distance S2 as shown in the diagram. It depends only on these distances and the aperture diameter A, via similar triangles, independent of the lens focal length:

![](/img/Eevee/DepthOfField/Exp-01.svg) 
(以上可以利用相似三角形计算得到)

The circle of confusion in the image plane is obtained by multiplying by magnification m:
![](/img/Eevee/DepthOfField/Exp-02.svg) 

where the magnification m is given by the ratio of focus distances:
![](/img/Eevee/DepthOfField/Exp-03.svg) 

Using the lens equation we can solve for the auxiliary variable f1:
![](/img/Eevee/DepthOfField/Exp-04.svg)

which yields
![](/img/Eevee/DepthOfField/Exp-05.svg)

and express the magnification in terms of focused distance and focal length:
![](/img/Eevee/DepthOfField/Exp-06.svg)



which gives the final result:
![](/img/Eevee/DepthOfField/Exp-07.svg)

(f :  focused distance  <br>  
S1 :  focal length     <br>  
S2 :  unforcal length
 )


### 3. linear_depth
主要是Shader通过这个函数，把 buffer的深度，变换成 view 空间
参考<br>  
[linear_depth](http://web.archive.org/web/20130416194336/http://olivers.posterous.com/linear-depth-in-glsl-for-real)
<br>  
[projectionmatrix](http://www.songho.ca/opengl/gl_projectionmatrix.html)


the link between eye-space Z (z_e below) and normalised device coordinates (NDC) Z (z_n below). From there, we have

A   = -(zFar + zNear) / (zFar - zNear);
B   = -2*zFar*zNear / (zFar - zNear);
z_n = -(A*z_e + B) / z_e; // z_n in [-1, 1]

Note that the value stored in the depth buffer is actually in the range [0, 1], so the depth buffer value z_b is:

z_b = 0.5*z_n + 0.5; // z_b in [0, 1]

If we have rendered this depth buffer to a texture and wish to access the real depth in a later shader, we must undo the non-linear mapping above:

z_e = 2*zFar*zNear / (zFar + zNear - (zFar - zNear)*(2*z_b -1));

(经过代入计算, 感觉自己计算出来的结果和 以上的公式 多一个负号)


计算得到

```
float linearize_depth(float d,float zNear,float zFar)
{
    return zNear * zFar / (zFar + d * (zNear - zFar));
}
```





## Dof 算法思路
todo...


