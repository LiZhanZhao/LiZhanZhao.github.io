---
layout:     post
title:      "blender eevee Refraction Make it available for opaque materials"
subtitle:   ""
date:       2021-3-19 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/8/10  *   Eevee: Refraction: Make it available for opaque materials .<br> 

> Theses Materials are rendered after the SSR pass.
The only difference with previous method is that they have a depth prepass (less overdraw) and are not sorted.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
*效果*
![](/img/Eevee/SSR/Refraction/4.png)


## 作用
后处理折射效果实现, 实体模型都可以折射

<br>

## 渲染管线

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Prepare Refraction */
	EEVEE_effects_do_refraction(sldata, vedata);

	/* Restore main FB */
	DRW_framebuffer_bind(fbl->main);

	/* Opaque refraction */
	DRW_stats_group_start("Opaque Refraction");
	DRW_draw_pass(psl->refract_depth_pass);
	DRW_draw_pass(psl->refract_depth_pass_cull);
	DRW_draw_pass(psl->refract_pass);
	DRW_stats_group_end();

	/* Transparent */
	DRW_pass_sort_shgroup_z(psl->transparent_pass);
	DRW_stats_group_start("Transparent");
	DRW_draw_pass(psl->transparent_pass);
	DRW_stats_group_end();
	...
}
```
>
- 这里先执行 EEVEE_effects_do_refraction, 再执行 Opaque Refraction, 再执行 Transparent
- 如果物体的材质使用的 Refraction的话，那么他们就会在 Opaque Refraction 这个阶段继续渲染
- 那么就比较好理解了，就是如果物体的材质开启了 Refraction 开关的话，就会在 EEVEE_effects_do_refraction(sldata, vedata); 之后执行。

<br>

### 1. EEVEE_effects_do_refraction
这里就不多说了，主要是 看 [Add Screnn Space Refraction](http://shaderstore.cn/2021/03/18/blender-eevee-2017-8-9-Add-Screen-Space-Refraction/)


### 2. Opaque Refraction
#### a. 初始化
*eevee_materials.c*
```
void EEVEE_materials_cache_init(EEVEE_Data *vedata)
{
	...
	{
		DRWState state = DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS | DRW_STATE_WIRE;
		psl->refract_depth_pass = DRW_pass_create("Refract Depth Pass", state);
		stl->g_data->refract_depth_shgrp = DRW_shgroup_create(e_data.default_prepass_sh, psl->refract_depth_pass);

		state = DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS | DRW_STATE_CULL_BACK;
		psl->refract_depth_pass_cull = DRW_pass_create("Refract Depth Pass Cull", state);
		stl->g_data->refract_depth_shgrp_cull = DRW_shgroup_create(e_data.default_prepass_sh, psl->refract_depth_pass_cull);

		state = DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS | DRW_STATE_CLIP_PLANES | DRW_STATE_WIRE;
		psl->refract_depth_pass_clip = DRW_pass_create("Refract Depth Pass Clip", state);
		stl->g_data->refract_depth_shgrp_clip = DRW_shgroup_create(e_data.default_prepass_clip_sh, psl->refract_depth_pass_clip);

		state = DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS | DRW_STATE_CLIP_PLANES | DRW_STATE_CULL_BACK;
		psl->refract_depth_pass_clip_cull = DRW_pass_create("Refract Depth Pass Cull Clip", state);
		stl->g_data->refract_depth_shgrp_clip_cull = DRW_shgroup_create(e_data.default_prepass_clip_sh, psl->refract_depth_pass_clip_cull);
	}

	{
		DRWState state = DRW_STATE_WRITE_COLOR | DRW_STATE_DEPTH_EQUAL | DRW_STATE_CLIP_PLANES | DRW_STATE_WIRE;
		psl->refract_pass = DRW_pass_create("Opaque Refraction Pass", state);
	}
	...
}
```
>
- 这里主要是 初始化 psl->refract_depth_pass, psl->refract_depth_pass_cull, psl->refract_depth_pass_clip, psl->refract_depth_pass_clip_cull 和 psl->refract_pass
- default_prepass_sh 使用 prepass_vert.glsl 和 prepass_frag.glsl
- default_prepass_clip_sh 使用 prepass_vert.glsl 和 prepass_frag.glsl , 但是是添加了 define CLIP_PLANES


#### b. prepass_vert.glsl 和 prepass_frag.glsl
*prepass_vert.glsl*
```

uniform mat4 ModelViewProjectionMatrix;
uniform mat4 ModelMatrix;
#ifdef CLIP_PLANES
uniform vec4 ClipPlanes[1];
#endif

in vec3 pos;

void main()
{
	gl_Position = ModelViewProjectionMatrix * vec4(pos, 1.0);
#ifdef CLIP_PLANES
	vec4 worldPosition = (ModelMatrix * vec4(pos, 1.0));
	gl_ClipDistance[0] = dot(worldPosition, ClipPlanes[0]);
#endif
	/* TODO motion vectors */
}

```

<br>

*prepass_frag.glsl*
```

#ifdef USE_ALPHA_HASH

/* From the paper "Hashed Alpha Testing" by Chris Wyman and Morgan McGuire */
float hash(vec2 a) {
	return fract(1e4 * sin(17.0 * a.x + 0.1 * a.y) * (0.1 + abs(sin(13.0 * a.y + a.x))));
}

float hash3d(vec3 a) {
	return hash(vec2(hash(a.xy), a.z));
}

//uniform float hashScale;
float hashScale = 0.001;

float hashed_alpha_threshold(vec3 co)
{
	/* Find the discretized derivatives of our coordinates. */
	float max_deriv = max(length(dFdx(co)), length(dFdy(co)));
	float pix_scale = 1.0 / (hashScale * max_deriv);

	/* Find two nearest log-discretized noise scales. */
	float pix_scale_log = log2(pix_scale);
	vec2 pix_scales;
	pix_scales.x = exp2(floor(pix_scale_log));
	pix_scales.y = exp2(ceil(pix_scale_log));

	/* Compute alpha thresholds at our two noise scales. */
	vec2 alpha;
	alpha.x = hash3d(floor(pix_scales.x * co));
	alpha.y = hash3d(floor(pix_scales.y * co));

	/* Factor to interpolate lerp with. */
	float fac = fract(log2(pix_scale));

	/* Interpolate alpha threshold from noise at two scales. */
	float x = mix(alpha.x, alpha.y, fac);

	/* Pass into CDF to compute uniformly distrib threshold. */
	float a = min(fac, 1.0 - fac);
	float one_a = 1.0 - a;
	float denom = 1.0 / (2 * a * one_a);
	float one_x = (1 - x);
	vec3 cases = vec3(
		(x * x) * denom,
		(x - 0.5 * a) / one_a,
		1.0 - (one_x * one_x * denom)
	);

	/* Find our final, uniformly distributed alpha threshold. */
	float threshold = (x < one_a) ?	((x < a) ? cases.x : cases.y) :	cases.z;

	/* Avoids threshold == 0. */
	threshold = clamp(threshold, 1.0e-6, 1.0);

	return threshold;
}

#endif

#ifdef USE_ALPHA_CLIP
uniform float alphaThreshold;
#endif

#ifdef SHADOW_SHADER
out vec4 FragColor;
#endif

void main()
{
	/* For now do nothing.
	 * In the future, output object motion blur. */

#if defined(USE_ALPHA_HASH) || defined(USE_ALPHA_CLIP)
#define NODETREE_EXEC

	Closure cl = nodetree_exec();

#if defined(USE_ALPHA_HASH)
	/* Hashed Alpha Testing */
	if (cl.opacity < hashed_alpha_threshold(worldPosition))
		discard;
#elif defined(USE_ALPHA_CLIP)
	/* Alpha clip */
	if (cl.opacity <= alphaThreshold)
		discard;
#endif
#endif

#ifdef SHADOW_SHADER
	float dist = distance(lampPosition.xyz, worldPosition.xyz);
	FragColor = vec4(dist, 0.0, 0.0, 1.0);
#endif
}

```

#### c.material_opaque

*eevee_materials.c*
```
static void material_opaque(
        Material *ma, GHash *material_hash, EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata,
        bool do_cull, bool use_flat_nor, struct GPUMaterial **gpumat, struct GPUMaterial **gpumat_depth,
        struct DRWShadingGroup **shgrp, struct DRWShadingGroup **shgrp_depth, struct DRWShadingGroup **shgrp_depth_clip)
{
	...

	const bool use_gpumat = (ma->use_nodes && ma->nodetree);
	const bool use_refract = ((ma->blend_flag & MA_BL_SS_REFRACTION) != 0) && ((stl->effects->enabled_effects & EFFECT_REFRACT) != 0);

	EeveeMaterialShadingGroups *emsg = BLI_ghash_lookup(material_hash, (const void *)ma);

	if (emsg) {
		*shgrp = emsg->shading_grp;
		*shgrp_depth = emsg->depth_grp;
		*shgrp_depth_clip = emsg->depth_clip_grp;

		/* This will have been created already, just perform a lookup. */
		*gpumat = (use_gpumat) ? EEVEE_material_mesh_get(
		        scene, ma, stl->effects->use_ao, stl->effects->use_bent_normals, false, false, use_refract) : NULL;
		*gpumat_depth = (use_gpumat) ? EEVEE_material_mesh_depth_get(
		        scene, ma, (ma->blend_method == MA_BM_HASHED), false) : NULL;
		return;
	}

	if (use_gpumat) {
		/* Shading */
		*gpumat = EEVEE_material_mesh_get(scene, ma,
		        stl->effects->use_ao, stl->effects->use_bent_normals, false, false, use_refract);

		*shgrp = DRW_shgroup_material_create(*gpumat, use_refract ? psl->refract_pass : psl->material_pass);
		if (*shgrp) {
			static int ssr_id;
			static float refract_thickness = 0.0f; /* TODO Param */
			ssr_id = (stl->effects->use_ssr) ? 0 : -1;
			add_standard_uniforms(*shgrp, sldata, vedata, &ssr_id, (use_refract) ? &refract_thickness : NULL);
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
			        (ma->blend_method == MA_BM_HASHED), false);

			if (use_refract) {
				*shgrp_depth = DRW_shgroup_material_create(*gpumat_depth, (do_cull) ? psl->refract_depth_pass_cull : psl->refract_depth_pass);
				*shgrp_depth_clip = DRW_shgroup_material_create(*gpumat_depth, (do_cull) ? psl->refract_depth_pass_clip_cull : psl->refract_depth_pass_clip);
			}
			else {
				*shgrp_depth = DRW_shgroup_material_create(*gpumat_depth, (do_cull) ? psl->depth_pass_cull : psl->depth_pass);
				*shgrp_depth_clip = DRW_shgroup_material_create(*gpumat_depth, (do_cull) ? psl->depth_pass_clip_cull : psl->depth_pass_clip);
			}

			if (*shgrp != NULL) {
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
		        stl->effects->use_ao, stl->effects->use_bent_normals, stl->effects->use_ssr);
		DRW_shgroup_uniform_vec3(*shgrp, "basecol", color_p, 1);
		DRW_shgroup_uniform_float(*shgrp, "metallic", metal_p, 1);
		DRW_shgroup_uniform_float(*shgrp, "specular", spec_p, 1);
		DRW_shgroup_uniform_float(*shgrp, "roughness", rough_p, 1);
	}

	/* Fallback default depth prepass */
	if (*shgrp_depth == NULL) {
		if (use_refract) {
			*shgrp_depth = (do_cull) ? stl->g_data->refract_depth_shgrp_cull : stl->g_data->refract_depth_shgrp;
			*shgrp_depth_clip = (do_cull) ? stl->g_data->refract_depth_shgrp_clip_cull : stl->g_data->refract_depth_shgrp_clip;
		}
		else {
			*shgrp_depth = (do_cull) ? stl->g_data->depth_shgrp_cull : stl->g_data->depth_shgrp;
			*shgrp_depth_clip = (do_cull) ? stl->g_data->depth_shgrp_clip_cull : stl->g_data->depth_shgrp_clip;
		}
	}

	emsg = MEM_mallocN(sizeof("EeveeMaterialShadingGroups"), "EeveeMaterialShadingGroups");
	emsg->shading_grp = *shgrp;
	emsg->depth_grp = *shgrp_depth;
	emsg->depth_clip_grp = *shgrp_depth_clip;
	BLI_ghash_insert(material_hash, ma, emsg);
}
```
>
- 这里比较关键的是 use_refract 这个开关，
- *shgrp = DRW_shgroup_material_create(*gpumat, use_refract ? psl->refract_pass : psl->material_pass); 如果打开了 use_refract  的话，就把物体渲染归纳在 refract_pass Pass，如果没有打开 use_refract 的话，就是 material_pass
- *shgrp_depth = (do_cull) ? stl->g_data->refract_depth_shgrp_cull : stl->g_data->refract_depth_shgrp; 也是一样的道理
- *shgrp_depth_clip = (do_cull) ? stl->g_data->refract_depth_shgrp_clip_cull : stl->g_data->refract_depth_shgrp_clip; 也是一样的道理

