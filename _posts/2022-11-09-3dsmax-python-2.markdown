---
layout:     post
title:      "3dsmax python 界面开发"
subtitle:   ""
date:       2022-11-06 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 3dsmax
---

## 准备
- pyside 的安装 : pip install pyside2

## 在 pycharm 上添加 External Tools
- 添加快速打开 Qt Designer External Tools: 
> 菜单 File -> Settings -> Tools -> External Tools -> 直接添加 Program
> pyside designer 存放的位置
C:\Python\Python310\Lib\site-packages\PySide2

- 添加 将ui文件转为py文件的 External Tools
> 菜单 File -> Settings -> Tools -> External Tools -> 直接添加 Program  
<br>
> pyside2-uic, 主要是看电脑，可能在C:\Pythonxx\Scripts\pyside-uic.exe, 也有可能在 C:\Users\xxxxx\AppData\Local\Packages\PythonSoftwareFoundation.Python.3.10_qbz5n2kfra8p0\LocalCache\local-packages\Python310\Scripts  
<br>
> Arguments 添加 $FileName$ -o $FileNameWithoutExtension$.py  
<br>
> Working directory 添加 $FileDir$


## 案例 
- hello_world.py 有一个同级目录 ui_res, ui_res 里面 有一个 "____init____.py", "test.ui"  
<br>
- "____init____.py" 内容 , 添加这个 脚本主要是为了可以后续 在hello_world.py import 把这个目录认为是一个package

```
# -*- coding:utf-8 -*-
```
<br>

- test.ui 会被上面的 pyside2-uic 从ui文件 转为 py 文件，得到 test.py 

```
# -*- coding: utf-8 -*-

from PySide2.QtCore import *
from PySide2.QtGui import *
from PySide2.QtWidgets import *


class Ui_MainWindow(object):
    def setupUi(self, MainWindow):
        if not MainWindow.objectName():
            MainWindow.setObjectName(u"MainWindow")
        MainWindow.resize(800, 600)
        self.centralwidget = QWidget(MainWindow)
        self.centralwidget.setObjectName(u"centralwidget")
        self.pushButton = QPushButton(self.centralwidget)
        self.pushButton.setObjectName(u"pushButton")
        self.pushButton.setGeometry(QRect(360, 370, 75, 23))
        font = QFont()
        font.setFamily(u"Arial")
        self.pushButton.setFont(font)
        MainWindow.setCentralWidget(self.centralwidget)
        self.menubar = QMenuBar(MainWindow)
        self.menubar.setObjectName(u"menubar")
        self.menubar.setGeometry(QRect(0, 0, 800, 23))
        MainWindow.setMenuBar(self.menubar)
        self.statusbar = QStatusBar(MainWindow)
        self.statusbar.setObjectName(u"statusbar")
        MainWindow.setStatusBar(self.statusbar)

        self.retranslateUi(MainWindow)

        QMetaObject.connectSlotsByName(MainWindow)
    # setupUi

    def retranslateUi(self, MainWindow):
        MainWindow.setWindowTitle(QCoreApplication.translate("MainWindow", u"MainWindow", None))
        self.pushButton.setText(QCoreApplication.translate("MainWindow", u"PushButton", None))
    # retranslateUi

```
<br>

- hello_world.py

```python

import MaxPlus
from PySide2.QtWidgets import QMessageBox,QMainWindow

import os
import sys
import types


cwd = os.path.abspath(os.path.dirname(__file__))

sys.path.append(cwd)

import ui_res
reload(ui_res)

from ui_res.test import Ui_MainWindow


class ToolWindow(QMainWindow , Ui_MainWindow):
    def __init__(self, parent=MaxPlus.GetQMaxMainWindow()):
        super(ToolWindow, self).__init__(parent)

        setattr(Ui_MainWindow,
                "retranslateUi",
                types.MethodType(getattr(self, "retranslateUi").im_func, Ui_MainWindow)
                )

        self.setupUi(self)

        self.pushButton.clicked.connect(self.onClickAddRules)

    def retranslateUi(self, MainWindow):
        self.pushButton.setText("PushButton_xxxxx")

    def onClickAddRules(self):
        """
        点击添加规则按钮
        :return:
        """
        msg_box = QMessageBox(MaxPlus.GetQMaxMainWindow())
        msg_box.setText("Hello World!")
        msg_box.show()


def main():
    MainWindow = ToolWindow()
    MainWindow.show()


if __name__ == '__main__':
    main()
```
>  setattr(Ui_MainWindow, "retranslateUi", vtypes.MethodType(getattr(self, "retranslateUi").im_func, Ui_MainWindow))   
的作用就是为了在 用 hello_world.py 的 retranslateUi 函数 去覆盖 test.py 的 retranslateUi 函数
