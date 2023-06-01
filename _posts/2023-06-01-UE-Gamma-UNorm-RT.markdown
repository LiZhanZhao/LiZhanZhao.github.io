---
layout:     post
title:      "UE Gamma + RT格式 + RT精度记录"
subtitle:   ""
date:       2023-06-01 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
typora-root-url: ../
tags:
    - Other

---



# Gamma 简单记录



- 大概的一个流程，个人觉得挺好理解的：

  ![image-20230601162651991](/img/2023/06-01/image-20230601162651991.png)



![image-20230601162555116](/img/2023/06-01/image-20230601162555116.png)

- SRGB 会偏亮，所以
- 



# RenderTarget 格式记录

截帧的时候发现这样的一种格式 ：

- B8A8AR8A8_TYPELESS as UNorm

  UNorm (Unsigned Normalized)： 这种格式将数据存储为无符号归一化整数。在这种情况下，值被映射到0到1的范围内。例如，一个8位UNorm值的范围是从0 (代表0.0) 到255 (代表1.0)。UNorm格式广泛用于各种类型的纹理，包括颜色，光照，法线映射等。

  

- B8A8AR8A8_TYPELESS as SRGB :

  sRGB格式是为了解决人眼感知亮度非线性和显示设备显示亮度非线性的问题。在sRGB格式中，颜色值在存储前进行了非线性的伽马校正（gamma correction）。然后，在读取纹理并在着色器中使用这些值时，硬件将自动进行逆伽马校正，将颜色值从sRGB空间转换回线性空间。这是因为大多数图形处理（如光照计算）在数学上假设颜色值是线性的。sRGB纹理主要用于存储颜色数据。



总的来说，这两种格式都是处理颜色数据的方式，但sRGB格式包含了一个额外的伽马校正步骤，使得颜色值在存储和读取时更符合人眼对亮度的感知。而UNorm则是一种更直接，更通用的方式，将颜色值映射到0和1之间。



# RenderTarget  UNorm 精度

有时候需要注意，给数据给UNorm的RT要保证数据都是在 0-1, 不然数据会丢失