---
layout:     post
title:      "svn 清理失败 cleanup 失败的解决方案"
subtitle:   ""
date:       2022-11-4 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Other
---


## 参考
- [这里](https://blog.51cto.com/u_14142841/5592886) 
- [工具下载](https://www.sqlite.org/download.html)

## 步骤
-  step1: 到 sqlite官网 [工具下载](http://www.sqlite.org/download.html) 下载 sqlite3.exe,  找到 Precompiled Binaries for Windows，点击 sqlite-shell-win32-x86-3080500.zip  
<br>
- step2: 将下载到的 sqlite3.exe 文件复制到 本地磁盘的某个临时目录下  （我的svn源代码放在共享磁盘中，发现 sqlite老是找不到 svn的 wc.db文件）  
<br>
-  step3:  然后 设置 svn源代码 文件夹 及文件 显示 所有文件（包括隐藏文件），会发现 .svn/wc.db 文件， 将 其复制到 step2的临时目录下  
<br>
-  step4:  开始 -> 运行 -> 打开 cmd命令 -> 
> sqlite3 wc.db  
 <br>
> select * from work_queue;  
 <br>
> delete from work_quere;
 <br>
<br>

- step 5: 将 wc.db 覆盖到 svn源代码目录的 .svn目录下  
<br>
-  step 6: 对 svn源代码目录 右键, clean up, 稍等1至5分钟左右，然后会提示 清理成功。

