---
layout:     post
title:      "blender eevee Initial implementation of Volumetrics"
subtitle:   ""
date:       2021-2-10 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/3  * Eevee :  Initial implementation of Volumetrics.

> This method is very cheap and inaccurate. This will fill the gap untill better model is supported.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/Volumetrics/1.png)
![](/img/Eevee/Volumetrics/2.png)

## 作用 
开始实现 Volumetrics


## 编译
- 重新生成SLN
- git 定位到  2017/7/3  * Eevee : Initial implementation of Volumetrics .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## 渲染
*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Volumetrics */
	EEVEE_effects_do_volumetrics(vedata);
	...
}
```

*eevee_effects.c*
```
void EEVEE_effects_do_volumetrics(EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_VOLUMETRIC) != 0) {
		DefaultTextureList *dtxl = DRW_viewport_texture_list_get();

		e_data.depth_src = dtxl->depth;

		/* Compute volumetric integration at halfres. */
		DRW_framebuffer_texture_attach(fbl->volumetric_fb, stl->g_data->volumetric, 0, 0);
		DRW_framebuffer_bind(fbl->volumetric_fb);
		DRW_draw_pass(psl->volumetric_integrate_ps);

		/* Resolve at fullres */
		DRW_framebuffer_texture_detach(dtxl->depth);
		DRW_framebuffer_bind(fbl->main);
		DRW_draw_pass(psl->volumetric_resolve_ps);

		/* Restore */
		DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0, 0);
	}
}
```
>
- 只要在界面上开启了Volumetrics后处理，才会进行渲染
- Volumetrics 用到两个Pass
- 一个是 volumetric_integrate_ps 把渲染结果渲染到 stl->g_data->volumetric RT上
- 一个是 volumetric_resolve_ps ，利用 stl->g_data->volumetric RT 把东西渲染到 fbl->main 上.


## volumetric_integrate_ps

### 初始化

*eevee_materials.c*
```
void EEVEE_materials_init(EEVEE_StorageList *stl)
{
	...
	ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, e_data.frag_shader_lib);
	BLI_dynstr_append(ds_frag, datatoc_volumetric_frag_glsl);
	frag_str = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	e_data.default_volume_sh = DRW_shader_create_fullscreen(frag_str, SHADER_DEFINES "#define STEP_INTEGRATE\n");
	...
}

struct GPUShader *EEVEE_material_world_volume_get(struct Scene *UNUSED(scene), World *UNUSED(wo))
{
	return e_data.default_volume_sh;
}

```

*eevee_effects.c*
```
void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	struct Gwn_Batch *quad = DRW_cache_fullscreen_quad_get();
	...
	struct GPUShader *sh = EEVEE_material_world_volume_get(NULL, NULL);
	psl->volumetric_integrate_ps = DRW_pass_create("Volumetric Integration", DRW_STATE_WRITE_COLOR);
	DRWShadingGroup *grp = DRW_shgroup_create(sh, psl->volumetric_integrate_ps);
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
	...
}
```
>
- volumetric_integrate_ps 使用后处理，最主要的ps是 volumetric_frag.glsl, 定义了宏 #define STEP_INTEGRATE


### Shader

*volumetric_frag.glsl* 定义了 STEP_INTEGRATE 的内容

```
out vec4 FragColor;

//#ifdef STEP_INTEGRATE

uniform sampler2D depthFull;

float find_next_step(float near, float far, float noise, int iter, int iter_count)
{
	const float lambda = 0.8f; /* TODO : Parameter */

	float progress = (float(iter) + noise) / float(iter_count);

	float linear_split = mix(near, far, progress);

	if (ProjectionMatrix[3][3] == 0.0) {
		float exp_split = near * pow(far / near, progress);
		return mix(linear_split, exp_split, lambda);
	}
	else {
		return linear_split;
	}
}

void participating_media_properties(vec3 wpos, out vec3 absorption, out vec3 scattering, out float anisotropy)
{
	/* TODO Call nodetree from here. */
	absorption = vec3(0.00);
	scattering = vec3(1.0) * step(-1.0, -wpos.z);

	anisotropy = -0.8;
}

float phase_function_isotropic()
{
	return 1.0 / (4.0 * M_PI);
}

float phase_function(vec3 v, vec3 l, float g)
{
#if 0
	/* Henyey-Greenstein */
	float cos_theta = dot(v, l);
	float sqr_g = g * g;
	return (1- sqr_g) / (4.0 * M_PI * pow(1 + sqr_g - 2 * g * cos_theta, 3.0 / 2.0));
#else
	return phase_function_isotropic();
#endif
}

vec3 light_volume(LightData ld, vec4 l_vector, vec3 l_col)
{
	float dist = max(1e-4, abs(l_vector.w - ld.l_radius));
	return l_col * (4.0 * ld.l_radius * ld.l_radius * M_PI * M_PI) / (dist * dist);
}

/* Based on Frosbite Unified Volumetric.
 * https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite */
void main()
{
	vec2 uv = (gl_FragCoord.xy * 2.0) / ivec2(textureSize(depthFull, 0));
	float scene_depth = texelFetch(depthFull, ivec2(gl_FragCoord.xy) * 2, 0).r; /* use the same depth as in the upsample step */
	vec3 vpos = get_view_space_from_depth(uv, scene_depth);
	vec3 wpos = (ViewMatrixInverse * vec4(vpos, 1.0)).xyz;
	vec3 wdir = (ProjectionMatrix[3][3] == 0.0) ? normalize(cameraPos - wpos) : cameraForward;

	/* Note: this is NOT the distance to the camera. */
	float max_z = vpos.z;

	/* project ray to clip plane so we can integrate in even steps in clip space. */
	vec3 wdir_proj = wdir / abs(dot(cameraForward, wdir));
	float wlen = length(wdir_proj);

	/* Transmittance: How much light can get through. */
	vec3 transmittance = vec3(1.0);

	/* Scattering: Light that has been accumulated from scattered light sources. */
	vec3 scattering = vec3(0.0);

	vec3 ray_origin = (ProjectionMatrix[3][3] == 0.0)
		? cameraPos
		: (ViewMatrixInverse * vec4(get_view_space_from_depth(uv, 0.5), 1.0)).xyz;

	/* Start from near clip. TODO make start distance an option. */
	float rand = texture(utilTex, vec3(gl_FragCoord.xy / LUT_SIZE, 2.0)).r;
	/* Less noisy but noticeable patterns, could work better with temporal AA. */
	// float rand = (1.0 / 16.0) * float(((int(gl_FragCoord.x + gl_FragCoord.y) & 0x3) << 2) + (int(gl_FragCoord.x) & 0x3));
	float near = get_view_z_from_depth(0.0);
	float far  = get_view_z_from_depth(1.0);
	float dist = near;
	for (int i = 1; i < 64; ++i) {
		float new_dist = find_next_step(near, far, rand, i, 64);
		float step = dist - new_dist; /* Marching step */
		dist = new_dist;

		vec3 ray_wpos = ray_origin + wdir_proj * dist;

		/* Volume Sample */
		vec3 s_absorption, s_scattering; /* mu_a, mu_s */
		float s_anisotropy;
		participating_media_properties(ray_wpos, s_absorption, s_scattering, s_anisotropy);

		vec3 s_extinction = max(vec3(1e-8), s_absorption + s_scattering); /* mu_t */

		/* Evaluate each light */
		vec3 Lscat = vec3(0.0);

#if 1 /* Lights */
		for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
			LightData ld = lights_data[i];

			vec4 l_vector;
			l_vector.xyz = ld.l_position - ray_wpos;
			l_vector.w = length(l_vector.xyz);

#if 1 /* Shadows & Spots */
			float Vis = light_visibility(ld, ray_wpos, l_vector);
#else
			float Vis = 1.0;
#endif
			vec3 Li = light_volume(ld, l_vector, ld.l_color);

			Lscat += Li * Vis * s_scattering * phase_function(-wdir, l_vector.xyz / l_vector.w, s_anisotropy);
		}
#endif

		/* Environment : Average color. */
		IrradianceData ir_data = load_irradiance_cell(0, vec3(1.0));
		Lscat += (ir_data.cubesides[0] + ir_data.cubesides[1] + ir_data.cubesides[2]) * 0.333333 * s_scattering * phase_function_isotropic();

		ir_data = load_irradiance_cell(0, vec3(-1.0));
		Lscat += (ir_data.cubesides[0] + ir_data.cubesides[1] + ir_data.cubesides[2]) * 0.333333 * s_scattering * phase_function_isotropic();

		/* Evaluate Scattering */
		float s_len = wlen * step;
		vec3 Tr = exp(-s_extinction * s_len);

		/* integrate along the current step segment */
		Lscat = (Lscat - Lscat * Tr) / s_extinction;
		/* accumulate and also take into account the transmittance from previous steps */
		scattering += transmittance * Lscat;

		/* Evaluate transmittance to view independantely */
		transmittance *= Tr;

		if (dist < max_z)
			break;
	}

	float mono_transmittance = dot(transmittance, vec3(1.0)) / 3.0;

	FragColor = vec4(scattering, mono_transmittance);
}

// #else /* STEP_UPSAMPLE */
```
>
- 以上的理论知识都在 [这里](https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite)
- 理论的公式
- ![](/img/Eevee/Volumetrics/3-0.png)
- ![](/img/Eevee/Volumetrics/3-1.png)
- ![](/img/Eevee/Volumetrics/4.png)
- ![](/img/Eevee/Volumetrics/5.png)
- 上面的代码基本上都是按照图片的公式来进行计算。
- stl->g_data->volumetric RT上保存了 scattering 和 mono_transmittance。


## volumetric_resolve_ps

### 初始化
*eevee_effects.c*
```

void EEVEE_effects_init(EEVEE_Data *vedata)
{
	...
	e_data.volumetric_upsample_sh = DRW_shader_create_fullscreen(datatoc_volumetric_frag_glsl, "#define STEP_UPSAMPLE\n");
	...
}

void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	struct Gwn_Batch *quad = DRW_cache_fullscreen_quad_get();
	...
	psl->volumetric_resolve_ps = DRW_pass_create("Volumetric Resolve", DRW_STATE_WRITE_COLOR | DRW_STATE_TRANSMISSION);
		grp = DRW_shgroup_create(e_data.volumetric_upsample_sh, psl->volumetric_resolve_ps);
		DRW_shgroup_uniform_buffer(grp, "depthFull", &e_data.depth_src);
		DRW_shgroup_uniform_buffer(grp, "volumetricBuffer", &stl->g_data->volumetric);
		DRW_shgroup_call_add(grp, quad, NULL);
	...
}
```
>
- volumetric_resolve_ps 使用后处理，最主要的ps是 volumetric_frag.glsl, 定义了宏 #define STEP_UPSAMPLE


### Shader

*volumetric_frag.glsl* 定义了 STEP_UPSAMPLE 的内容
```

uniform sampler2D depthFull;
uniform sampler2D volumetricBuffer;

uniform mat4 ProjectionMatrix;

vec4 get_view_z_from_depth(vec4 depth)
{
	vec4 d = 2.0 * depth - 1.0;
	return -ProjectionMatrix[3][2] / (d + ProjectionMatrix[2][2]);
}

void main()
{
#if 0 /* 2 x 2 with bilinear */

	const vec4 bilinear_weights[4] = vec4[4](
		vec4(9.0 / 16.0,  3.0 / 16.0, 3.0 / 16.0, 1.0 / 16.0 ),
		vec4(3.0 / 16.0,  9.0 / 16.0, 1.0 / 16.0, 3.0 / 16.0 ),
		vec4(3.0 / 16.0,  1.0 / 16.0, 9.0 / 16.0, 3.0 / 16.0 ),
		vec4(1.0 / 16.0,  3.0 / 16.0, 3.0 / 16.0, 9.0 / 16.0 )
	);

	/* Depth aware upsampling */
	vec4 depths;
	ivec2 texel_co = ivec2(gl_FragCoord.xy * 0.5) * 2;

	/* TODO use textureGather on glsl 4.0 */
	depths.x = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 0)).r;
	depths.y = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 0)).r;
	depths.z = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 2)).r;
	depths.w = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 2)).r;

	vec4 target_depth = texelFetch(depthFull, ivec2(gl_FragCoord.xy), 0).rrrr;

	depths = get_view_z_from_depth(depths);
	target_depth = get_view_z_from_depth(target_depth);

	vec4 weights = 1.0 - step(0.05, abs(depths - target_depth));

	/* Index in range [0-3] */
	int pix_id = int(dot(mod(ivec2(gl_FragCoord.xy), 2), ivec2(1, 2)));
	weights *= bilinear_weights[pix_id];

	float weight_sum = dot(weights, vec4(1.0));

	if (weight_sum == 0.0) {
		weights.x = 1.0;
		weight_sum = 1.0;
	}

	texel_co = ivec2(gl_FragCoord.xy * 0.5);

	vec4 integration_result;
	integration_result  = texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 0)) * weights.x;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 0)) * weights.y;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 1)) * weights.z;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 1)) * weights.w;

#else /* 4 x 4 */

	/* Depth aware upsampling */
	vec4 depths[4];
	ivec2 texel_co = ivec2(gl_FragCoord.xy * 0.5) * 2;

	/* TODO use textureGather on glsl 4.0 */
	texel_co += ivec2(-2, -2);
	depths[0].x = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 0)).r;
	depths[0].y = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 0)).r;
	depths[0].z = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 2)).r;
	depths[0].w = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 2)).r;

	texel_co += ivec2(4, 0);
	depths[1].x = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 0)).r;
	depths[1].y = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 0)).r;
	depths[1].z = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 2)).r;
	depths[1].w = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 2)).r;

	texel_co += ivec2(-4, 4);
	depths[2].x = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 0)).r;
	depths[2].y = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 0)).r;
	depths[2].z = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 2)).r;
	depths[2].w = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 2)).r;

	texel_co += ivec2(4, 0);
	depths[3].x = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 0)).r;
	depths[3].y = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 0)).r;
	depths[3].z = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 2)).r;
	depths[3].w = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 2)).r;

	vec4 target_depth = texelFetch(depthFull, ivec2(gl_FragCoord.xy), 0).rrrr;

	depths[0] = get_view_z_from_depth(depths[0]);
	depths[1] = get_view_z_from_depth(depths[1]);
	depths[2] = get_view_z_from_depth(depths[2]);
	depths[3] = get_view_z_from_depth(depths[3]);

	target_depth = get_view_z_from_depth(target_depth);

	vec4 weights[4];
	weights[0] = 1.0 - step(0.05, abs(depths[0] - target_depth));
	weights[1] = 1.0 - step(0.05, abs(depths[1] - target_depth));
	weights[2] = 1.0 - step(0.05, abs(depths[2] - target_depth));
	weights[3] = 1.0 - step(0.05, abs(depths[3] - target_depth));

	float weight_sum;
	weight_sum  = dot(weights[0], vec4(1.0));
	weight_sum += dot(weights[1], vec4(1.0));
	weight_sum += dot(weights[2], vec4(1.0));
	weight_sum += dot(weights[3], vec4(1.0));

	if (weight_sum == 0.0) {
		weights[0].x = 1.0;
		weight_sum = 1.0;
	}

	texel_co = ivec2(gl_FragCoord.xy * 0.5);

	vec4 integration_result;

	texel_co += ivec2(-1, -1);
	integration_result  = texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 0)) * weights[0].x;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 0)) * weights[0].y;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 1)) * weights[0].z;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 1)) * weights[0].w;

	texel_co += ivec2(2, 0);
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 0)) * weights[1].x;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 0)) * weights[1].y;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 1)) * weights[1].z;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 1)) * weights[1].w;

	texel_co += ivec2(-2, 2);
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 0)) * weights[2].x;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 0)) * weights[2].y;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 1)) * weights[2].z;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 1)) * weights[2].w;

	texel_co += ivec2(2, 0);
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 0)) * weights[3].x;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 0)) * weights[3].y;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 1)) * weights[3].z;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 1)) * weights[3].w;
#endif

	FragColor = integration_result / weight_sum;
}
```
>
- 上一步的 stl->g_data->volumetric RT 会传进来作为 volumetricBuffer。
- 简单理解的就是，一个pixle 和周围的 pixel 的 scattering 和 transmittance 进行加权平均。
- transmittance 作为 alpha， 注意 volumetric_resolve_ps 在创建的时候，使用了 DRW_STATE_TRANSMISSION，处理的方式就是 glBlendFunc(GL_ONE, GL_SRC_ALPHA); 在 draw_manager.c DRW_state_set 函数中会进行判断和设置渲染状态。