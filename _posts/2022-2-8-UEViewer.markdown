---
layout:     post
title:      "UEViewer 应用"
subtitle:   ""
date:       2022-2-8 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Other
---



## 准备的软件

- 工具 UE Viewer  ：[下载](https://github.com/gildor2/UEViewer)



## 工具破解游戏步骤



- 下载好 UE Viewer 工具 和 游戏 之后，用 UE Viewer 打开 游戏的安装目录，如下：
- ![1](/img/UEViewer/1.png)



- UE Viewer 工具破解 Granblue Fantasy: Versus，需要选择游戏的Unreal版本，如下：

  ![2](/img/UEViewer/2.png)

  

- 除了游戏的Unreal版本之后，还需要用到ASE Key，目前 《 Granblue Fantasy: Versus》的ASE Key 为 **0x2A472D4B6150645367566B597033733676397924423F4528482B4D6251655468**，如下：

  ![3](/img/UEViewer/3.png)



- 经过上面的操作，成功破解游戏，就可以直接如下的界面 ：

  ![4](/img/UEViewer/4.png)



## 工具导出游戏贴图

- 游戏的大部分角色资源保存在目录  **RED/Content/Chara** 中，目录名字和角色名字一一对应， [这里](http://granbluefantasyvs.com/characters/) 可以找到角色的名字：

![5](/img/UEViewer/5.png)



- 找到对应的角色目录，定位到子目录 **Material** 中，在这个目录可以获得角色所有的贴图：

![6](/img/UEViewer/6.png)



- 我们随便双击一个图片资源，UE Viewer 就可以直接显示图片，方便知道每一个资源对应什么图片，如下：

  ![7](/img/UEViewer/7.png)





- 在导出图片之前，先要设置导出的图片格式和导出目录，步骤如下：

  ![8](/img/UEViewer/8.png)

​		![9](/img/UEViewer/9.png)



- 设置完图片的格式之后，就可以进行导出，导出目前的图片的操作如下：

  ![10](/img/UEViewer/10.png)



- 如果导出整一个目录的纹理，先打开文件浏览界面：

  ![11](/img/UEViewer/11.png)

  

  再右键文件夹：

  ![12](/img/UEViewer/12.png)



​	执行完以上的菜单，就在 **导出目录** 找到对应的资源。



## 工具导出游戏人物+动作资源为 gltf

- 先定位 角色的子目录 **Mesh** 下，这个目录保存的是角色的模型资源。

  ![13](/img/UEViewer/13.png)



- 双击资源可以直接看到模型，如下：

  ![14](/img/UEViewer/14.png)



- 添加动作，定位到目录 角色的 **Adv** 中，

  ![15](/img/UEViewer/15.png)



- **追加** 动作资源到模型中：

  ![](/img/UEViewer/16.png)



- 这个播放可以利用 按键 【  和  】 来选择动作，空格Space 进行播放动作，如下：

  ![gif1.0](/img/UEViewer/gif1.0.gif)



- 在导出之前，也要先设置模型的导出格式，这里设置为 gltf ，然后导出就是直接在导出目录看到导出的 gltf 文件：

  ![17](/img/UEViewer/17.png)



## gltf 资源导出为FBX

- 直接用 导出 gltf 文件 到 Blender 中

  ![18](/img/UEViewer/18.png)



- 导入gltf 之后，就可以在Blender直接播放资源动作：

  ![gif2.0](/img/UEViewer/gif2.0.gif)



- 把资源导出为FBX，如下：

  ![19](/img/UEViewer/19.png)
