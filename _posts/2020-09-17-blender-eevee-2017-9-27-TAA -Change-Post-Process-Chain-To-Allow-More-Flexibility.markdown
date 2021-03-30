---
layout:     post
title:      "blender eevee TAA Change post process chain to allow more flexibility"
subtitle:   ""
date:       2021-3-29 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/9/27  *   Eevee : TAA : Change post process chain to allow more flexibility<br> 

> This basically do not use hardware blending and do the blending in the shader.
This will allow neighborhood clamping if we ever implement that.


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 

## 编译
定位之后，想要编译成功，还要进行一下的修改
> source/blender/editors/sculpt_paint/paint_vertex.c, 
- line 1734 : BKE_sculpt_update_mesh_elements(scene, scene->toolsettings->sculpt, ob, 0, false, false);
- line 2245 : wpd->vp_handle = ED_vpaint_proj_handle_create(C, scene, ob, &wpd->vertexcosnos);
- line 3267 : vpd->vp_handle = ED_vpaint_proj_handle_create(C, scene, ob, &vpd->vertexcosnos);
- line 3937 : create_vgroups_from_armature(op->reports, C, scene, ob, armob, type, (me->editflag & ME_EDIT_MIRROR_X));
- line 4163 : DerivedMesh *dm = mesh_get_derived_final(C, scene, ob, scene->customdata_mask);


## alpha hashed 
为了测试方便，目前修改了 gpu_shader_material.glsl 中的 node_bsdf_transparent 的代码，为了是的 node transparent 的 alpha hashed 可以正常显示。
> source/blender/gpu/shaders/gpu_shader_material.glsl
```
void node_bsdf_transparent(vec4 color, out Closure result)
{
	/* this isn't right */
	//result.radiance = vec3(0.0);
	//result.opacity = 0.0;
	result.radiance = color.rgb;
	result.opacity = color.a;
#ifdef EEVEE_ENGINE
	result.ssr_id = TRANSPARENT_CLOSURE_FLAG;
#endif
}
```


## 作用
初始版本的TAA效果


## 准备
先看 [这里](http://shaderstore.cn/2021/03/29/blender-eevee-2017-9-26-Implement-Temporal-Anti-Aliasing-Super-Sampling/)


## 效果
*没有开启TAA的 Alpha Hashed 情况*
![](/img/Eevee/TAA/01/1.png)
<br>
*开启TAA的 Alpha Hashed 情况*
![](/img/Eevee/TAA/01/2.png)
<br>


### Shader

```

uniform sampler2D colorBuffer;
uniform sampler2D historyBuffer;
uniform float alpha;

out vec4 FragColor;

void main()
{
	/* TODO History buffer Reprojection */
	vec4 history = texelFetch(historyBuffer, ivec2(gl_FragCoord.xy), 0).rgba;
	vec4 color = texelFetch(colorBuffer, ivec2(gl_FragCoord.xy), 0).rgba;
	FragColor = mix(history, color, alpha);
}
```
>
- 这里跟之前最主要的区别就是多了 historyBuffer，下面就看看这个 historyBuffer是怎么来的


<br><br>


### 渲染流程 + Shader 参数

*eevee_effects.c*
```
void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	if ((effects->enabled_effects & EFFECT_TAA) != 0) {
		psl->taa_resolve = DRW_pass_create("Temporal AA Resolve", DRW_STATE_WRITE_COLOR);
		DRWShadingGroup *grp = DRW_shgroup_create(e_data.taa_resolve_sh, psl->taa_resolve);

		DRW_shgroup_uniform_buffer(grp, "historyBuffer", &txl->color_double_buffer);
		DRW_shgroup_uniform_buffer(grp, "colorBuffer", &txl->color);
		DRW_shgroup_uniform_float(grp, "alpha", &effects->taa_alpha, 1);
		DRW_shgroup_call_add(grp, quad, NULL);
	}
	...
}


void EEVEE_draw_effects(EEVEE_Data *vedata)
{
	...
	/* Temporal Anti-Aliasing */
	/* MUST COME FIRST. */
	if ((effects->enabled_effects & EFFECT_TAA) != 0) {
		if (effects->taa_current_sample != 1) {
			DRW_framebuffer_bind(fbl->effect_fb);
			DRW_draw_pass(psl->taa_resolve);

			/* Restore the depth from sample 1. */
			DRW_framebuffer_blit(fbl->depth_double_buffer_fb, fbl->main, true);

			/* Special Swap */
			SWAP(struct GPUFrameBuffer *, fbl->effect_fb, fbl->double_buffer);
			SWAP(GPUTexture *, txl->color_post, txl->color_double_buffer);
			swap_double_buffer = false;
			effects->source_buffer = txl->color_double_buffer;
			effects->target_buffer = fbl->main;
		}
		else {
			/* Save the depth buffer for the next frame.
			 * This saves us from doing anything special
			 * in the other mode engines. */
			DRW_framebuffer_blit(fbl->main, fbl->depth_double_buffer_fb, true);
		}

		if ((effects->taa_total_sample == 0) ||
		    (effects->taa_current_sample < effects->taa_total_sample))
		{
			DRW_viewport_request_redraw();
		}
	}
	...
}
```
>
- 上面看到 Shader 中的 historyBuffer 参数是由 txl->color_double_buffer 提供, txl->color_double_buffer 保存的是 上一帧的 画面
- 在TAA渲染的时候，可以看到 先绑定 fbl->effect_fb framebuffer，然后再 进行 DRW_draw_pass(psl->taa_resolve), 那么TAA就先把东西渲染到 fbl->effect_fb framebuffer 上。
-  接着 SWAP(struct GPUFrameBuffer *, fbl->effect_fb, fbl->double_buffer), 意思就是 fbl->effect_fb 和 fbl->double_buffer 交换，试得 fbl->effect_fb 指向一个新的 framebuffer, fbl->double_buffer 指向了 TAA渲染后到 framebuffer
-  接着 effects->source_buffer = txl->color_double_buffer; 和 effects->target_buffer = fbl->main; 意思就是 接下的后处理 输入都是  TAA渲染后到 framebuffer，输出是 fbl->main

<br><br>

### 计算 Shader 参数
*eevee_effects.c*
```
/* translate a matrix created by orthographic_m4 or perspective_m4 in XY coords (used to jitter the view) */
void window_translate_m4(float winmat[4][4], float perspmat[4][4], const float x, const float y)
{
	if (winmat[2][3] == -1.0f) {
		/* in the case of a win-matrix, this means perspective always */
		float v1[3];
		float v2[3];
		float len1, len2;

		v1[0] = perspmat[0][0];
		v1[1] = perspmat[1][0];
		v1[2] = perspmat[2][0];

		v2[0] = perspmat[0][1];
		v2[1] = perspmat[1][1];
		v2[2] = perspmat[2][1];

		len1 = (1.0f / len_v3(v1));
		len2 = (1.0f / len_v3(v2));

		float test1 = len1 * winmat[0][0];
		float test2 = len2 * winmat[1][1];

		// 这里测试过，不会进去，也就是test1 和 test2 基本都是 1
		if (test1 < 0.99 || test2 < 0.99 || test1 > 1.001 || test2 > 1.001) {
			test1 = 2;
		}

		winmat[2][0] += len1 * winmat[0][0] * x;
		winmat[2][1] += len2 * winmat[1][1] * y;

		// 可以写成
		//winmat[2][0] += x;
		//winmat[2][1] += y;
	}
	else {
		winmat[3][0] += x;
		winmat[3][1] += y;
	}
}

void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	if (BKE_collection_engine_property_value_get_bool(props, "taa_enable")) {
		float persmat[4][4], viewmat[4][4];

		enabled_effects |= EFFECT_TAA | EFFECT_DOUBLE_BUFFER;

		/* Until we support reprojection, we need to make sure
		 * that the history buffer contains correct information. */
		bool view_is_valid = stl->g_data->valid_double_buffer;

		view_is_valid = view_is_valid && (stl->g_data->view_updated == false);

		effects->taa_total_sample = BKE_collection_engine_property_value_get_int(props, "taa_samples");
		MAX2(effects->taa_total_sample, 0);

		// DRW_MAT_PERS 是 VP 矩阵
		DRW_viewport_matrix_get(persmat, DRW_MAT_PERS);
		// DRW_MAT_VIEW 是 V 矩阵
		DRW_viewport_matrix_get(viewmat, DRW_MAT_VIEW);
		// DRW_MAT_WIN 是 P 矩阵
		DRW_viewport_matrix_get(effects->overide_winmat, DRW_MAT_WIN);

		view_is_valid = view_is_valid && compare_m4m4(persmat, effects->prev_drw_persmat, 0.0001f);
		copy_m4_m4(effects->prev_drw_persmat, persmat);

		if (view_is_valid &&
		    ((effects->taa_total_sample == 0) ||
		     (effects->taa_current_sample < effects->taa_total_sample)))
		{
			effects->taa_current_sample += 1;

			effects->taa_alpha = 1.0f / (float)(effects->taa_current_sample);

			double ht_point[2];
			double ht_offset[2] = {0.0, 0.0};
			unsigned int ht_primes[2] = {2, 3};

			BLI_halton_2D(ht_primes, ht_offset, effects->taa_current_sample - 1, ht_point);

			float w = viewport_size[0];
			float h = viewport_size[1];

			window_translate_m4(
			        effects->overide_winmat, persmat,
			        ((float)(ht_point[0]) * 2.0f - 1.0f) / viewport_size[0],
			        ((float)(ht_point[1]) * 2.0f - 1.0f) / viewport_size[1]);

			mul_m4_m4m4(effects->overide_persmat, effects->overide_winmat, viewmat);
			invert_m4_m4(effects->overide_persinv, effects->overide_persmat);
			invert_m4_m4(effects->overide_wininv, effects->overide_winmat);

			DRW_viewport_matrix_override_set(effects->overide_persmat, DRW_MAT_PERS);
			DRW_viewport_matrix_override_set(effects->overide_persinv, DRW_MAT_PERSINV);
			DRW_viewport_matrix_override_set(effects->overide_winmat, DRW_MAT_WIN);
			DRW_viewport_matrix_override_set(effects->overide_wininv, DRW_MAT_WININV);
		}
		else {
			effects->taa_current_sample = 1;
		}

		DRWFboTexture tex_double_buffer = {&txl->depth_double_buffer, DRW_TEX_DEPTH_24};

		DRW_framebuffer_init(&fbl->depth_double_buffer_fb, &draw_engine_eevee_type,
		                    (int)viewport_size[0], (int)viewport_size[1],
		                    &tex_double_buffer, 1);
	}
	else {
		/* Cleanup to release memory */
		DRW_TEXTURE_FREE_SAFE(txl->depth_double_buffer);
		DRW_FRAMEBUFFER_FREE_SAFE(fbl->depth_double_buffer_fb);
	}
	...
}
```
>
- 这里可以看到 effects->taa_alpha  是如何计算的 1.0f / (float)(effects->taa_current_sample)
<br><br>
-  effects->overide_winmat 是 修改后的 投影矩阵，effects->overide_winmat 是通过 修改的 P 矩阵，主要是 让 点 和 effects->overide_winmat 相乘之后，x,y 坐标 会有一个多一个 jitter，也就是抖动偏移
<br><br>
- modify the projection matrix, adding small translations in x and y
<br><br>
- 看看 window_translate_m4 的注释，经过测试，其实 可以写成 winmat[2][0] += x, winmat[2][1] += y, 直接 让 点 乘以 winmat 之后，进行一个jitter
<br><br>
- 修改完 effects->overide_winmat 之后，也就是 P 矩阵，然后就利用 effects->overide_winmat 来修改 effects->overide_persmat，也就是修改 VP 矩阵
- 还有 effects->overide_persinv VP 的逆矩阵 和  effects->overide_wininv P 的逆矩阵，都是经过修改的





### 渲染流程的一些改变

```
static void EEVEE_engine_init(void *ved)
{
    ...
    /* EEVEE_effects_init needs to go first for TAA */
	EEVEE_effects_init(sldata, vedata);

	EEVEE_materials_init(stl);
	EEVEE_lights_init(sldata);
	EEVEE_lightprobes_init(sldata, vedata);

	if (stl->effects->taa_current_sample > 1) {
		/* XXX otherwise it would break the other engines. */
		DRW_viewport_matrix_override_unset(DRW_MAT_PERS);
		DRW_viewport_matrix_override_unset(DRW_MAT_PERSINV);
		DRW_viewport_matrix_override_unset(DRW_MAT_WIN);
		DRW_viewport_matrix_override_unset(DRW_MAT_WININV);
	}
    ...
}
```
>
- 初始化 添加taa_current_sample的判断，设置 DRW_viewport_matrix_override_unset

<br><br>

```

/* TODO : put this somewhere in BLI */
static float halton_1D(int prime, int n)
{
	float inv_prime = 1.0f / (float)prime;
	float f = 1.0f;
	float r = 0.0f;

	while (n > 0) {
		f = f * inv_prime;
		r += f * (n % prime);
		n = (int)(n * inv_prime);
	}

	return r;
}

static void EEVEE_draw_scene(void *vedata)
{
	...

	/* Number of iteration: needed for all temporal effect (SSR, TAA)
	 * when using opengl render. */
	int loop_ct = DRW_state_is_image_render() ? 4 : 1;

	static float rand = 0.0f;

	/* XXX temp for denoising render. TODO plug number of samples here */
	if (DRW_state_is_image_render()) {
		rand += 1.0f / 16.0f;
		rand = rand - floorf(rand);

		/* Set jitter offset */
		EEVEE_update_util_texture(rand);
	}
	else if (((stl->effects->enabled_effects & EFFECT_TAA) != 0) && (stl->effects->taa_current_sample > 1)) {
		rand = halton_1D(2, stl->effects->taa_current_sample - 1);

		/* Set jitter offset */
		/* PERF This is killing perf ! */
		EEVEE_update_util_texture(rand);
	}

	while (loop_ct--) {

		/* Refresh Probes */
		...

		/* Refresh shadows */
		...

		/* Attach depth to the hdr buffer and bind it */
		DRW_framebuffer_texture_detach(dtxl->depth);
		DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0, 0);
		DRW_framebuffer_bind(fbl->main);
		DRW_framebuffer_clear(false, true, false, NULL, 1.0f);

		if (((stl->effects->enabled_effects & EFFECT_TAA) != 0) && stl->effects->taa_current_sample > 1) {
			DRW_viewport_matrix_override_set(stl->effects->overide_persmat, DRW_MAT_PERS);
			DRW_viewport_matrix_override_set(stl->effects->overide_persinv, DRW_MAT_PERSINV);
			DRW_viewport_matrix_override_set(stl->effects->overide_winmat, DRW_MAT_WIN);
			DRW_viewport_matrix_override_set(stl->effects->overide_wininv, DRW_MAT_WININV);
		}

		/* Depth prepass */
		...

		/* Create minmax texture */
		...

		/* Compute GTAO Horizons */
		...

		/* Restore main FB */
		...

		/* Shading pass */
		...

		/* Screen Space Reflections */
		...

		DRW_draw_pass(psl->probe_display);

		/* Prepare Refraction */
		...

		/* Restore main FB */
		...

		/* Opaque refraction */
		...

		/* Transparent */
		...

		/* Volumetrics */
		...

		/* Post Process */
		...

		if (stl->effects->taa_current_sample > 1) {
			DRW_viewport_matrix_override_unset(DRW_MAT_PERS);
			DRW_viewport_matrix_override_unset(DRW_MAT_PERSINV);
			DRW_viewport_matrix_override_unset(DRW_MAT_WIN);
			DRW_viewport_matrix_override_unset(DRW_MAT_WININV);
		}
	}

	stl->g_data->view_updated = false;
}
```
>
- 渲染流程 考虑到了 overide_persmat ，overide_persinv ，overide_winmat，overide_wininv
- 个人理解 这些 overide_persmat 矩阵，就是替换了 原来的 persmat 矩阵，例如Shader 用到的矩阵 persmat 矩阵都会被这些 overide_persmat 给替换了。