---
layout:     post
title:      "Blender 布光"
subtitle:   ""
date:       2020-12-01 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 基础

### 光类型（Blender | 内部灯光类型）
- 自然光 (方向光)
- 发热 -> 发光 -> 人造光源

### 物理灯光属性 - Light Size（灯光大小与软硬阴影的关系）
- 有光就有影
- 灯光面积越大，投影越柔和， Blender
- 主要决定太阳生成软硬投影是由天气决定，大雾多云就会把阳光遮挡住，影子就会软
- 想要得到一个柔和的影子，就设置灯光的大小和半径，或者太阳光的角度  （Blender）
- 阴影的软硬由光强度有关系

![](/img/Blender/01.png)


### 物理灯光属性 - Light Luminance （灯光亮度）
- Luminance 灯光亮度 (cd/m2)
- Power （灯光亮度） 人造光源电力，单位 瓦  (Point  Spot  Area)
- Strength （太阳强度） 自然光照, Sun太阳光 环境光(自然光)
![](/img/Blender/02.png)


### 物理灯光属性 - Light Bounces（直射光线反弹产生间接光）
- Bounces 光线反弹 (直接光，间接光照)
- Cycles 在Light 里面有 Max Bounces 可以控制反弹
- 在 Properties 中 渲染可以设置 Light Paths 
- Eevee 可以在 Irradiance Volume 实现 间接光


## 艺术

### 艺术表现 - Light Bounces（主体的三大面、五大调子）
- 亮面是受光面， 灰面是过度面，暗面是物体自身的投影面，投影的接受面
- 截图04
- 灯不能太亮，会损失细节，要看到五大调子，一定要保证有细节
![](/img/Blender/03.png) ![](/img/Blender/04.png)


### 物理灯光属性 - Color Temperature（色温）
- 6500 K 接近白色
- 描述的是太阳光，太阳色温的变化，把黑体加热，的温度变化 描述为 色温
- ![](/img/Blender/05-0.png)
- ![](/img/Blender/05-1.png)
- ![](/img/Blender/05-3.png)

### 光学理论 - Light Color（光色的冷暖平衡）
- 偏青色的物体，用红光照不亮  （1，0，0） * （0，1，1）
- 颜色不要太纯，饱和度不要那么高
- 冷暖平衡，冷暖对比，非常常用
- ![](/img/Blender/06.png)
- ![](/img/Blender/06-01.png)
- ![](/img/Blender/06-02.png)
- ![](/img/Blender/06-03.png)
- ![](/img/Blender/06-04.png)
- ![](/img/Blender/06-05.png)

### Light Direction | 灯光方位（心理回应+视觉舒服的关键）
- Dirction Top 恐怖，害怕，神秘
![](/img/Blender/07-01.png)
<br>

- Direction + Top + Front 感觉舒服一些
![](/img/Blender/07-02.png)
<br>

- Direction + Top + Back  主要是把人物的边缘勾勒出来
![](/img/Blender/07-03.png)
<br>

- Direction + Front  顺光， 扁平的效果，一般很少去用
![](/img/Blender/07-04.png)
<br>

- Direction + Bottom 心里不舒服
![](/img/Blender/07-05.png)
<br>

- Direction + Top + left 舒服一些
![](/img/Blender/07-06.png)
<br>

- Direction + Top + right 舒服一些
![](/img/Blender/07-07.png)
<br>

- Direct + Left 未必舒服
![](/img/Blender/07-08.png)
<br>

- Direct + Right 未必舒服
![](/img/Blender/07-09.png)
<br>

- Direct + Rembrandt 非常舒服
![](/img/Blender/07-10.png)
<br>

- 灯光高于我们的眼睛，或者头顶的时候，就会舒服起来


### Light Falloff | 灯光衰减（灯光离主体远近的关系及重要特征）
- 灯光衰减，灯光距离主体产生的亮度递减
-  在 Properties 中 工作区 Color Management -> View Transform -> False Color   来观察是否过曝
-  太阳光没有衰减，其他都有



### 专业布光
- Key Light 主光
- Rim 轮廓光， 冷暖
- Fill 补光
- ![](/img/Blender/08-01.png)
- ![](/img/Blender/08-02.png)


### Key Light | 主光的作用及表现力

-  Key Light，摆对位置， 利用好衰减，强度设置要合理，用硬光还是软光

- ![](/img/Blender/09-01.png)

- ![](/img/Blender/09-02.png)

- ![](/img/Blender/09-03.png)


### Fill Light | 辅光的作用及表现力

- 反光板 和 副光 作用是一样的， 是认为通过补灯照明的方式，来完善美学"三大面，五大调子"中的反光调细节

- 亮度不能太亮，不能扰乱主光的投影，冷暖对比

- ![](/img/Blender/10-01.png)

### Rim Light | 轮廓光的作用及表现力

- ![](/img/Blender/11-01.png)
- ![](/img/Blender/11-02.png)
- 比较强烈的光，