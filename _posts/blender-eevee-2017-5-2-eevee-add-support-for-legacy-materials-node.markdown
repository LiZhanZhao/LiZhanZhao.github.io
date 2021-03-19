---
layout:     post
title:      "blender eevee  Add support for legacy materials node. (not PBR)"
subtitle:   ""
date:       2020-08-05 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> 2017/5/2   Eevee: Add support for legacy materials node. (not PBR)


## 作用 
*渲染物体可以使用nodetree*


## 编译

- 直接编译


## 效果展示
在 *Node Editor* 中选择 World 分页 和 打勾 *Use Nodes*
![](/img/Eevee/LegacyMaterialsNode/1.png)



## CPU

### 渲染
*eevee_engine.c*
```c
if (ma->use_nodes && ma->nodetree) {
	const DRWContextState *draw_ctx = DRW_context_state_get();
	Scene *scene = draw_ctx->scene;
	struct GPUMaterial *gpumat = GPU_material_from_nodetree(
		scene, ma->nodetree, &ma->gpumaterial, &DRW_engine_viewport_eevee_type, 0,
		datatoc_lit_surface_vert_glsl, NULL, e_data.frag_shader_lib,
		"#define PROBE_CAPTURE\n"
		"#define MAX_LIGHT 128\n"
		"#define MAX_SHADOW_CUBE 42\n"
		"#define MAX_SHADOW_MAP 64\n"
		"#define MAX_SHADOW_CASCADE 8\n"
		"#define MAX_CASCADE_NUM 4\n");

	DRWShadingGroup *shgrp = DRW_shgroup_material_create(gpumat, psl->material_pass);

	if (shgrp) {
		DRW_shgroup_call_add(shgrp, mat_geom[i], ob->obmat);
	}
	else {
		/* Shader failed : pink color */
		static float col[3] = {1.0f, 0.0f, 1.0f};
		static float spec[3] = {1.0f, 0.0f, 1.0f};
		static short hardness = 1;
		shgrp = DRW_shgroup_create(e_data.default_lit, psl->default_pass);
		DRW_shgroup_uniform_vec3(shgrp, "diffuse_col", col, 1);
		DRW_shgroup_uniform_vec3(shgrp, "specular_col", spec, 1);
		DRW_shgroup_uniform_short(shgrp, "hardness", &hardness, 1);
		DRW_shgroup_call_add(shgrp, mat_geom[i], ob->obmat);
	}
}
else {
	DRWShadingGroup *shgrp = DRW_shgroup_create(e_data.default_lit, psl->default_pass);
	DRW_shgroup_uniform_vec3(shgrp, "diffuse_col", &ma->r, 1);
	DRW_shgroup_uniform_vec3(shgrp, "specular_col", &ma->specr, 1);
	DRW_shgroup_uniform_short(shgrp, "hardness", &ma->har, 1);
	DRW_shgroup_call_add(shgrp, mat_geom[i], ob->obmat);
}

```
>
- 这里其实和 [World nodetree gpumaterial compatibility](http://shaderstore.cn/2020/07/31/blender-eevee-2017-4-28-eevee-World-nodetree-gpumaterial-compatibility/) 基本一致, 
- 这里就是判断物体是否使用nodetree进行编辑材质, 如果没有用nodetree的话，就用默认材质继续渲染(default_frag.glsl)
- 如果使用了nodetree的话，就使用自动生成的shader


## 总结渲染物体使用到的shader文件
*vs*
- lit_surface_vert.glsl

*gs*
- NULL


*fs-nodetree*
- bsdf_common_lib.glsl
- ltc_lib.glsl
- bsdf_direct_lib.glsl
- lit_surface_frag.glsl
- *blender/source/blender/gpu/shaders/gpu_shader_material.glsl*
- *gpu_codegen.c 文件中的 code_generate_fragment 函数* 自动生成的代码


*fs-default*
- bsdf_common_lib.glsl
- ltc_lib.glsl
- bsdf_direct_lib.glsl
- lit_surface_frag.glsl
- default_frag.glsl