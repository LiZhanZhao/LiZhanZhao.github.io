---
layout:     post
title:      "blender eevee Transparency Add support for blend ADD and MULTIPLY."
subtitle:   ""
date:       2021-2-21 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/10  *  Eevee: Transparency: Add support for blend ADD and MULTIPLY .<br> 

> This introduces a new transparency pass.
It bypass the radial distance encoding in alpha for the transparent shaders.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/BlendAddAndMultiply/1.png)
![](/img/Eevee/BlendAddAndMultiply/2.png)
![](/img/Eevee/BlendAddAndMultiply/3.png)


## 作用
blend ADD 和 MULTIPLY 的实现

## 编译
- 重新生成SLN
- git 定位到  2017/7/10  * Eevee: Transparency: Add support for blend ADD and MULTIPLY .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 渲染

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
    ...
    /* Transparent */
	DRW_draw_pass(psl->transparent_pass);

    /* Post Process */
	EEVEE_draw_effects(vedata);
    
}
```
>
- 在渲染管线中 多了 transparent_pass 来渲染物体, transparent_pass 是在后处理(EEVEE_draw_effects)的前一步进行。


## 渲染前
*eevee_materials.c*
```
void EEVEE_materials_cache_init(EEVEE_Data *vedata)
{
    ...
    {
		DRWState state = DRW_STATE_WRITE_COLOR | DRW_STATE_DEPTH_LESS | DRW_STATE_CLIP_PLANES | DRW_STATE_WIRE;
		psl->transparent_pass = DRW_pass_create("Material Transparent Pass", state);
	}
    ...
}
```
>
- transparent_pass 渲染透明物体，渲染状态没有进行写深度

<br><br>

```
void EEVEE_materials_cache_populate(EEVEE_Data *vedata, EEVEE_SceneLayerData *sldata, Object *ob)
{
    ...
    for (int i = 0; i < materials_len; ++i) {
        Material *ma = give_current_material(ob, i + 1);

        gpumat_array[i] = NULL;
        gpumat_depth_array[i] = NULL;
        shgrp_array[i] = NULL;
        shgrp_depth_array[i] = NULL;
        shgrp_depth_clip_array[i] = NULL;

        if (ma == NULL)
            ma = &defmaterial;

        switch (ma->blend_method) {
            case MA_BM_SOLID:
            case MA_BM_CLIP:
            case MA_BM_HASHED:
                material_opaque(ma, material_hash, sldata, vedata, do_cull, use_flat_nor,
                        &gpumat_array[i], &gpumat_depth_array[i],
                        &shgrp_array[i], &shgrp_depth_array[i], &shgrp_depth_clip_array[i]);
                break;
            case MA_BM_ADD:
            case MA_BM_MULTIPLY:
                material_transparent(ma, sldata, vedata, do_cull, use_flat_nor,
                        &gpumat_array[i], &shgrp_array[i]);
                break;
        }
    }
    ...
}
```
>
- 代码这里就比较清晰了，如果是Solid, Clip, Hashed 就会跑进去material_opaque，如果是Blend Add 和 Blend Multiply 就会跑进去material_transparent。

## material_opaque

*eevee_materials.c*
```
static void material_opaque(
        Material *ma, GHash *material_hash, EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata,
        bool do_cull, bool use_flat_nor, struct GPUMaterial **gpumat, struct GPUMaterial **gpumat_depth,
        struct DRWShadingGroup **shgrp, struct DRWShadingGroup **shgrp_depth, struct DRWShadingGroup **shgrp_depth_clip)
{
	const DRWContextState *draw_ctx = DRW_context_state_get();
	Scene *scene = draw_ctx->scene;
	EEVEE_StorageList *stl = ((EEVEE_Data *)vedata)->stl;
	EEVEE_PassList *psl = ((EEVEE_Data *)vedata)->psl;

	float *color_p = &ma->r;
	float *metal_p = &ma->ray_mirror;
	float *spec_p = &ma->spec;
	float *rough_p = &ma->gloss_mir;

	const bool use_gpumat = (ma->use_nodes && ma->nodetree);

	EeveeMaterialShadingGroups *emsg = BLI_ghash_lookup(material_hash, (const void *)ma);

	if (emsg) {
		*shgrp = emsg->shading_grp;
		*shgrp_depth = emsg->depth_grp;
		*shgrp_depth_clip = emsg->depth_clip_grp;

		/* This will have been created already, just perform a lookup. */
		*gpumat = (use_gpumat) ? EEVEE_material_mesh_get(
		        scene, ma, stl->effects->use_ao, stl->effects->use_bent_normals, false, false) : NULL;
		*gpumat_depth = (use_gpumat) ? EEVEE_material_mesh_depth_get(
		        scene, ma, (ma->blend_method == MA_BM_HASHED)) : NULL;
		return;
	}

	if (use_gpumat) {
		/* Shading */
		*gpumat = EEVEE_material_mesh_get(scene, ma,
		        stl->effects->use_ao, stl->effects->use_bent_normals, false, false);

		*shgrp = DRW_shgroup_material_create(*gpumat, psl->material_pass);
		if (*shgrp) {
			add_standard_uniforms(*shgrp, sldata, vedata);
		}
		else {
			/* Shader failed : pink color */
			static float col[3] = {1.0f, 0.0f, 1.0f};
			static float half = 0.5f;

			color_p = col;
			metal_p = spec_p = rough_p = &half;
		}

		/* Alpha CLipped : Discard pixel from depth pass, then
		 * fail the depth test for shading. */
		if (ELEM(ma->blend_method, MA_BM_CLIP, MA_BM_HASHED)) {
			*gpumat_depth = EEVEE_material_mesh_depth_get(scene, ma,
			        (ma->blend_method == MA_BM_HASHED));

			*shgrp_depth = DRW_shgroup_material_create(*gpumat_depth, do_cull ? psl->depth_pass_cull : psl->depth_pass);
			*shgrp_depth_clip = DRW_shgroup_material_create(*gpumat_depth, do_cull ? psl->depth_pass_clip_cull : psl->depth_pass_clip);

			if (shgrp_depth) {
				if (ma->blend_method == MA_BM_CLIP) {
					DRW_shgroup_uniform_float(*shgrp_depth, "alphaThreshold", &ma->alpha_threshold, 1);
					DRW_shgroup_uniform_float(*shgrp_depth_clip, "alphaThreshold", &ma->alpha_threshold, 1);
				}
			}
		}
	}

	/* Fallback to default shader */
	if (*shgrp == NULL) {
		*shgrp = EEVEE_default_shading_group_get(sldata, vedata, false, use_flat_nor,
		        stl->effects->use_ao, stl->effects->use_bent_normals);
		DRW_shgroup_uniform_vec3(*shgrp, "basecol", color_p, 1);
		DRW_shgroup_uniform_float(*shgrp, "metallic", metal_p, 1);
		DRW_shgroup_uniform_float(*shgrp, "specular", spec_p, 1);
		DRW_shgroup_uniform_float(*shgrp, "roughness", rough_p, 1);
	}

	/* Fallback default depth prepass */
	if (*shgrp_depth == NULL) {
		*shgrp_depth = do_cull ? stl->g_data->depth_shgrp_cull : stl->g_data->depth_shgrp;
		*shgrp_depth_clip = do_cull ? stl->g_data->depth_shgrp_clip_cull : stl->g_data->depth_shgrp_clip;
	}

	emsg = MEM_mallocN(sizeof("EeveeMaterialShadingGroups"), "EeveeMaterialShadingGroups");
	emsg->shading_grp = *shgrp;
	emsg->depth_grp = *shgrp_depth;
	emsg->depth_clip_grp = *shgrp_depth_clip;
	BLI_ghash_insert(material_hash, ma, emsg);
}
```
>
- 实体模型主要是 关联 material_pass 和 depth_pass_cull，depth_pass，和 depth_pass_clip_cull，depth_pass_clip


## material_transparent
*eevee_materials.c*
```

struct GPUMaterial *EEVEE_material_mesh_get(
        struct Scene *scene, Material *ma,
        bool use_ao, bool use_bent_normals, bool use_blend, bool use_multiply)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	int options = VAR_MAT_MESH;

	if (use_ao) options |= VAR_MAT_AO;
	if (use_bent_normals) options |= VAR_MAT_BENT;
	if (use_blend) options |= VAR_MAT_BLEND;
	if (use_multiply) options |= VAR_MAT_MULT;

	GPUMaterial *mat = GPU_material_from_nodetree_find(&ma->gpumaterial, engine, options);
	if (mat) {
		return mat;
	}

	char *defines = eevee_get_defines(options);

	mat = GPU_material_from_nodetree(
	        scene, ma->nodetree, &ma->gpumaterial, engine, options,
	        datatoc_lit_surface_vert_glsl, NULL, e_data.frag_shader_lib,
	        defines);

	MEM_freeN(defines);

	return mat;
}

static void material_transparent(
        Material *ma, EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata,
        bool do_cull, bool use_flat_nor, struct GPUMaterial **gpumat, struct DRWShadingGroup **shgrp)
{
	const DRWContextState *draw_ctx = DRW_context_state_get();
	Scene *scene = draw_ctx->scene;
	EEVEE_StorageList *stl = ((EEVEE_Data *)vedata)->stl;
	EEVEE_PassList *psl = ((EEVEE_Data *)vedata)->psl;

	float *color_p = &ma->r;
	float *metal_p = &ma->ray_mirror;
	float *spec_p = &ma->spec;
	float *rough_p = &ma->gloss_mir;

	if (ma->use_nodes && ma->nodetree) {
		/* Shading */
		*gpumat = EEVEE_material_mesh_get(scene, ma,
		        stl->effects->use_ao, stl->effects->use_bent_normals,
		        true, (ma->blend_method == MA_BM_MULTIPLY));

		*shgrp = DRW_shgroup_material_create(*gpumat, psl->transparent_pass);
		if (*shgrp) {
			add_standard_uniforms(*shgrp, sldata, vedata);
		}
		else {
			/* Shader failed : pink color */
			static float col[3] = {1.0f, 0.0f, 1.0f};
			static float half = 0.5f;

			color_p = col;
			metal_p = spec_p = rough_p = &half;
		}
	}

	/* Fallback to default shader */
	if (*shgrp == NULL) {
		*shgrp = EEVEE_default_shading_group_create(
		        sldata, vedata, psl->transparent_pass,
		        false, use_flat_nor, stl->effects->use_ao, stl->effects->use_bent_normals, true);
		DRW_shgroup_uniform_vec3(*shgrp, "basecol", color_p, 1);
		DRW_shgroup_uniform_float(*shgrp, "metallic", metal_p, 1);
		DRW_shgroup_uniform_float(*shgrp, "specular", spec_p, 1);
		DRW_shgroup_uniform_float(*shgrp, "roughness", rough_p, 1);
	}

	DRWState cur_state = (do_cull) ? DRW_STATE_CULL_BACK : 0;
	DRWState all_state = DRW_STATE_CULL_BACK | DRW_STATE_BLEND | DRW_STATE_ADDITIVE | DRW_STATE_MULTIPLY;

	switch (ma->blend_method) {
		case MA_BM_ADD:
			cur_state |= DRW_STATE_ADDITIVE;
			break;
		case MA_BM_MULTIPLY:
			cur_state |= DRW_STATE_MULTIPLY;
			break;
		default:
			BLI_assert(0);
			break;
	}

	/* Disable other blend modes and use the one we want. */
	DRW_shgroup_state_disable(*shgrp, all_state);
	DRW_shgroup_state_enable(*shgrp, cur_state);
}
```
>
- 代码可以看到，物体材质是关联 transparent_pass, 材质Shader代码会设置宏 VAR_MAT_BLEND。
- 根据 ma->blend_method == MA_BM_MULTIPLY 来是否设置 宏 VAR_MAT_MULT。
- 材质生成Shader代码文件主要是 虽然是生成代码，但是代码主要来自lit_surface_vert.glsl 和 lit_surface_frag.glsl, bsdf_common_lib.glsl
- 根据blend_method的选择，来定义宏 DRW_STATE_ADDITIVE 或者 DRW_STATE_MULTIPLY
- 如果不使用nodetree的话，就使用default_frag.glsl


## Shader

*default_frag.glsl*
```

uniform vec3 basecol;
uniform float metallic;
uniform float specular;
uniform float roughness;

out vec4 FragColor;

void main()
{
	vec3 dielectric = vec3(0.034) * specular * 2.0;
	vec3 diffuse = mix(basecol, vec3(0.0), metallic);
	vec3 f0 = mix(dielectric, basecol, metallic);
	vec3 radiance = eevee_surface_lit((gl_FrontFacing) ? worldNormal : -worldNormal, diffuse, f0, roughness, 1.0);
#if defined(USE_ALPHA_BLEND)
	FragColor = vec4(radiance, 1.0);
#else
	FragColor = vec4(radiance, length(viewPosition));
#endif
}

```
>
- 定义了USE_ALPHA_BLEND 的话，alpha默认就是1.

<br><br>

*bsdf_common_lib.glsl*
```
...
.../* TODO find a better place */
#ifdef USE_MULTIPLY

out vec4 fragColor;

#define NODETREE_EXEC
void main()
{
	Closure cl = nodetree_exec();
	fragColor = vec4(mix(vec3(1.0), cl.radiance, cl.opacity), 1.0);
}
#endif
...
```
>
- 使用NodeTree的时候调用
- 如果定义了 USE_MULTIPLY 的话，就用这个main函数，不使用下面nodeTree生成的main函数

<br><br>
```
/* ********************** matcap style render ******************** */

void material_preview_matcap(vec4 color, sampler2D ima, vec4 N, vec4 mask, out vec4 result)
{
	...
}

//------------------------------------- 以上的都是gpu_shader_material.glsl代码
//------------------------------------- 以下才是生成的代码

in vec3 var0;
const mat4 cons2 = mat4(1.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 1.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 1.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 1.000000000000);
const vec3 cons3 = vec3(0.000000000000, 0.000000000000, 0.000000000000);
const vec3 cons4 = vec3(1.000000000000, 1.000000000000, 1.000000000000);
const float cons5 = float(0.000000000000);
const float cons6 = float(0.000000000000);
uniform sampler2D samp0;

Closure nodetree_exec(void)
{
	vec3 tmp7;
	vec4 tmp10;
	float tmp11;
	vec4 tmp13;
	Closure tmp15;
	Closure tmp17;

	mapping(var0, cons2, cons3, cons4, cons5, cons6, tmp7);
	node_tex_image(tmp7, samp0, tmp10, tmp11);
	srgb_to_linearrgb(tmp10, tmp13);
	node_bsdf_transparent(tmp13, tmp15);
	node_output_eevee_material(tmp15, tmp17);

	return tmp17;
}
#ifndef NODETREE_EXEC
out vec4 fragColor;
void main()
{
	Closure cl = nodetree_exec();
	fragColor = vec4(cl.radiance, cl.opacity);
}
#endif

```

<br><br>

# Transparency: Add object center Z sorting

## 来源

- 主要看这个commit

> GIT : 2017/7/10  *  Eevee: Transparency: Add object center Z sorting .<br> 

> Better algo should take bounding box center, but it's not referenced yet in the draw call and cannot be tweaked by user.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51

## 作用
透明物体渲染排序

## 渲染

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Transparent */
	DRW_pass_sort_shgroup_z(psl->transparent_pass);
	DRW_draw_pass(psl->transparent_pass);
	...
}

```


*draw_manager.c*
```
typedef struct ZSortData {
	float *axis;
	float *origin;
} ZSortData;

static int pass_shgroup_dist_sort(void *thunk, const void *a, const void *b)
{
	const DRWCall *call_a = (DRWCall *)((const DRWShadingGroup *)a)->calls.first;
	const DRWCall *call_b = (DRWCall *)((const DRWShadingGroup *)b)->calls.first;
	const ZSortData *zsortdata = (ZSortData *)thunk;

	float tmp[3];
	sub_v3_v3v3(tmp, zsortdata->origin, call_a->obmat[3]);
	const float a_sq = dot_v3v3(zsortdata->axis, tmp);
	sub_v3_v3v3(tmp, zsortdata->origin, call_b->obmat[3]);
	const float b_sq = dot_v3v3(zsortdata->axis, tmp);

	if      (a_sq < b_sq) return  1;
	else if (a_sq > b_sq) return -1;
	else                  return  0;
}

/**
 * Sort Shading groups by decreasing Z between
 * the first call object center and a given world space point.
 **/
void DRW_pass_sort_shgroup_z(DRWPass *pass)
{
	RegionView3D *rv3d = DST.draw_ctx.rv3d;

	float (*viewinv)[4];
	viewinv = (viewport_matrix_override.override[DRW_MAT_VIEWINV])
	          ? viewport_matrix_override.mat[DRW_MAT_VIEWINV] : rv3d->viewinv;

	ZSortData zsortdata = {viewinv[2], viewinv[3]};
	BLI_listbase_sort_r(&pass->shgroups, pass_shgroup_dist_sort, &zsortdata);
}
```
>
- ZSortData zsortdata = {viewinv[2], viewinv[3]}; 对应的是 struct ZSortData 的 axis 和 origin
- pass_shgroup_dist_sort 的核心就是，先算出a,b 物体在Camera 的 Z 轴的 距离，然后进行排序，一般都是 距离远的先渲染