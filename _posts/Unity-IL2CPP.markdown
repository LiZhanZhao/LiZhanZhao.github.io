---
layout:     post
title:      "Unity IL2CPP 游戏逆向"
subtitle:   ""
date:       2020-09-14 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Unity
---
## 参考
>
- [记一次 Unity IL2CPP 游戏逆向](https://dev.moe/1282)
- [反编译so文件(IDA_Pro)](https://www.cnblogs.com/whycxb/p/9143896.html)

## 工具
>
- [Il2CppDumper v4.6.0](https://github.com/Perfare/Il2CppDumper/releases?after=v5.0.0)
- IDA 在 [反编译so文件(IDA_Pro)](https://www.cnblogs.com/whycxb/p/9143896.html) 可以下载

## 注意
>
- 如果apk包解压之后就有 assets/bin/Data/Managed/Assembly-CSharp.dll，那么就可以直接使用 ***.NET Reflector*** 进行反编译。
- 如果没有 assets/bin/Data/Managed/Assembly-CSharp.dll，是用了 IL2CPP 来编译。这样一来，游戏用到的字符串等被放到了 assets/bin/Data/Managed/Metadata/global-metadata.dat，而游戏二进制文件则位于 lib/armeabi-v7a/libil2cpp.so。


## 步骤
>
- 使用Il2CppDumper.exe, 添加 lib/armeabi-v7a/libil2cpp.so 和 assets/bin/Data/Managed/Metadata/global-metadata.dat 可获得 DummyDll、dump.cs 以及 script.py 几个文件
- 用 IDA 打开 libil2cpp.so，Alt+F7 运行刚才生成的 script.py，稍等片刻，定位到对应函数 F5 反编译，就获得了一份有可读性的代码