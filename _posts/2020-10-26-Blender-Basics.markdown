---
layout:     post
title:      "Blender 入门总结"
subtitle:   ""
date:       2020-10-26 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender 应用
---

## 理解大模块(区域)
- Blender 有很多大模块，例如 3D Viewport, UV Editor, Shader Editor 
- 每一个大模块都是不同的窗口，窗口都会有自己的 Header, Tool Setting, 工具栏，属性栏, 菜单栏
- 大模块也会有自己的小模块，例如 3D Viewport 模块里面，有 Object Mode , Edit Mode 等


## 双显示器配置
- Window菜单 -> New Window 或者（New Main Window）就可以多一个窗口出来给另外一个显示器了

## 不同版本Blender互相打开文件
- 不同版本Blender互相打开文件的时候，要注意要不要勾选 load UI 选项


## Preferences 窗口
- Preferences窗口在 Editor->Preferences 下

#### 设置中文
- 打开 Editor 菜单下 Preferences  ，InterFace 页面 -> Translation 
这样就打开其他文件的时候，文件名字带中文都可以显示出来

#### 设置左右键选择物体
- 打开 Editor 菜单下 Preferences，KeyMap 页面 -> Select With 

#### 设置按下空格健的意义
- 打开 Editor 菜单下 Preferences，KeyMap 页面 -> Spacebar Action
- 对应的选项是 : 播放，悬浮出现工具栏，搜索


## 模块窗口操作
#### 显示工具栏 和 属性栏 
- T 键，显示 最左侧的工具栏， N 键 显示 最右侧的属性栏

#### Show Tool Setting 
- 显示最右边工具栏每一个工具的详细参数，需要开启 Show Tool setting 
右键 Header 那一栏  -> Header -> Show Tool Setting 

#### 最大化窗口 
- 想要最大化哪一个窗口，就在那个窗口中 view -> Area -> Toogle Maximize Aear （快捷键是 Ctrl + Space）
	全屏的话，也 view -> Area -> Toogle FullScreen Aear （快捷键是 Ctrl + alt + Space）

#### 添加快速菜单
- 在任意的菜单中右键 add to the quick favorites，就可以把菜单添加到 quick favorites 菜单列表中，然后在窗口中按 Q 键，就可以出现对应的菜单列表，如果想删除的话，也可以在对应的菜单右键，remove to the quick favorites

#### 底部信息栏
- 提示鼠标左中右键盘 的作用 + 物体信息(面数顶点数内存)

#### 大模块布局，切屏
- 鼠标放在窗口的最右上交变成十字，就可以进行切屏

#### 工具栏显示大小
- Ctrl + 鼠标中键 可以进行缩放

#### 全局的UI缩放
- 打开 Editor 菜单下 Preferences -> Interface -> Resolution Scale

#### 恢复界面的设置
- 菜单 File -> Defaults -> Load Factory Setting  可以恢复界面的设置



## Blender 文件结构
- OutLine 模块里面的 Blender File

#### Link
- Link 其他文件的物体，无法在本文件更改

#### Append
- Append 其他文件的物体， 在本文件是可以更改



## 视图基础操作

#### 摇移视图
- 按住鼠标中键移动 摇移视图

#### 缩放视图
- 滚动鼠标中键缩放视图

#### 1 3 7 9 5 0 
- 小键盘 1 前视图 快捷键 Ctrl + 1
- 小键盘 3 右视图 快捷键Ctrl + 3
- 小键盘 7 顶视图
- 小键盘 9 底视图
- 小键盘 5 正交透视视图切换
- 小键盘 0 摄影机视图
- 小键盘 . 最大化选择的物体匹配到视图
- 小键盘 2 4 6 8 翻转视图

#### 平移视角
- Shift + 鼠标中键移动 平移视角

#### 视图饼饼菜单
- 按键 ~ 打开饼菜单

#### 四视图
- View-> Area -> Toggle Quad View  (快捷键  Ctrl + Alt + Q )  



## 选择
#### 全选
- 按下 A 全选  AA （Alt + A） 取消选择 

#### 框选
- 快捷键 B 
- 选择工具模式下， +Shift 增加选择， +Ctrl 减去选择
- 快捷键模式下 +Shift 减去选择

#### 绘制选择
- 按下 C 绘制选择，
- 选择工具模式下， 绘制选择(鼠标左键) 增加选择   + Shift 增加选择    + Ctrl 减去选择
- 快捷键模式下  (按C), 绘制选择(鼠标左键) 增加选择     + Shift 减去选择
- 快捷键模式下  (按C)，滚动鼠标中键增大缩小选区

#### 反选
- Ctrl + I 反选

#### 高级选择工具
- Select 菜单下的，(select all by type) 的另外一些选择命令
- Shift + G 根据物体的相同属性组进行选择


## 显示与隐藏 

#### 菜单
- Object -> Show/Hide

#### 快捷键
- H 隐藏选中物体
- Alt + H  显示所有物体
- Shift + H 隐藏未选择的所有物体


## 撤销

#### 菜单
- Edit-> Undo Redo 菜单

#### 快捷键
- Ctrl + Z 撤销 Undo
- Shift + Ctrl + Z  返回操作，重做  redo
- Alt + Ctrl + Z 打开历史记录  Undo History
- Shift + R  重复最后一步命令   Repeat Last


## 父子连接关系设置
- 设置父子关系的时候，先选择的是子物件，后选择的是父物体

#### 父子关系设置
- Ctrl + P 父子关系设置面板

#### 清楚父子连接关系
- Alt + P 清楚父子连接关系

## Collection
#### 快捷键
- M键 将选择的物体进行 Collection 设置
- Ctrl + H 只显示某一个集合
- Alt + H  显示所有几何，但是要点击Scene Collection
- 数字1，2，3，4... 是单独显示 某一个 Collection
- Shift + 1 2 3 4 ... 是 同时显示 多个 collection


## 视窗overlays
- overlays 可以在渲染的时候，再编辑模型
- 在 Edit -> Preferences -> KeyMap ,打开 Extra Shading Pie Menu Items） ,点击 Z 健的时候，可以看到 Overlays
- 可以在OverLays 设置WireFrame 的强度

## LookDev 开发预览显示模式、Rendered视图实时渲染模式
- LookDev 预览模式是用Eevee的
- LookDev 在viewport shading 面板中，开启 Scene Lights 才开启场景灯光，
		如果没有开启 Scene Lights 的选项，环境球就会计算 球谐 SH
		如果开启了Scene Lights 的选项，环境球只是计算IBL高光
		如果开启了scene world的话，就用world下面的环境球来计算高光
- Shift + Z 线框及最后一个激活的显示模式进行切换

## 认识原点， 认识轴心点
- Shift + S  浮动面板 可以设置游标到世界原点
- w 键可以显示 Object Context Menu , 设置物体的原点
- 可以在Pivot Point 中设置轴心点，就是设置作用点，也可以理解为原点
- 轴心主要在 Pivot Point面板 进行操作
- 键盘 .  弹出设置 轴心点 饼图



## Auto Save自动保存

- Edit -> Preferences -> Save & Load   设置自动保存

- 在File -> Recover 中进行恢复

- 设置撤销的步数 Edit -> Preferences -> System -> Memory & Limits -> Undo Steps 

## 无小键盘的笔记本及苹果鼠标模拟设置
- Edit -> Preferences -> input -> Keyboard -> Emulate Numpad
- Edit -> Preferences -> input -> Mouse -> Emulate 3 Button Mouse


## 常用操作

#### 添加和删除
- SHIFT+A 添加 
-  X 删除

#### 移动 旋转 缩放
-  G R S 键

#### 清除位移、旋转、缩放
-  ALT+G R S
- 菜单 Object -> clear -> Location Rotation Scale

#### 弹出，悬浮 工具栏
- Shift + Space 

#### 复制与关联复制
- Shift + D 复制  (Object -> Duplicate Objects)
- Alt + D 关联复制



#### 设置为激活的摄影机
- Ctrl + 0 将选择的物体设置为激活的摄影机

#### 关联设置
- Ctrl + L  Make Link 设置关联
- 设置关联选择的先后顺序：先选的物体 = 我要关联到谁？ 后选的物体 = 他们要关联到我？

#### 显示选中物体菜单操作(右键)
- 右键没有反应就按下w键盘

#### 游戏的方式查看 Blender 场景文件
-  快捷键 Shift + ~
- view -> Navigation -> Fly Navigation 
- view -> Navigation -> Walk Navigation
-  W S A D + Shift 极速
- G 激活中立，
- 滚动鼠标中键 ， 增加或者减少速度

#### 导入导出选项与插件
- Edit -> Preferences -> Add-ons 打开关闭导入导出的插件

#### 设置近远平面值
- Tool 中的 view 可以设置近远平面值  clip start end
- 设置近远平面，Camera 在属性栏 Camera属性面板得 Lens


#### 单位
- 单位统一是米

#### Compositor 合成
- 设置好 View Layer 可以用来做单个物体渲染输出，一个物体一张图片
- 利用Compositor 可以把多个ViewLayer输出的图进行合成

## 骨骼
#### Single Bone
- 按：Tab键，切换到 物件模式（Object Mode） 
- 按：Shift + A，选择：Armatrue（骨架） -> Single Bone （单个骨头）
- 选择一个骨骼，-> context data -> Viewport Display -> InFront 来把骨骼显示在最前面
- 在 Editor Mode 里面可以 按 E 进行 子骨骼
- 在 Pose Mode 里面是进行 K 帧 
- 骨架跟模型产生关系, 在object mode 下，先选择 模型，再选择 骨架，然后Ctrl + P, 直接点击 Automatic Weights



## Shader

### View Node
添加 the node wrangler addon. Once the node wrangler addon is enabled, the addon creates such a node when pressing Ctrl+Shift + RMB on the node.

### Detach Links 
Alt-D