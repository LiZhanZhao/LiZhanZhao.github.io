---
layout:     post
title:      "blender eevee Add Support For Volumetrics In Node Tree"
subtitle:   ""
date:       2021-2-17 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/27  * Eevee :  Add support for volumetrics in node tree.

> Only volume scatter is implemented for now.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/VolumetricsNodeTree/1.png)

## 作用 
在World中进行配置Volumetrics，目前只支持 volume scatter 节点

## 配置
![](/img/Eevee/VolumetricsNodeTree/2.png)
![](/img/Eevee/VolumetricsNodeTree/3.png)


## 编译
- 重新生成SLN
- git 定位到  2017/6/27  * Eevee : Add support for volumetrics in node tree .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 渲染前
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_Data *vedata)
{
	...
	if (BKE_collection_engine_property_value_get_bool(props, "volumetric_enable")) {
		/* Integration result */
		DRWFboTexture tex_vol = {&stl->g_data->volumetric, DRW_TEX_RGBA_16, DRW_TEX_MIPMAP | DRW_TEX_FILTER | DRW_TEX_TEMP};

		DRW_framebuffer_init(&fbl->volumetric_fb, &draw_engine_eevee_type,
		                    (int)viewport_size[0] / 2, (int)viewport_size[1] / 2,
		                    &tex_vol, 1);

		World *wo = scene->world;
		if ((wo != NULL) && (wo->use_nodes != NULL) && (wo->nodetree != NULL)) {
			effects->enabled_effects |= EFFECT_VOLUMETRIC;
		}
	}
}
```
>
- 只要打开了Volumetric后处理按钮，world 使用 nodetree 的话，就会 开启 EFFECT_VOLUMETRIC
- ![](/img/Eevee/VolumetricsNodeTree/4.png)

<br><br><br><br>


```
void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	if ((effects->enabled_effects & EFFECT_VOLUMETRIC) != 0) {
		const DRWContextState *draw_ctx = DRW_context_state_get();
		Scene *scene = draw_ctx->scene;
		struct World *wo = scene->world; /* Already checked non NULL */

		struct GPUMaterial *mat = EEVEE_material_world_volume_get(scene, wo);
		psl->volumetric_integrate_ps = DRW_pass_create("Volumetric Integration", DRW_STATE_WRITE_COLOR);
		DRWShadingGroup *grp = DRW_shgroup_material_create(mat, psl->volumetric_integrate_ps);

		if (grp != NULL) {
			DRW_shgroup_uniform_buffer(grp, "depthFull", &e_data.depth_src);
			DRW_shgroup_uniform_buffer(grp, "shadowCubes", &sldata->shadow_depth_cube_pool);
			DRW_shgroup_uniform_buffer(grp, "irradianceGrid", &sldata->irradiance_pool);
			DRW_shgroup_uniform_block(grp, "light_block", sldata->light_ubo);
			DRW_shgroup_uniform_block(grp, "grid_block", sldata->grid_ubo);
			DRW_shgroup_uniform_block(grp, "shadow_block", sldata->shadow_ubo);
			DRW_shgroup_uniform_int(grp, "light_count", &sldata->lamps->num_light, 1);
			DRW_shgroup_uniform_int(grp, "grid_count", &sldata->probes->num_render_grid, 1);
			DRW_shgroup_uniform_texture(grp, "utilTex", EEVEE_materials_get_util_tex());
			DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)stl->g_data->viewvecs, 2);
			DRW_shgroup_call_add(grp, quad, NULL);

			psl->volumetric_resolve_ps = DRW_pass_create("Volumetric Resolve", DRW_STATE_WRITE_COLOR | DRW_STATE_TRANSMISSION);
			grp = DRW_shgroup_create(e_data.volumetric_upsample_sh, psl->volumetric_resolve_ps);
			DRW_shgroup_uniform_buffer(grp, "depthFull", &e_data.depth_src);
			DRW_shgroup_uniform_buffer(grp, "volumetricBuffer", &stl->g_data->volumetric);
			DRW_shgroup_call_add(grp, quad, NULL);
		}
		else {
			/* Compilation failled */
			effects->enabled_effects &= ~EFFECT_VOLUMETRIC;
		}
	}
	...
}
```
>
- EFFECT_VOLUMETRIC 开启了之后就会跑进去if
- volumetric_integrate_ps 代码主要用来自函数 EEVEE_material_world_volume_get


*eevee_materials.c*
```
struct GPUMaterial *EEVEE_material_world_volume_get(struct Scene *scene, World *wo)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	int options = VAR_WORLD_VOLUME;

	GPUMaterial *mat = GPU_material_from_nodetree_find(&wo->gpumaterial, engine, options);
	if (mat != NULL) {
		return mat;
	}
	return GPU_material_from_nodetree(
	        scene, wo->nodetree, &wo->gpumaterial, engine, options,
	        datatoc_background_vert_glsl, NULL, e_data.volume_shader_lib,
	        SHADER_DEFINES "#define VOLUMETRICS\n");
}
```
>
- EEVEE_material_world_volume_get 函数主要是 生成 volumetric_integrate_ps 的Shader代码。
- volumetric_integrate_ps 由 background_vert.glsl 和 volumetric_frag.glsl 组成，而且是定义了 #define VOLUMETRICS， 生成代码的规则就是，在World的Node Tree 中，把连接到 World Output 的 Volume 的所有节点都生成出来。


## Shader
偶
*bsdf_common_lib.glsl*
```
#ifdef VOLUMETRICS

#define NODETREE_EXEC

struct Closure {
	vec3 absorption;
	vec3 scatter;
	vec3 emission;
	float anisotropy;
};

#define CLOSURE_DEFAULT Closure(vec3(0.0), vec3(0.0), vec3(0.0), 0.0);

Closure closure_mix(Closure cl1, Closure cl2, float fac)
{
	Closure cl;
	cl.absorption = mix(cl1.absorption, cl2.absorption, fac);
	cl.scatter = mix(cl1.scatter, cl2.scatter, fac);
	cl.emission = mix(cl1.emission, cl2.emission, fac);
	cl.anisotropy = mix(cl1.anisotropy, cl2.anisotropy, fac);
	return cl;
}

Closure closure_add(Closure cl1, Closure cl2)
{
	Closure cl;
	cl.absorption = cl1.absorption + cl2.absorption;
	cl.scatter = cl1.scatter + cl2.scatter;
	cl.emission = cl1.emission + cl2.emission;
	cl.anisotropy = cl1.anisotropy + cl2.anisotropy;
	return cl;
}

Closure nodetree_exec(void); /* Prototype */

#endif /* VOLUMETRICS */
```

<br><br>

*gpu_shader_material.glsl*
```
void node_background(vec4 color, float strength, out Closure result)
{
#ifndef VOLUMETRICS
	color *= strength;
	result = Closure(color.rgb, color.a);
#else
	result = CLOSURE_DEFAULT;
#endif
}

void node_volume_scatter(vec4 color, float density, float anisotropy, out Closure result)
{
#ifdef VOLUMETRICS
	result = Closure(vec3(0.0), color.rgb * density, vec3(0.0), anisotropy);
#else
	result = CLOSURE_DEFAULT;
#endif
}
```
>
- node_background 是节点 Background 的实现代码
- node_volume_scatter 是节点 Volume Scatter 的实现代码


<br><br>

*volumetric_frag.glsl*
```
void participating_media_properties(vec3 wpos, out vec3 absorption, out vec3 scattering, out float anisotropy)
{
	Closure cl = nodetree_exec();

	absorption = cl.absorption;
	scattering = cl.scatter;
	anisotropy = cl.anisotropy;
}
```
>
-  nodetree_exec 是生成的代码，如果在World的Node tree 中连接了 Volume Scatter 节点的话，那么这里的 nodetree_exec 就变成了 node_volume_scatter 里面的代码。
