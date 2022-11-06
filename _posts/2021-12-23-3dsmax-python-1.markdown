---
layout:     post
title:      "3dsmax python 配置开发环境"
subtitle:   ""
date:       2022-11-06 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 3dsmax
---

## 软件准备
- PyCharm

## 配置步骤
- 新建一个PyCharm 工程  
<br>
- 在 [MXSPyCOM](https://github.com/techartorg/MXSPyCOM/releases) 下载 MXSPyCOM  
<br>
- 把 下载好的 MXSPyCom 的内容 直接拷贝到工程中，例如 tool/MXSPyCom  
<br>
- 假如是 3dsmax 2018 的话，那么进入到 3dsmax 的安装目录，找到 3dsmaxpy.exe， 然后 软链接 一个 python.exe 出来
mklink python.exe 3dsmaxpy.exe， 这里如果觉得麻烦的话，就直接复制改名字也可以  
<br>
- Pycharm 中 File -> Setting -> Python Interpreter  -> 把刚才的 3dsmax 里面的 python.exe 添加进来， 
这里需要注意，Add 之后要 选择 Existing Environment 来进行添加  
<br>
- 新建自己的main.py

```python
import os
import subprocess

if __name__ == '__main__':
    proj_path = os.path.dirname(__file__)
    main_script = os.path.join(os.path.dirname(__file__), "tools/mxspycom/hello_world.py")
    ret = subprocess.call(["./tools/mxspycom/mxspycom_new.exe", "-s", main_script])
```
  
<br>
- 上面 main.py 中的 hello_world.py 如下

```python
import MaxPlus
from PySide2 import QtGui
from PySide2.QtWidgets import QApplication, QMessageBox


class _GCProtector(object):
    widgets = []

app = QApplication.instance()
if not app:
    app = QApplication([])

def main():
    msg_box = QMessageBox(MaxPlus.GetQMaxMainWindow())
    _GCProtector.widgets.append(msg_box)

    msg_box.setText("Hello World!")
    msg_box.show()



if __name__ == '__main__':
    main()
```

<br>
- 把 initialize_COM_server.ms 拷贝到 3dsmax 的 目录的 scripts / Startup 下面  
<br>

- 这个时候可以点击运行，如果发现有库找不到的话，可以点击对应的代码地方，根据提示来点击自动导入库就可以解决


## PyCharm 快捷键设置
- External Tools.xml

```xml
<toolSet name="External Tools">
  <tool name="Run in 3DsMax" description="Run current script in 3dsmax" showInMainMenu="false" showInEditor="false" showInProject="false" showInSearchPopup="false" disabled="false" useConsole="true" showConsoleOnStdOut="true" showConsoleOnStdErr="true" synchronizeAfterRun="true">
    <exec>
      <option name="COMMAND" value="$ProjectFileDir$\tools\mxspycom\mxspycom.exe" />
      <option name="PARAMETERS" value="-s $FilePath$" />
      <option name="WORKING_DIRECTORY" value="$ProjectFileDir$" />
    </exec>
  </tool>
</toolSet>
```
<br>

- 把 这个文件放在 C:\Users\xxxx\AppData\Roaming\JetBrains\PyCharm2020.3\tools 中  
<br>

- 可以设置快捷键 Settings - keymap