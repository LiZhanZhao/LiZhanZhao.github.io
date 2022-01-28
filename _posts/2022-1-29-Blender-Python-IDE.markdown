---
layout:     post
title:      "Blender Python IDE"
subtitle:   ""
date:       2022-1-28 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender Python
---



## 准备的软件

- PyCharm ：[下载](https://www.jetbrains.com/pycharm/)
- blender-autocomplete : [下载](https://github.com/Korchy/blender_autocomplete)



## Blender插件的开发形式

- 首先在Blender的 **自定义插件** 目录添加 自己的目录，例如：

  Blender 2.93 的自定义插件目录：**C:\Users\xxxxx\AppData\Roaming\Blender Foundation\Blender\2.93\scripts\addons**



- 在addons目录下新建自己的插件目录，然后添加 **\_\_init\_\_.py** 文件 

- ```
  bl_info = {
      "name": "My Add-on Sandbox",
      "author": "Chris Conlan",
      "version": (1, 0, 0),
      "blender": (2, 93, 0),
      "location": "View3D",
      "description": "In-filesystem Add-on Development Sandbox",
      "category": "Development",
  }
  import bpy;
  from bpy.types import UIList, Panel, Operator
  
  class TestPanel(Panel):
      bl_label = "Tools"
      bl_space_type = 'VIEW_3D'
      bl_region_type = 'UI'
      bl_category = 'Hello World'
  
      def draw(self, context):
          layout = self.layout
          scn = context.scene
          data = bpy.data
  
  
  def register():
      bpy.utils.register_class(TestPanel)
      return
  
      
   
  def unregister():
      bpy.utils.unregister_class(TestPanel)
      return
  
  if __name__ == "__main__":
      register()
  ```



- 然后在Blender的菜单 **Edit->Preferences->Add-ons** 找到对应的插件
- 当修改插件的时候，就可以点击 **Add-ons** 的 **Refresh** 按钮进行更新插件



## PyCharm 设置

- 直接在 **C:\Users\xxxxx\AppData\Roaming\Blender Foundation\Blender\2.93\scripts\addons** 新建工程

- 添加自动提示，先结压 **blender-autocomplete **，里面有对应的Blender版本的文件夹

- 打开 PyCharm 的 File -> Settings -> Porject -> Project Structure -> Add Content Root

  就可以添加对应的 blender-autocomplete/ 2.93 目录，这样在PyCharm里写Blender Python就可以有自动提示了

 

