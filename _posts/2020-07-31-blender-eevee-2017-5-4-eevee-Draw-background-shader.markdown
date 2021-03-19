---
layout:     post
title:      "blender eevee Draw background shader."
subtitle:   ""
date:       2020-08-21 24:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> 2017/5/4   * Eevee: Draw background shader.<br>


## 作用 
渲染背景，这个背景跟环境贴图保持一致



## 编译

- 直接编译


## 效果展示


![](/img/Eevee/BackgroundShader/1.png)


## CPU
**eevee_engine.c**

```c

static void EEVEE_draw_scene(void *vedata)
{
	...
	DRW_draw_pass(psl->background_pass);
	...
}
```

```c
static void EEVEE_engine_init(void *ved)
{
	...
	if (!e_data.default_background) {
		e_data.default_background = DRW_shader_create_fullscreen(datatoc_default_world_frag_glsl, NULL);
	}

	...
}
static void EEVEE_cache_init(void *vedata)
{
	...

	psl->background_pass = DRW_pass_create("Background Pass", DRW_STATE_WRITE_DEPTH | DRW_STATE_WRITE_COLOR);

	struct Batch *geom = DRW_cache_fullscreen_quad_get();
	DRWShadingGroup *grp = NULL;

	const DRWContextState *draw_ctx = DRW_context_state_get();
	Scene *scene = draw_ctx->scene;
	World *wo = scene->world;

	float *col = ts.colorBackground;
	if (wo) {
		col = &wo->horr;
	}

	if (wo && wo->use_nodes && wo->nodetree) {
		struct GPUMaterial *gpumat = GPU_material_from_nodetree(
			scene, wo->nodetree, &wo->gpumaterial, &DRW_engine_viewport_eevee_type, 1,
			datatoc_background_vert_glsl, NULL, e_data.frag_shader_lib,
			"#define WORLD_BACKGROUND\n"
			"#define MAX_LIGHT 128\n"
			"#define MAX_SHADOW_CUBE 42\n"
			"#define MAX_SHADOW_MAP 64\n"
			"#define MAX_SHADOW_CASCADE 8\n"
			"#define MAX_CASCADE_NUM 4\n");

		grp = DRW_shgroup_material_create(gpumat, psl->background_pass);

		if (grp) {
			DRW_shgroup_call_add(grp, geom, NULL);
		}
		else {
			/* Shader failed : pink background */
			static float pink[3] = {1.0f, 0.0f, 1.0f};
			col = pink;
		}
	}

	/* Fallback if shader fails or if not using nodetree. */
	if (grp == NULL) {
		grp = DRW_shgroup_create(e_data.default_background, psl->background_pass);
		DRW_shgroup_uniform_vec3(grp, "color", col, 1);
		DRW_shgroup_call_add(grp, geom, NULL);
	}

	...
}
```
>
- 简单理解， psl->background_pass 可以使用 wo->nodetree 生成出来的shader来作为渲染，如果不使用 wo->nodetree 的话，就直接用 e_data.default_background， 用 wo->nodetree 生成代码可以参考 [World Node Tree](http://shaderstore.cn/2020/07/31/blender-eevee-2017-4-28-eevee-World-nodetree-gpumaterial-compatibility/)
- e_data.default_background 使用了 default_world_frag.glsl ， e_data.default_background 只是简单输出一个颜色

## Shader
** background_vert.glsl **
```glsl

mat4 ViewProjectionMatrixInverse;

in vec2 pos;

out vec3 varposition;
out vec3 varnormal;
out vec3 viewPosition;
out vec3 worldPosition;

void main()
{
	gl_Position = vec4(pos, 1.0, 1.0);
	varposition = viewPosition = vec3(pos, -1.0);
	varnormal = normalize(-varposition);
}

```

** default_world_frag.glsl **
```

uniform vec3 color;

out vec4 FragColor;

void main() {
	FragColor = vec4(color, 1.0);
}

```