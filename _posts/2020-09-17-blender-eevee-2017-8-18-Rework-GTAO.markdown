---
layout:     post
title:      "blender eevee Rework GTAO"
subtitle:   ""
date:       2021-3-19 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/8/18  *   Eevee : Rework GTAO <br> 

> 
This includes big improvement:
- The horizon search is decoupled from the BSDF evaluation. This means using multiple BSDF nodes have a much lower impact when enbaling AO.
- The horizon search is optimized by splitting the search into 4 corners searching similar directions to help which GPU cache coherence.
- The AO options are now uniforms and do not trigger shader recompilation (aka. freeze UI).
- Include a quality slider similar to the SSR one.
- Add a switch for disabling bounce light approximation.
- Fix problem with Bent Normals when occlusion get very dark.
- Add a denoise option to that takes the neighbors pixel values via glsl derivatives. This reduces noise but exhibit 2x2 blocky artifacts.

> 
he downside : Separating the horizon search uses more memory (~3MB for each samples on HD viewport). We could lower the bit depth to 4bit per horizon but it produce noticeable banding (might be fixed with some dithering).

> SVN : 2017/8/17  [windows] add numpy headers to python include dir, required for D2716


## 编译
要定位到这个 commit 才可以正常编译看效果

> GIT : 2017/8/30  *   Eevee: Fix conditional statement depending on unitialized value <br> 



## 效果

*关闭GTAO*
![](/img/Eevee/GTAO/02/1.png)
<br>
*打开GTAO*
![](/img/Eevee/GTAO/02/2.png)

## 作用
优化GTAO，分离 Compute GTAO Horizons  的步骤

<br>

## 准备
可以先阅读 [这里](http://shaderstore.cn/2021/02/05/blender-eevee-2017-6-22-Ambient-Occlusion-Initial-Implementation/) 补充理论知识

<br>

## 1. 渲染流程

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Create minmax texture */
	DRW_stats_group_start("Main MinMax buffer");
	EEVEE_create_minmax_buffer(vedata, dtxl->depth, -1);
	DRW_stats_group_end();

	/* Compute GTAO Horizons */
	EEVEE_effects_do_gtao(sldata, vedata);

	/* Restore main FB */
	DRW_framebuffer_bind(fbl->main);

	/* Shading pass */
	DRW_stats_group_start("Shading");
	DRW_draw_pass(psl->background_pass);
	EEVEE_draw_default_passes(psl);
	DRW_draw_pass(psl->material_pass);
	DRW_stats_group_end();
	...
}
```
>
- 渲染流程多了 EEVEE_effects_do_gtao 函数，而且 EEVEE_effects_do_gtao 在 DRW_draw_pass(psl->material_pass); 之前就执行了
- EEVEE_effects_do_gtao 的作用是 Compute GTAO Horizons 
<br>


## 2. EEVEE_effects_do_gtao

*eevee_effects.c*
```
void EEVEE_effects_do_gtao(EEVEE_SceneLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_GTAO) != 0) {
		DefaultTextureList *dtxl = DRW_viewport_texture_list_get();
		e_data.depth_src = dtxl->depth;

		DRW_stats_group_start("GTAO Horizon Scan");
		for (effects->ao_sample_nbr = 0.0;
		     effects->ao_sample_nbr < effects->ao_samples;
		     ++effects->ao_sample_nbr)
		{
			DRW_framebuffer_texture_detach(txl->gtao_horizons);
			DRW_framebuffer_texture_layer_attach(fbl->gtao_fb, txl->gtao_horizons, 0, (int)effects->ao_sample_nbr, 0);
			DRW_framebuffer_bind(fbl->gtao_fb);

			DRW_draw_pass(psl->ao_horizon_search);
		}
		DRW_stats_group_end();

		/* Restore */
		DRW_framebuffer_bind(fbl->main);
	}
}
```

>
- EEVEE_effects_do_gtao 函数主要是利用 psl->ao_horizon_search Pass 渲染 txl->gtao_horizons RT

<br>


## 3. Pass ao_horizon_search 
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	DynStr *ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, datatoc_bsdf_common_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_ambient_occlusion_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_effect_gtao_frag_glsl);
	char *frag_str = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	e_data.gtao_sh = DRW_shader_create_fullscreen(frag_str, NULL);
	e_data.gtao_debug_sh = DRW_shader_create_fullscreen(frag_str, "#define DEBUG_AO\n");

	MEM_freeN(frag_str);
	...

	if (BKE_collection_engine_property_value_get_bool(props, "gtao_enable")) {
		/* Ambient Occlusion*/
		effects->enabled_effects |= EFFECT_GTAO;

		effects->ao_dist = BKE_collection_engine_property_value_get_float(props, "gtao_distance");
		effects->ao_factor = BKE_collection_engine_property_value_get_float(props, "gtao_factor");
		effects->ao_quality = 1.0f - BKE_collection_engine_property_value_get_float(props, "gtao_quality");
		effects->ao_samples = BKE_collection_engine_property_value_get_int(props, "gtao_samples");
		effects->ao_samples_inv = 1.0f / effects->ao_samples;

		effects->ao_settings = 1.0; /* USE_AO */
		if (BKE_collection_engine_property_value_get_bool(props, "gtao_use_bent_normals")) {
			effects->ao_settings += 2.0; /* USE_BENT_NORMAL */
		}
		if (BKE_collection_engine_property_value_get_bool(props, "gtao_denoise")) {
			effects->ao_settings += 4.0; /* USE_DENOISE */
		}

		effects->ao_offset = 0.0f;
		effects->ao_bounce_fac = (float)BKE_collection_engine_property_value_get_bool(props, "gtao_bounce");

		effects->ao_texsize[0] = ((int)viewport_size[0]);
		effects->ao_texsize[1] = ((int)viewport_size[1]);

		/* Round up to multiple of 2 */
		if ((effects->ao_texsize[0] & 0x1) != 0) {
			effects->ao_texsize[0] += 1;
		}
		if ((effects->ao_texsize[1] & 0x1) != 0) {
			effects->ao_texsize[1] += 1;
		}

		CLAMP(effects->ao_samples, 1, 32);

		if (effects->hori_tex_layers != effects->ao_samples) {
			DRW_TEXTURE_FREE_SAFE(txl->gtao_horizons);
		}

		if (txl->gtao_horizons == NULL) {
			effects->hori_tex_layers = effects->ao_samples;
			txl->gtao_horizons = DRW_texture_create_2D_array((int)viewport_size[0], (int)viewport_size[1], effects->hori_tex_layers, DRW_TEX_RG_8, 0, NULL);
		}

		DRWFboTexture tex = {&txl->gtao_horizons, DRW_TEX_RG_8, 0};

		DRW_framebuffer_init(&fbl->gtao_fb, &draw_engine_eevee_type,
		                    effects->ao_texsize[0], effects->ao_texsize[1],
		                    &tex, 1);

		if (G.debug_value == 6) {
			DRWFboTexture tex_debug = {&stl->g_data->gtao_horizons_debug, DRW_TEX_RGBA_8, DRW_TEX_TEMP};

			DRW_framebuffer_init(&fbl->gtao_debug_fb, &draw_engine_eevee_type,
			                    (int)viewport_size[0], (int)viewport_size[1],
			                    &tex_debug, 1);
		}
	}
	else {
		/* Cleanup */
		DRW_TEXTURE_FREE_SAFE(txl->gtao_horizons);
		DRW_FRAMEBUFFER_FREE_SAFE(fbl->gtao_fb);
		effects->ao_settings = 0.0f;
	}
}

void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	psl->ao_horizon_search = DRW_pass_create("GTAO Horizon Search", DRW_STATE_WRITE_COLOR);
	DRWShadingGroup *grp = DRW_shgroup_create(e_data.gtao_sh, psl->ao_horizon_search);
	DRW_shgroup_uniform_buffer(grp, "maxzBuffer", &txl->maxzbuffer);
	DRW_shgroup_uniform_buffer(grp, "depthBuffer", &e_data.depth_src);
	DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)stl->g_data->viewvecs, 2);
	DRW_shgroup_uniform_vec2(grp, "mipRatio[0]", (float *)stl->g_data->mip_ratio, 10);
	DRW_shgroup_uniform_vec4(grp, "aoParameters[0]", &stl->effects->ao_dist, 2);
	DRW_shgroup_uniform_float(grp, "sampleNbr", &stl->effects->ao_sample_nbr, 1);
	DRW_shgroup_uniform_ivec2(grp, "aoHorizonTexSize", (int *)stl->effects->ao_texsize, 1);
	DRW_shgroup_uniform_texture(grp, "utilTex", EEVEE_materials_get_util_tex());
	DRW_shgroup_call_add(grp, quad, NULL);
	...
}
```
>
- ao_horizon_search Pass 主要后处理，使用了bsdf_common_lib.glsl 和 ambient_occlusion_lib.glsl 和 effect_gtao_frag.glsl
- txl->gtao_horizons RT 是一个TextureArray，有 effects->ao_samples 个元素
- 所以 ao_horizon_search Pass 是渲染这个 TextureArray的每一个元素

<br>

## 4. effect_gtao_frag.glsl 

*effect_gtao_frag.glsl*

```
/**
 * This shader only compute maximum horizon angles for each directions.
 * The final integration is done at the resolve stage with the shading normal.
 **/

uniform float rotationOffset;

out vec4 FragColor;

#ifdef DEBUG_AO
uniform sampler2D normalBuffer;

void main()
{
	vec4 texel_size = 1.0 / vec2(textureSize(depthBuffer, 0)).xyxy;
	vec2 uvs = saturate(gl_FragCoord.xy * texel_size.xy);

	float depth = textureLod(depthBuffer, uvs, 0.0).r;

	vec3 viewPosition = get_view_space_from_depth(uvs, depth);
	vec3 V = viewCameraVec;
	vec3 normal = normal_decode(texture(normalBuffer, uvs).rg, V);

	vec3 bent_normal;
	float visibility;
#if 1
	gtao_deferred(normal, viewPosition, depth, visibility, bent_normal);
#else
	vec2 rand = vec2((1.0 / 4.0) * float((int(gl_FragCoord.y) & 0x1) * 2 + (int(gl_FragCoord.x) & 0x1)), 0.5);
	rand = fract(rand.x + texture(utilTex, vec3(floor(gl_FragCoord.xy * 0.5) / LUT_SIZE, 2.0)).rg);
	gtao(normal, viewPosition, rand, visibility, bent_normal);
#endif
	denoise_ao(normal, depth, visibility, bent_normal);

	FragColor = vec4(visibility);
}

#else
uniform float sampleNbr;

void main()
{
	ivec2 hr_co = ivec2(gl_FragCoord.xy);
	ivec2 fs_co = get_fs_co(hr_co);

	vec2 uvs = saturate((vec2(fs_co) + 0.5) / vec2(textureSize(depthBuffer, 0)));
	float depth = textureLod(depthBuffer, uvs, 0.0).r;

	if (depth == 1.0) {
		/* Do not trace for background */
		FragColor = vec4(0.0);
		return;
	}

	/* Avoid self shadowing. */
	depth = saturate(depth - 3e-6); /* Tweaked for 24bit depth buffer. */

	vec3 viewPosition = get_view_space_from_depth(uvs, depth);

	float phi = get_phi(hr_co, fs_co, sampleNbr);
	float offset = get_offset(fs_co, sampleNbr);
	vec2 max_dir = get_max_dir(viewPosition.z);

	FragColor.xy = search_horizon_sweep(phi, viewPosition, uvs, offset, max_dir);

	/* Resize output for integer texture. */
	FragColor = pack_horizons(FragColor.xy).xyxy;
}
#endif

```
>
- 理论的可以参考 [这里](http://shaderstore.cn/2021/02/05/blender-eevee-2017-6-22-Ambient-Occlusion-Initial-Implementation/)
- 这里简单地理解就是把 用后处理 计算 h1 和 h2, 并保存下来供之后使用



## 5. lit_surface_frag.glsl 和应用

```
vec3 eevee_surface_lit(vec3 N, vec3 albedo, vec3 f0, float roughness, float ao, int ssr_id, out vec3 ssr_spec)
{
	...
	/* Ambient Occlusion */
	vec3 bent_normal;
	float final_ao = occlusion_compute(N, viewPosition, ao, rand.rg, bent_normal);
	...
}
```

<br>

*eevee_materials.c* 

```
static void add_standard_uniforms(
        DRWShadingGroup *shgrp, EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata,
        int *ssr_id, float *refract_depth, bool use_ssrefraction)
{
	...
	DRW_shgroup_uniform_vec4(shgrp, "aoParameters[0]", &vedata->stl->effects->ao_dist, 2);
	...
	if (vedata->stl->effects->use_ao) {
		DRW_shgroup_uniform_buffer(shgrp, "horizonBuffer", &vedata->txl->gtao_horizons);
		DRW_shgroup_uniform_ivec2(shgrp, "aoHorizonTexSize", (int *)vedata->stl->effects->ao_texsize, 1);
	}
	...	
}
```
>
- 这里可以知道是传入什么参数到Shader lit_surface_frag.glsl 中, 可以看到上面的 vedata->txl->gtao_horizons 也传入来了, vedata->txl->gtao_horizons 保存了 h1 和 h2.
- occlusion_compute 函数中调用 gtao_deferred 就会用到 vedata->txl->gtao_horizons RT。




## 6. ambient_occlustion_lib.glsl

*ambient_occlusion_lib.glsl*
```

/* Based on Practical Realtime Strategies for Accurate Indirect Occlusion
 * http://blog.selfshadow.com/publications/s2016-shading-course/activision/s2016_pbs_activision_occlusion.pdf
 * http://blog.selfshadow.com/publications/s2016-shading-course/activision/s2016_pbs_activision_occlusion.pptx */

#define MAX_PHI_STEP 32
#define MAX_SEARCH_ITER 32
#define MAX_LOD 6.0

#ifndef UTIL_TEX
#define UTIL_TEX
uniform sampler2DArray utilTex;
#endif /* UTIL_TEX */

uniform vec4 aoParameters[2];
uniform sampler2DArray horizonBuffer;

/* Cannot use textureSize(horizonBuffer) when rendering to it */
uniform ivec2 aoHorizonTexSize;

#define aoDistance   aoParameters[0].x
#define aoSamples    aoParameters[0].y
#define aoFactor     aoParameters[0].z
#define aoInvSamples aoParameters[0].w

#define aoOffset     aoParameters[1].x /* UNUSED */
#define aoBounceFac  aoParameters[1].y
#define aoQuality    aoParameters[1].z
#define aoSettings   aoParameters[1].w

#define USE_AO            1
#define USE_BENT_NORMAL   2
#define USE_DENOISE       4

vec2 pack_horizons(vec2 v) { return v * 0.5 + 0.5; }
vec2 unpack_horizons(vec2 v) { return v * 2.0 - 1.0; }

/* Returns the texel coordinate in horizonBuffer
 * for a given fullscreen coord */
ivec2 get_hr_co(ivec2 fs_co)
{
	bvec2 quarter = notEqual(fs_co & ivec2(1), ivec2(0));

	ivec2 hr_co = fs_co / 2;
	hr_co += ivec2(quarter) * (aoHorizonTexSize / 2);

	return hr_co;
}

/* Returns the texel coordinate in fullscreen (depthBuffer)
 * for a given horizonBuffer coord */
ivec2 get_fs_co(ivec2 hr_co)
{
	hr_co *= 2;
	bvec2 quarter = greaterThanEqual(hr_co, aoHorizonTexSize);

	hr_co -= ivec2(quarter) * (aoHorizonTexSize - 1);

	return hr_co;
}

/* Returns the phi angle in horizonBuffer
 * for a given horizonBuffer coord */
float get_phi(ivec2 hr_co, ivec2 fs_co, float sample)
{
	bvec2 quarter = greaterThanEqual(hr_co, aoHorizonTexSize / 2);
	ivec2 tex_co = ((int(aoSettings) & USE_DENOISE) != 0) ? hr_co - ivec2(quarter) * (aoHorizonTexSize / 2) : fs_co;
	float blue_noise = texture(utilTex, vec3((vec2(tex_co) + 0.5) / LUT_SIZE, 2.0)).r;

	float phi = sample * aoInvSamples;

	if ((int(aoSettings) & USE_DENOISE) != 0) {
		/* Interleaved jitter for spatial 2x2 denoising */
		phi += 0.25 * aoInvSamples * (float(quarter.x) + 2.0 * float(quarter.y));
		blue_noise *= 0.25;
	}
	/* Blue noise is scaled to cover the rest of the range. */
	phi += aoInvSamples * blue_noise;
	phi *= M_PI;

	return phi;
}

/* Returns direction jittered offset for a given fullscreen coord */
float get_offset(ivec2 fs_co, float sample)
{
	float offset = sample * aoInvSamples;

	/* Interleaved jitter for spatial 2x2 denoising */
	offset += 0.25 * dot(vec2(1.0), vec2(fs_co & 1));
	offset += texture(utilTex, vec3((vec2(fs_co / 2) + 0.5 + 16.0) / LUT_SIZE, 2.0)).r;
	return offset;
}

/* Returns maximum screen distance an AO ray can travel for a given view depth */
vec2 get_max_dir(float view_depth)
{
	float homcco = ProjectionMatrix[2][3] * view_depth + ProjectionMatrix[3][3];
	float max_dist = aoDistance / homcco;
	return vec2(ProjectionMatrix[0][0], ProjectionMatrix[1][1]) * max_dist;
}

void get_max_horizon_grouped(vec4 co1, vec4 co2, vec3 x, float lod, inout float h)
{
	co1 *= mipRatio[int(lod + 1.0)].xyxy; /* +1 because we are using half res top level */
	co2 *= mipRatio[int(lod + 1.0)].xyxy; /* +1 because we are using half res top level */

	float depth1 = textureLod(maxzBuffer, co1.xy, floor(lod)).r;
	float depth2 = textureLod(maxzBuffer, co1.zw, floor(lod)).r;
	float depth3 = textureLod(maxzBuffer, co2.xy, floor(lod)).r;
	float depth4 = textureLod(maxzBuffer, co2.zw, floor(lod)).r;

	vec4 len, s_h;

	vec3 s1 = get_view_space_from_depth(co1.xy, depth1); /* s View coordinate */
	vec3 omega_s1 = s1 - x;
	len.x = length(omega_s1);
	s_h.x = omega_s1.z / len.x;

	vec3 s2 = get_view_space_from_depth(co1.zw, depth2); /* s View coordinate */
	vec3 omega_s2 = s2 - x;
	len.y = length(omega_s2);
	s_h.y = omega_s2.z / len.y;

	vec3 s3 = get_view_space_from_depth(co2.xy, depth3); /* s View coordinate */
	vec3 omega_s3 = s3 - x;
	len.z = length(omega_s3);
	s_h.z = omega_s3.z / len.z;

	vec3 s4 = get_view_space_from_depth(co2.zw, depth4); /* s View coordinate */
	vec3 omega_s4 = s4 - x;
	len.w = length(omega_s4);
	s_h.w = omega_s4.z / len.w;

	/* Blend weight after half the aoDistance to fade artifacts */
	vec4 blend = saturate((1.0 - len / aoDistance) * 2.0);

	h = mix(h, max(h, s_h.x), blend.x);
	h = mix(h, max(h, s_h.y), blend.y);
	h = mix(h, max(h, s_h.z), blend.z);
	h = mix(h, max(h, s_h.w), blend.w);
}

vec2 search_horizon_sweep(float phi, vec3 pos, vec2 uvs, float jitter, vec2 max_dir)
{
	vec2 t_phi = vec2(cos(phi), sin(phi)); /* Screen space direction */

	max_dir *= max_v2(abs(t_phi));

	/* Convert to pixel space. */
	t_phi /= vec2(textureSize(maxzBuffer, 0));

	/* Avoid division by 0 */
	t_phi += vec2(1e-5);

	jitter *= 0.25;

	/* Compute end points */
	vec2 corner1 = min(vec2(1.0) - uvs,  max_dir); /* Top right */
	vec2 corner2 = max(vec2(0.0) - uvs, -max_dir); /* Bottom left */
	vec2 iter1 = corner1 / t_phi;
	vec2 iter2 = corner2 / t_phi;

	vec2 min_iter = max(-iter1, -iter2);
	vec2 max_iter = max( iter1,  iter2);

	vec2 times = vec2(-min_v2(min_iter), min_v2(max_iter));

	vec2 h = vec2(-1.0); /* init at cos(pi) */

	/* This is freaking sexy optimized. */
	for (float i = 0.0, ofs = 4.0, time = -1.0;
		 i < MAX_SEARCH_ITER && time > times.x;
		 i++, time -= ofs, ofs = min(exp2(MAX_LOD) * 4.0, ofs + ofs * aoQuality))
	{
		vec4 t = max(times.xxxx, vec4(time) - (vec4(0.25, 0.5, 0.75, 1.0) - jitter) * ofs);
		vec4 cos1 = uvs.xyxy + t_phi.xyxy * t.xxyy;
		vec4 cos2 = uvs.xyxy + t_phi.xyxy * t.zzww;
		float lod = min(MAX_LOD, max(i - jitter * 4.0, 0.0) * aoQuality);
		get_max_horizon_grouped(cos1, cos2, pos, lod, h.y);
	}

	for (float i = 0.0, ofs = 4.0, time = 1.0;
		 i < MAX_SEARCH_ITER && time < times.y;
		 i++, time += ofs, ofs = min(exp2(MAX_LOD) * 4.0, ofs + ofs * aoQuality))
	{
		vec4 t = min(times.yyyy, vec4(time) + (vec4(0.25, 0.5, 0.75, 1.0) - jitter) * ofs);
		vec4 cos1 = uvs.xyxy + t_phi.xyxy * t.xxyy;
		vec4 cos2 = uvs.xyxy + t_phi.xyxy * t.zzww;
		float lod = min(MAX_LOD, max(i - jitter * 4.0, 0.0) * aoQuality);
		get_max_horizon_grouped(cos1, cos2, pos, lod, h.x);
	}

	return h;
}

void integrate_slice(vec3 normal, float phi, vec2 horizons, inout float visibility, inout vec3 bent_normal)
{
	/* TODO OPTI Could be precomputed. */
	vec2 t_phi = vec2(cos(phi), sin(phi)); /* Screen space direction */

	/* Projecting Normal to Plane P defined by t_phi and omega_o */
	vec3 np = vec3(t_phi.y, -t_phi.x, 0.0); /* Normal vector to Integration plane */
	vec3 t = vec3(-t_phi, 0.0);
	vec3 n_proj = normal - np * dot(np, normal);
	float n_proj_len = max(1e-16, length(n_proj));

	float cos_n = clamp(n_proj.z / n_proj_len, -1.0, 1.0);
	float n = sign(dot(n_proj, t)) * fast_acos(cos_n); /* Angle between view vec and normal */

	/* (Slide 54) */
	vec2 h = fast_acos(horizons);
	h.x = -h.x;

	/* Clamping thetas (slide 58) */
	h.x = n + max(h.x - n, -M_PI_2);
	h.y = n + min(h.y - n, M_PI_2);

	/* Solving inner integral */
	vec2 h_2 = 2.0 * h;
	vec2 vd = -cos(h_2 - n) + cos_n + h_2 * sin(n);
	float vis = (vd.x + vd.y) * 0.25 * n_proj_len;

	visibility += vis;

	/* Finding Bent normal */
	float b_angle = (h.x + h.y) * 0.5;
	/* The 0.5 factor below is here to equilibrate the accumulated vectors.
	 * (sin(b_angle) * -t_phi) will accumulate to (phi_step * result_nor.xy * 0.5).
	 * (cos(b_angle) * 0.5) will accumulate to (phi_step * result_nor.z * 0.5). */
	bent_normal += vec3(sin(b_angle) * -t_phi, cos(b_angle) * 0.5);
}

void denoise_ao(vec3 normal, float frag_depth, inout float visibility, inout vec3 bent_normal)
{
	vec2 d_sign = vec2(ivec2(gl_FragCoord.xy) & 1) - 0.5;

	if ((int(aoSettings) & USE_DENOISE) == 0) {
		d_sign *= 0.0;
	}

	/* 2x2 Bilateral Filter using derivatives. */
	vec2 n_step = step(-0.2, -abs(vec2(length(dFdx(normal)), length(dFdy(normal)))));
	vec2 z_step = step(-0.1, -abs(vec2(dFdx(frag_depth), dFdy(frag_depth))));

	visibility -= dFdx(visibility) * d_sign.x * z_step.x * n_step.x;
	visibility -= dFdy(visibility) * d_sign.y * z_step.y * n_step.y;

	bent_normal -= dFdx(bent_normal) * d_sign.x * z_step.x * n_step.x;
	bent_normal -= dFdy(bent_normal) * d_sign.y * z_step.y * n_step.y;
}

void gtao_deferred(vec3 normal, vec3 position, float frag_depth, out float visibility, out vec3 bent_normal)
{
	vec2 uvs = get_uvs_from_view(position);

	vec4 texel_size = vec4(-1.0, -1.0, 1.0, 1.0) / vec2(textureSize(depthBuffer, 0)).xyxy;

	ivec2 fs_co = ivec2(gl_FragCoord.xy);
	ivec2 hr_co = get_hr_co(fs_co);

	bent_normal = vec3(0.0);
	visibility = 0.0;

	for (float i = 0.0; i < MAX_PHI_STEP; i++) {
		if (i >= aoSamples) break;

		vec2 horiz = unpack_horizons(texelFetch(horizonBuffer, ivec3(hr_co, int(i)), 0).rg);
		float phi = get_phi(hr_co, fs_co, i);

		integrate_slice(normal, phi, horiz.xy, visibility, bent_normal);
	}

	visibility *= aoInvSamples;
	bent_normal = normalize(bent_normal);
}

void gtao(vec3 normal, vec3 position, vec2 noise, out float visibility, out vec3 bent_normal)
{
	vec2 uvs = get_uvs_from_view(position);

	float homcco = ProjectionMatrix[2][3] * position.z + ProjectionMatrix[3][3];
	float max_dist = aoDistance / homcco; /* Search distance */
	vec2 max_dir = max_dist * vec2(ProjectionMatrix[0][0], ProjectionMatrix[1][1]);

	bent_normal = vec3(0.0);
	visibility = 0.0;

	for (float i = 0.0; i < MAX_PHI_STEP; i++) {
		if (i >= aoSamples) break;

		float phi = M_PI * (i + noise.x) * aoInvSamples;
		vec2 horizons = search_horizon_sweep(phi, position, uvs, noise.g, max_dir);

		integrate_slice(normal, phi, horizons, visibility, bent_normal);
	}

	visibility *= aoInvSamples;
	bent_normal = normalize(bent_normal);
}

/* Multibounce approximation base on surface albedo.
 * Page 78 in the .pdf version. */
float gtao_multibounce(float visibility, vec3 albedo)
{
	if (aoBounceFac == 0.0) return visibility;

	/* Median luminance. Because Colored multibounce looks bad. */
	float lum = dot(albedo, vec3(0.3333));

	float a =  2.0404 * lum - 0.3324;
	float b = -4.7951 * lum + 0.6417;
	float c =  2.7552 * lum + 0.6903;

	float x = visibility;
	return max(x, ((x * a + b) * x + c) * x);
}

/* Use the right occlusion  */
float occlusion_compute(vec3 N, vec3 vpos, float user_occlusion, vec2 randuv, out vec3 bent_normal)
{
	if ((int(aoSettings) & USE_AO) == 0) {
		bent_normal = N;
		return user_occlusion;
	}
	else {
		float visibility;
		vec3 vnor = mat3(ViewMatrix) * N;

#if defined(MESH_SHADER) && !defined(USE_ALPHA_HASH) && !defined(USE_ALPHA_CLIP) && !defined(SHADOW_SHADER) && !defined(USE_MULTIPLY) && !defined(USE_ALPHA_BLEND)
		gtao_deferred(vnor, vpos, gl_FragCoord.z, visibility, bent_normal);
#else
		gtao(vnor, vpos, randuv, visibility, bent_normal);
#endif
		denoise_ao(vnor, gl_FragCoord.z, visibility, bent_normal);

		/* Prevent some problems down the road. */
		visibility = max(1e-3, visibility);

		if ((int(aoSettings) & USE_BENT_NORMAL) != 0) {
			/* The bent normal will show the facet look of the mesh. Try to minimize this. */
			float mix_fac = visibility * visibility;
			bent_normal = normalize(mix(bent_normal, vnor, mix_fac));

			bent_normal = transform_direction(ViewMatrixInverse, bent_normal);
		}
		else {
			bent_normal = N;
		}

		/* Scale by user factor */
		visibility = pow(visibility, aoFactor);

		return min(visibility, user_occlusion);
	}
}

```
