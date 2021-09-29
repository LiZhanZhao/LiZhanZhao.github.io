---
layout:     post
title:      "Blender Python Basic"
subtitle:   ""
date:       2021-10-01 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender Python
---

## 编写插件基础

参考 [基础](https://zhuanlan.zhihu.com/p/138455778)

[官方文档](https://docs.blender.org/api/current/)

### 基础插件模板

```python

import bpy 

class TestPanel(bpy.types.Panel):
    bl_label="Title"
    bl_idname="PT_Test Panel"
    bl_space_type="VIEW_3D"  #它的作用就是定义这个插件是作用于哪个工作区域Workspace内
    bl_region_type="UI"     #设置使用区域
    bl_category="Hello World" #设置归属类别
    
    def draw(self,context):
        layout=self.layout # 简写，引用变量获取这个Panel面板类的布局 ，当然你也可以写作  l=self.layout(如果想简写地更加精简的话） 
        row=layout.row() #创建一个Horizontal Box ，即新一行
        row.label(text='Add a girl',icon='MESH_CUBE') # Label 标签定义行内的元素，有文字，有图标，双引号内的为各自的显示结果。
        row=layout.row() #另创建一个新Horizontal Box，即新一行
        row.label(text='Add a boy',icon='FUND')
        
        #bpy.ops.mesh.primitive_cube_add(enter_editmode=False, location=(0, 0, 0))
        row=layout.row()
        row.operator("mesh.primitive_cube_add",icon="CUBE", text = "111")
        
        
def register():
    bpy.utils.register_class(TestPanel)

def unregister():
    bpy.utils.unregister_class(TestPanel)
    
if __name__=="__main__":
    register()
```


<br>
<br>
效果图:
![](/img/BlenderPython/1.png)

<br>
<br>

### 多个class 使用的例子

参考 [这里](https://docs.blender.org/api/current/bpy.props.html?highlight=bpy%20props#module-bpy.props)

```
import bpy


class OBJECT_OT_property_example(bpy.types.Operator):
    bl_idname = "object.property_example"
    bl_label = "Property Example"
    bl_options = {'REGISTER', 'UNDO'}

    my_float: bpy.props.FloatProperty(name="Some Floating Point")
    my_bool: bpy.props.BoolProperty(name="Toggle Option")
    my_string: bpy.props.StringProperty(name="String Value")

    def execute(self, context):
        self.report(
            {'INFO'}, 'F: %.2f  B: %s  S: %r' %
            (self.my_float, self.my_bool, self.my_string)
        )
        print('My float:', self.my_float)
        print('My bool:', self.my_bool)
        print('My string:', self.my_string)
        return {'FINISHED'}


class OBJECT_PT_property_example(bpy.types.Panel):
    bl_idname = "object_PT_property_example"
    bl_label = "Property Example"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Tool"

    def draw(self, context):
        # You can set the property values that should be used when the user
        # presses the button in the UI.
        props = self.layout.operator('object.property_example')
        props.my_bool = True
        props.my_string = "Shouldn't that be 47?"

        # You can set properties dynamically:
        if context.object:
            props.my_float = context.object.location.x
        else:
            props.my_float = 327


bpy.utils.register_class(OBJECT_OT_property_example)
bpy.utils.register_class(OBJECT_PT_property_example)
```