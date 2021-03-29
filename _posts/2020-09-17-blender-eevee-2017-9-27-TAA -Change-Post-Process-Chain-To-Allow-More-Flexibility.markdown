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


```
if (BKE_collection_engine_property_value_get_bool(props, "taa_enable")) {
	float persmat[4][4], viewmat[4][4];

	enabled_effects |= EFFECT_TAA | EFFECT_DOUBLE_BUFFER;

	/* Until we support reprojection, we need to make sure
		* that the history buffer contains correct information. */
	bool view_is_valid = stl->g_data->valid_double_buffer;

	view_is_valid = view_is_valid && (stl->g_data->view_updated == false);

	effects->taa_total_sample = BKE_collection_engine_property_value_get_int(props, "taa_samples");
	MAX2(effects->taa_total_sample, 0);

	DRW_viewport_matrix_get(persmat, DRW_MAT_PERS);
	DRW_viewport_matrix_get(viewmat, DRW_MAT_VIEW);
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
```