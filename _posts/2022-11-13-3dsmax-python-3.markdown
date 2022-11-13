---
layout:     post
title:      "3dsmax python 界面添加菜单 + 安装插件"
subtitle:   ""
date:       2022-11-13 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 3dsmax
---
## 准备软件
- 3dsmax 2018

## 添加菜单的代码案例

- 可以参考[这里](https://help.autodesk.com/view/3DSMAX/2018/ENU/?guid=__files_GUID_F039181A_C072_4469_A329_AE60FF7535E7_htm)  
<br>
- 可以在3dsmax 里面 Help -> MaxScript Help, 一些maxscript 的 api 可以在这里找到含义
<br>
<br>
- 案例，在3dsmax 的主菜单栏上面添加一个  test  ->  Button Text Test 按钮，这一个按钮用来启动 某一个 插件 目录下的 ms 脚本

```
macroScript ActionTest
	buttonText:"Button Text Test"
	category:"TEST"
	tooltip:"MAX_Plugin"
(
	fileIn "test_3dsmax_python\ms\main.ms"
)
main_menu = menuMan.getMainMenuBar()
test_menu = undefined

for i=1 to main_menu.numItems() do (
    menu = main_menu.getItem i
    if menu.getTitle() == "TEST" do (
		test_menu = menu
		break
    )
)

if test_menu == undefined then (

	test_menu = menuMan.createMenu "TEST"

	action_item_test = menuMan.createActionItem "ActionTest" "TEST"

	test_menu.addItem action_item_test -1
		
	test_submenu = menuMan.createSubMenuItem "TEST" test_menu

	main_menu.addItem test_submenu -1

	menuMan.updateMenuBar()

) else (
	button_sub_menu_idx = -1
	test_submenu = test_menu.getSubMenu()
	for i=1 to test_submenu.numItems() do (
		sub_menu = test_submenu.getItem i
		if sub_menu.getTitle() == "Button Text Test" do (
			button_sub_menu_idx = i
			break
		)
	)

	if button_sub_menu_idx != -1 do (
		test_submenu.removeItemByPosition button_sub_menu_idx
	)

	action_item_test = menuMan.createActionItem "ActionTest" "TEST"
	test_submenu.addItem action_item_test -1
)
menuMan.updateMenuBar()

```

## 手动添加3dsmax 的 test_3dsmax_python 插件
- 定位到 C:\Users\xxxx\AppData\Local\Autodesk\3dsMax\2018 - 64bit\ENU\scripts
<br>
<br>
- 上面的目录下有一个 startup 目录，把 菜单 ms 放在这个 startup 目录 下面
<br>
<br>
- 在 scripts 目录的同一级 新建一个 test_3dsmax_python 目录, 插件的所有的ms 和 py 都放在这个目录下面，例如上面的插件引用的 fileIn "test_3dsmax_python\ms\main.ms"， 就可以直接放在 test_3dsmax_python 目录 下面的 ms 目录 
<br>
<br>
- 例如 main.ms
<br>
<br>

```
entry_file = getdir #userScripts + "\\test_3dsmax_python\\hello_world.py"
if doesFileExist entry_file then(
	version = maxVersion()
	if (version[1] >= 18000) then
	(
	    try (
    			python.ExecuteFile entry_file
    		)
    		catch(
    			error = entry_file+"打不开，请检查路径"
    			messageBox error title:"错误提示"
    		)
	)
	else
	(
	    messageBox "插件还未支持此Max版本" title:"错误提示"
	)
)
```

- 点击菜单就要调用 "test_3dsmax_python\ms\main.ms"， main.ms 就调用 "\\test_3dsmax_python\\hello_world.py"
<br>
<br>

- hello_world.py 如下
<br>
<br>

```


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