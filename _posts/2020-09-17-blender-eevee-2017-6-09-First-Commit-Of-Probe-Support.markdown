---
layout:     post
title:      "blender eevee First commit of Probe support"
subtitle:   ""
date:       2021-1-14 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/9  * Eevee: First commit of Probe support. <br>  

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		

		

## 效果
![](/img/Eevee/FirstCommitOfProbe/res.png)
![](/img/Eevee/FirstCommitOfProbe/res-1.png)
![](/img/Eevee/FirstCommitOfProbe/res-2.png)
![](/img/Eevee/FirstCommitOfProbe/res-3.png)

## 作用 
添加 Probe 功能，这个功能其实就是动态环境图效果，就是在Probe的位置，拍摄周围附近的场景物体作为环境图，传入给物体的Shader中
![](/img/Eevee/FirstCommitOfProbe/1.png)


## 编译
- 重新生成SLN
- git 定位到 2017/6/9 17:40:47   Merge branch 'master' into blender2.8  (eevee-25-2)    
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)




