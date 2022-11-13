---
layout:     post
title:      "3dsmax python 界面添加菜单"
subtitle:   ""
date:       2022-11-13 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 3dsmax
---

## 案例

```
macroScript ActionTest
	buttonText:"Button Text Test"
	category:"TEST"
	tooltip:"MAX_Plugin"
(
	fileIn "xxx\ms\main.ms"
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