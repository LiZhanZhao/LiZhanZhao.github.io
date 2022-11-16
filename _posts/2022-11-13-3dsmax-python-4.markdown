---
layout:     post
title:      "3dsmax python NSIS 打包 插件"
subtitle:   ""
date:       2022-11-16 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 3dsmax
---
## 参考
- [NSIS最简单的软件打包示例](https://zhuanlan.zhihu.com/p/161378184)
<br><br>
- [NSIS简单教程](https://blog.csdn.net/sunny_98_98/article/details/123015942)
<br><br>

## 准备软件
- [NSIS 下载](https://nsis.sourceforge.io/Download)

## 例子
- test.nsi
<br>

```nsi
; The name of the installer (安装包的名字)
Name "test_3dsmax_python_name"

; The file to write(输出安装包)
OutFile "test_3dsmax_python.exe"

; Request application privileges for Windows Vista(给安装包添加管理员权限)
RequestExecutionLevel user

; Build Unicode installer
Unicode True

; The default installation directory (默认的安装目录)
; InstallDir $PROGRAMFILES64\HelloLiam
InstallDir "$LOCALAPPDATA\Autodesk\3dsMax\2018 - 64bit"

;Request application privileges for Windows Vista
;RequestExecutionLevel admin
;--------------------------------
; Pages

Page directory
Page instfiles

;--------------------------------
; The stuff to install
Section "English" ;No components page, name is not important
  ; Set output path to the installation directory.(设置输出目录)
  SetOutPath "$INSTDIR\ENU\scripts"
  ; Put file there (把InstallFiles目录下的所有文件都进行输出)
  File /r "InstallFiles\*.*"
    ; 移动文件位置并修改名字
    Rename "$INSTDIR\ENU\scripts\test_3dsmax_python\ms\menu_init.ms" "$INSTDIR\ENU\scripts\startup\test_3dsmax_python_menu_init.ms"

    ; 一定要添加，不然下面的uninstaller section 就有问题
    WriteUninstaller "$INSTDIR\uninstall_test_3dsmax_python.exe"

SectionEnd ; end the section


Section "Chinese" ;No components page, name is not important
  ; Set output path to the installation directory.
  SetOutPath "$INSTDIR\CHS\scripts"
  ; Put file there
  ;File HelloLiam.exe ;add a file.
  File /r "InstallFiles\*.*"

    Rename "$INSTDIR\CHS\scripts\test_3dsmax_python\ms\menu_init.ms" "$INSTDIR\CHS\scripts\startup\test_3dsmax_python_menu_init.ms"
  
  WriteUninstaller "$INSTDIR\uninstall_test_3dsmax_python.exe"

SectionEnd ; end the section



# create a section to define what the uninstaller does.
# the section will always be named "Uninstall"
Section "Uninstall"
 
# Always delete uninstaller first
  Delete $INSTDIR\uninstall_test_3dsmax_python.exe

 
  IfFileExists "$INSTDIR\CHS\scripts\test_3dsmax_python" RMDIR_CHS_PLUGIN
  IfFileExists "$INSTDIR\ENU\scripts\test_3dsmax_python" RMDIR_ENU_PLUGIN

  RMDIR_CHS_PLUGIN:
    RMDir /r "$INSTDIR\CHS\scripts\test_3dsmax_python"
    Delete "$INSTDIR\CHS\scripts\startup\test_3dsmax_python_menu_init.ms"

  RMDIR_ENU_PLUGIN:
    RMDir /r "$INSTDIR\ENU\scripts\test_3dsmax_python"
    Delete "$INSTDIR\ENU\scripts\startup\test_3dsmax_python_menu_init.ms"
 

SectionEnd
```

## 例子使用

- 创建一个目录 test_nsis，然后把上面的 test.nsi 放在 目录下
<br><br>
- 目录 test_nsis 下再创建  InstallFiles 目录，InstallFiles 目录主要是用来放插件，例如 前面文章提及到的 test_3dsmax_python 的插件，也就是  InstallFiles 目录 存放 一整个 test_3dsmax_python 插件目录
<br><br>
- 打开  NSIS 软件 中的 Compile NSI Scripts 窗口，如果直接打开 test.nsi 就可以直接生成 test_3dsmax_python.exe 安装包
<br><br>
- 运行test_3dsmax_python.exe 安装包的话，就会把 test_3dsmax_python 插件直接安装到  "$LOCALAPPDATA\Autodesk\3dsMax\2018 - 64bit" 目录下的  "\CHS\scripts" 下面，还有 "\ENU\scripts" 也有，然后 也会把 这个 文件 "$INSTDIR\CHS\scripts\test_3dsmax_python\ms\menu_init.ms" 移动改名到 "$INSTDIR\CHS\scripts\startup\test_3dsmax_python_menu_init.ms"，主要是为了添加菜单，
<br><br>
- 在 "$LOCALAPPDATA\Autodesk\3dsMax\2018 - 64bit" 目录下的也会有 uninstall.exe  安装包，用来删除上面的所有的文件






