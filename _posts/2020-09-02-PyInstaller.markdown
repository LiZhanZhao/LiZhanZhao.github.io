---
layout:     post
title:      "Python + Pyinstaller"
subtitle:   ""
date:       2020-09-02 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Other
---

## 目的
Python是一个很好用的高效开发工具，但其程序执行时需要有解释环境才能运行，独立运行时非常不便，在Python强大的支持库中提供了一款很方便的工具“Pyinstaller”，可以将Python程序打包成可独立执行的EXE文件，降低脚本对环境的依赖性，同时运行更加迅速。

## 环境
Python 3.7

## 安装Pyinstaller
执行下面命令
***pip install pyinstaller***


## 使用Pyinstaller
- cmd 到/python/scripts 找到pyinstaller.exe
- pyinstall.exe -F xxx.py, 会生成build目录和dist目录，dist可以找到对应的exe文件



