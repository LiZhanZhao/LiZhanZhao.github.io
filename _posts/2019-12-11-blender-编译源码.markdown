---
layout:     post
title:      "blender-编译源码"
subtitle:   ""
date:       2019-12-11 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 预备知识
需要预备Git,SVN相关知识。

  
## 源码在哪里？
blender的源码在 [这里](https://github.com/sobotka/blender) 可以进行 Git Clone，下面在Window下进行编译源码。


## 最新版本blender的编译
- 安装vs2017  
<br>
- 在blender目录下，运行 make.bat ，漫长的等待。  
<br>
- 在blender目录的同级会出现 build_windows__x64_vc15_Release 目录，然后可以进入这个目录，打开  Blender.sln。  
<br>
- 找到CMakePredefinedTargets下的INSTALL执行生成。  
<br>
- 再选择blender，执行开始执行。

## 旧版本blender的编译
什么是旧版本的blender，当git切换到以前任意一个commit的时候，这个时候就要考虑到怎么编译blender，假如你切换到2017年的某一个commit的时候，blender要重新编译，但是环境怎么配置呢，下面就来举例子说明。

切换到blender到 ***2017/3/18 Fix Shader compilatioin*** 的commit 上

- 安装vs2015  
<br>
- 在blender的同级目录 svn checkout (https://svn.blender.org/svnroot/bf-blender/trunk/lib/) ,但是只需要 win64_vc_14 目录。这个时候会在blender的同级目录生成一个lib目录，lib目录下只有win64_vc_14目录。  
<br>
- 把lib目录revert到 ***618825 2017/3/8*** 的commit上，因为上面blender目录切换到 ***2017/3/18 Fix Shader compilatioin*** 所以对应的lib目录也要切换到旧的commit上，不然编译不起来。  
<br>
- 进入blender目录执行 ***make.bat full nobuild 2015***  ,执行完之后，在blender的同级目录会生成一个 ***build_windows_Full_x64_vc14_Release*** 目录，这个目录有 Blender.sln 文件。  
<br>
- 用vs2015 打开 ***Blender.sln*** , 先执行 找到***INSTALL工程***，点击生成，生成完之后，再找到***blender工程***，点击运行，这个时候blender就顺利出来了。

## blender编译需要用到地址
[blender git 地址](https://github.com/sobotka/blender)  

[blender svn 地址](https://svn.blender.org/svnroot/bf-blender/trunk)


