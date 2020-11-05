---
layout:     post
title:      "Blender 建模基础总结"
subtitle:   ""
date:       2020-10-29 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 物体模式 -> 编辑模式
- tab 快速 切换 Object Mode -> Edit Mode
- 多个物体进入编辑模式，按住shift + 鼠标左键点选  + tab, 可以把多个物体的UV整合在同一张图里面
- Ctrl + Tab 激活饼菜单

## 点、边、面、体元素
- BMesh 系统 = 多边形建模系统
- 切换到edit mode 之后，1，2，3 可以切换到 点线面的选择
- 打开显示面的中心点 -> OverLays -> Mesh Edit Mode -> 勾选Center， 
- Ctrl + L 选择体元素

## 元素的连续选择
- Alt + 鼠标右键 = 循环，环形，连续选择
- Shift + Alt + 鼠标右键 = 增加/减少 循环，环形，连续选择
- 根据元素走向连选
- Shift + 鼠标右键 依次增加/减少选择
- 按住Alt + 按住鼠标中键上下左右移动  快速切换视图 ，前后左右视图
- 鼠标左键先选择最左侧的一个面 -> 然后按住 Shift + Ctrl + 鼠标左键选择最右侧的一个面   <br>      即可选择同一个平面内的所有的面
- 鼠标左键先选择最左侧的一个面 -> 然后按住 Ctrl + 鼠标左键选择最右侧的一个面         <br>       即可选择头尾路径之间的面

## 高级选择选项

- ALT + Z 激活 X-Ray 透明显示模式，在透明模式下可以选择物体背面的元素
- Ctrl 加  +号键(小键盘)   递增选择(从选择的元素为中心向外增加)
- Ctrl 加  -号键(小键盘)  递减选择(从已选择的元素边缘向中心外减)
- Shift + G  激活相似属性的选项   select -> Selecty Similar 
- Ctrl + I 反选
- Shift + Ctrl + M  镜像选择


## 视口参数
- 小键盘 .  最大化视图显示选择的元素
- HOME 键，将场景中的所有物体最大化视图显示
- / 键 将选择的物体隔离并最大化视图显示

## X 详解删除选项
按下X键
- Vectices 点
- Edges 边
- Faces 面
- Only Edges & Faces 仅删除选中的边&面
- Only Face 仅删除选中的面，点与边元素保留

- Dissolve Vertices  融并点
- Dissolve Edges 融并边
- Dissolve Faces 融并面

- Limited Dissolve 根据角度融并面        (可以做一些Lowpoly的事情)

- Edges Collspse 将选中的边塌陷为一个点
- Edges Loops 以选中的边为中心塌陷为一个面



## G\R\S 变换及高级操作
- Alt + S 延法线缩放
- S 激活缩放 -> X/Y/Z 选择要拉平的轴向 -> 0 输入 -> 选择的元素就会被拉成一个平面
- GG 沿边滑动   (Shift + 鼠标滑动 = 慢速微调)  
(Alt 激活滑动边的参考线)
- Shift + Alt + S 将选择的面转化为圆形


## O 范围衰减操作要领
- O 激活范围衰减<br>
激活以后，滚动鼠标中键改变衰减的范围<br>
滚动鼠标中键视图中的圆形越大就意味着衰减的范围越大


## SHIFT+TAB 捕捉系统
- Shift + Tab 捕捉<br>
Increment 捕捉到栅格<br>
Vertex 捕捉到顶点<br>
Edge 捕捉到边<br>
Face 捕捉到面<br>
Volume 捕捉到体积<br>


## 认识 Normal 法线
- Shift + N  法线朝向设置
- OverLays 面板里可以显示点，线，面的法线

## 设置建模参考背景
根据视图创建的
- 先去前视图
- Shfit + A -> Image -> Reference  <br>
  旋转都可以看到
- Shfit + A -> Image -> Background <br>
  在某一个视图才看到
- 可以在右边的属性框进行设置 图片的透明度和是否双面什么的


## 全屏
- Ctrl + Alt + Space  全屏
- Overlays 窗口可以不显示栅格和轴

## 认识多边形细分建模
- 细分 Subdivision Surface 控制 View，<br>
	Ctrl + 0 原始文件<br>
	Ctrl + 1 1级细分<br>
	Ctrl + 2 2级细分<br>
	Ctrl + 3 3级细分<br>
	Ctrl + 4 4级细分<br>
- 镜像可以 Mirror 
- 右边 Mesh Options 里面有 AutoMergeEditing  自动焊接顶点  (Context 中 Auto Merge )
	在编辑模式下，两个独立的顶点如果重合在一起的时候，就会自动焊接为一个顶点

## 合并和分离
- 在物体模式下 Ctrl + J  合并多个独立的物体为一个物体
- 在编辑模式下，P 将选择的元素分离成一个独立的物体 
- Y 拆分选择的面，也是同一个物体


## 打开点线面菜单
- Ctrl + V 点菜单
- Ctrl + E 边菜单
- Ctrl + F 面菜单
- 可以转换为 三角面 (w -> Triangulate Faces   Ctrl + T)


## E 挤压命令 | ALT+E 挤压菜单
- Extrude along Normal

- Offset Even 偏移锁定 <br>
将挤压出的新元素进行修正，以便得到均匀的厚度

- Extrude Individual <br>
将选择的所有的面都进行独立挤压，选择多少面就挤压多少

- E 激活挤压，输入具体的数值，比如 2 回车确定

- Alt + E 激活挤压菜单 选择其中的一个挤压命令 -> 激活选择的命令


##  自由挤压命令
- CTRL+鼠标右键
- 按住Ctrl不放，移动鼠标位置，然后点击鼠标右键
- 在点元素下，挤压出边
- 在边元素下，挤压出面
- 在面元素下，挤压出体


## 物体融成一个点
- x 键 -> Edges Collspse 将选中的边塌陷为一个点


## 内插面 Insert face
- 快捷键 I 键
- 双击 I 各个面内插

## CTRL+B | 倒角
- Ctrl + B 激活倒角 <br>
移动鼠标预览倒角的大小 <br>
滚动鼠标中键设置片段数 <br>
输入精确的数值回车确定或者点击鼠标左键确认


- V 激活顶点的倒角 <br>
滚动鼠标中键设置片段数 <br>
或点击鼠标左键确认<br>

- P 激活倒角轮廓设置<br>
左移动鼠标为倒角内陷<br>
右移动鼠标是倒角外展<br>
滚动鼠标中键设置片段数<br>
输入精确的数值回车确定或者点击鼠标左键确认<br>


## CTRL+R | 环切   Loop Cut
- 移动鼠标预览环切位置, 滚动鼠标中键设置片段数 or 输入精确的数字
- 鼠标左键确认环切, 移动鼠标预览环切后的位置
- 鼠标左键确认位置
- 鼠标右键还原位置


## K | 切割  knife
- 空格键结束并确认切割
- K 激活切割工具, 按住Ctrl 可捕捉到边的中心点位置，将鼠标移到中心点位置左键点击即可切割, 空格确认最终的切割

- J键 在面中的两个点连接并创建一条边

- Bisect 分割，将物体一分为二

- Shift + Space  打开左侧工具栏为浮动选项栏

## Spin | 旋转成型
- 主要用于左管道
- 可以结合 Ctrl + 鼠标左键， 连线做一些形状，然后再配合 Spin来做效果

## F | 连线与封面
- F 两点之间连成线, 三点以上连成面, 两条及以上边连成面

## J 连接并切割
- 两点之间连成线, 通过的面中进行切割

## M | 合并焊接顶点
- 2.83 快捷键 = Alt M ,  2.83 之后 = M
- At First 合并到最先选择的点
- At Last 合并到最后选择的点
- At Cursor 合并到游标的位置
- At Center 合并到所有选择点的中心
- Collapse 塌陷为一个顶点


## Object Data | 认识物体数据
- 可以添加UV map
- 添加Vertex Color

## Bridge | 桥接边
- 桥接， Edge context Menu （w） -> Bridge Edge Loop


## SHIFT+E | 折痕
- 细分的时候，维持硬边效果， 配合细分使用
- -1 不起作用
- 0 折痕


## Solidify | 将面实体化
- Face -> Solidify Faces 
- Wife Frame 线框实体

## Boolean | 布尔运算
- Ctrl + F 打开面选择, 选择布尔运算菜单
- Ctrl + J 进行合并物体才可以进行布尔运算
- Difference 差集 减去选择元素
- Union 并集 将元素进行合并并减去相交的部分
- Intersect 交集 将元素之间相交的地方保留

## Units | 单位 | 精准尺寸比例
- 场景 -> Units 


## 展UV
- 在编辑模式下，选择物体点击 U
- Mark Seam 编辑缝合边来改变UV，类似于需要哪一些边来展开cube的原理
- 可以在属性窗口的View里面， OverLays 点击Stretching 来检查UV的好坏

## texture paint
- 可以画UV的贴图，在Tool里面可以设置颜色什么的

## 曲线与网格转换
- LoopTools 把四边形变成一个圆
- Shift + A  -> Curve -> Bezler <br>
	E 可以产生新的  <br>
	context(物体数据)-> Shape -> FillMode  可以调整样式  <br>
	context(物体数据)-> Geometry -> Depth 可以调整粗细  <br>
	context(物体数据)-> Shape -> Resolution Preview U 片段数   片段数少的话就比较好编辑  <br>
	Object菜单 -> Convert to -> Mesh From CurveMetaSurce/Text  <br>

## Alt + D
- 添加新的点，线，面