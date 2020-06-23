---
layout:     post
title:      "blender eevee Cascaded Shadow Maps"
subtitle:   ""
date:       2020-06-12 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 编译

- 主要看这两个commit

> 2017/4/20  Eevee: Start Implementation of Cascaded Shadow Maps

> 2017/4/21 * Eevee: Cascaded Shadow Maps, follow up.  
Compute coarse bounding box of split frustum. Can be improved  
Make use of 4 cascade.  
View dependant glitches are fixed.  
Optimized shader code.  


- 定位到 2017/4/21 * Eevee: Cascaded Shadow Maps, follow up. 



- 修改 blender\build_files\cmake\platform 下的 platform_win32_msvc.cmake 

> 39行添加下面一段  
macro(find_package_wrapper)  
	if(WITH_WINDOWS_FIND_MODULES)  
		find_package(${ARGV})  
	endif()  
endmacro()  
<br> 
441 行添加下面一行  
set(ALEMBIC_FOUND 1)  
例如 :  
if(WITH_ALEMBIC)  
	set(ALEMBIC ${LIBDIR}/alembic)  
	set(ALEMBIC_INCLUDE_DIR ${ALEMBIC}/include)  
	set(ALEMBIC_INCLUDE_DIRS ${ALEMBIC_INCLUDE_DIR})  
	set(ALEMBIC_LIBPATH ${ALEMBIC}/lib)  
	set(ALEMBIC_LIBRARIES optimized alembic debug alembic_d)  
	set(ALEMBIC_FOUND 1)  -- 新加  
endif(  
  


- blender需要重新编译和生成, 命令 : *make.bat full nobuild 2015*

- 执行INSTALL

- 执行blender


## 效果展示



## 理论




## 实践

### 渲染过程

