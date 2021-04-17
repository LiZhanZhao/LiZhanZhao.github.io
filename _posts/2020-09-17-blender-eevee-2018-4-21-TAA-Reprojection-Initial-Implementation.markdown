---
layout:     post
title:      "blender eevee TAA Reprojection Initial implementation"
subtitle:   ""
date:       2021-4-16 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2018/4/21  *  Eevee :TAA Reprojection Initial implementation. <br> 

> 
This "improve" the viewport experience by reducing the noise from random
sampling effects (SSAO, Contact Shadows, SSR) when moving the viewport or
during playback.

>
This does not do Anti Aliasing because this would conflict with the outline
pass. We could enable AA jittering in "only render" mode though.

>
There are many things to improve but this is a solid basis to build upon.


> SVN : 2018/3/18  vc14 libs: add missing package folder. 


<br><br>

## 作用
实现 TAA Reprojection

<br><br>



### 渲染前

#### 1. 初始化 EEVEE_temporal_sampling_init 

```
static void eevee_create_shader_temporal_sampling(void)
{
	char *frag_str = BLI_string_joinN(
	        datatoc_common_uniforms_lib_glsl,
	        datatoc_common_view_lib_glsl,
	        datatoc_bsdf_common_lib_glsl,
	        datatoc_effect_temporal_aa_glsl);

	e_data.taa_resolve_sh = DRW_shader_create_fullscreen(frag_str, NULL);
	e_data.taa_resolve_reproject_sh = DRW_shader_create_fullscreen(frag_str, "#define USE_REPROJECTION\n");

	MEM_freeN(frag_str);
}


static void eevee_create_cdf_table_temporal_sampling(void)
{
	float *cdf_table = MEM_mallocN(sizeof(float) * FILTER_CDF_TABLE_SIZE, "Eevee Filter CDF table");

	float filter_width = 2.0f; /* Use a 2 pixel footprint by default. */

	{
		/* Use blackman-harris filter. */
		filter_width *= 2.0f;
		compute_cdf(filter_blackman_harris, cdf_table);
	}

	invert_cdf(cdf_table, e_data.inverted_cdf);

	/* Scale and offset table. */
	for (int i = 0; i < FILTER_CDF_TABLE_SIZE; ++i) {
		e_data.inverted_cdf[i] = (e_data.inverted_cdf[i] - 0.5f) * filter_width;
	}

	MEM_freeN(cdf_table);
}

int EEVEE_temporal_sampling_init(EEVEE_ViewLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
    ...
    if (!e_data.taa_resolve_sh) {
		eevee_create_shader_temporal_sampling();
		eevee_create_cdf_table_temporal_sampling();
	}
    ...

    int repro_flag = 0;
	if (!DRW_state_is_image_render() &&
		BKE_collection_engine_property_value_get_bool(props, "taa_reprojection"))
	{
		repro_flag = EFFECT_TAA_REPROJECT | EFFECT_VELOCITY_BUFFER | EFFECT_DEPTH_DOUBLE_BUFFER | EFFECT_DOUBLE_BUFFER | EFFECT_POST_BUFFER;
		effects->taa_reproject_sample = ((effects->taa_reproject_sample + 1) % 16);
	}

    ...
}
```
>
- 如果打开了 taa_reprojection 选项的话，EFFECT_TAA_REPROJECT 就会在flag中

<br><br>


#### 2. EEVEE_temporal_sampling_cache_init
```
void EEVEE_temporal_sampling_cache_init(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & (EFFECT_TAA | EFFECT_TAA_REPROJECT)) != 0) {
		struct GPUShader *sh = (effects->enabled_effects & EFFECT_TAA_REPROJECT)
		                        ? e_data.taa_resolve_reproject_sh
		                        : e_data.taa_resolve_sh;

		psl->taa_resolve = DRW_pass_create("Temporal AA Resolve", DRW_STATE_WRITE_COLOR);
		DRWShadingGroup *grp = DRW_shgroup_create(sh, psl->taa_resolve);

		DRW_shgroup_uniform_texture_ref(grp, "colorHistoryBuffer", &txl->color_double_buffer);
		DRW_shgroup_uniform_texture_ref(grp, "colorBuffer", &txl->color);

		if (effects->enabled_effects & EFFECT_TAA_REPROJECT) {
			DefaultTextureList *dtxl = DRW_viewport_texture_list_get();
			DRW_shgroup_uniform_texture_ref(grp, "velocityBuffer", &effects->velocity_tx);
			DRW_shgroup_uniform_block(grp, "common_block", sldata->common_ubo);
		}
		else {
			DRW_shgroup_uniform_float(grp, "alpha", &effects->taa_alpha, 1);
		}
		DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL);
	}
}
```
>
- taa_resolve Pass 使用的Shader，会根据是否有 EFFECT_TAA_REPROJECT 来决定
<br><br>
- 使用是 定义了  EFFECT_TAA_REPROJECT ，也就是打开了 taa_reprojection 选项的话,使用的Shader就是 effect_temporal_aa.glsl, 并且定义了 宏 USE_REPROJECTION
<br><br>
- 如果没有打开 taa_reprojection的话，Shader就是 effect_temporal_aa.glsl, 没有定义了 宏 USE_REPROJECTION
<br><br>
- 如果定义了 EFFECT_TAA_REPROJECT 的话，需要 velocityBuffer RT

<br><br>


### 渲染

#### 1.EEVEE_draw_effects
*eevee_engine.c*
```
static void eevee_draw_background(void *vedata)
{
    ...
    if (((stl->effects->enabled_effects & EFFECT_TAA) != 0) &&
        (stl->effects->taa_current_sample > 1) &&
        !DRW_state_is_image_render() &&
        !taa_use_reprojection)
    {
        DRW_viewport_matrix_override_set(stl->effects->overide_persmat, DRW_MAT_PERS);
        DRW_viewport_matrix_override_set(stl->effects->overide_persinv, DRW_MAT_PERSINV);
        DRW_viewport_matrix_override_set(stl->effects->overide_winmat, DRW_MAT_WIN);
        DRW_viewport_matrix_override_set(stl->effects->overide_wininv, DRW_MAT_WININV);
    }

    ...

    /* Post Process */
    DRW_stats_group_start("Post FX");
    EEVEE_draw_effects(sldata, vedata);
    DRW_stats_group_end();

    ...
}
```
>
- 如果 不 使用 taa_use_reprojection 的情况下，才会去设置 overide_persmat + overide_persinv 矩阵 等

<br><br>

*eevee_effects.c*
```
void EEVEE_draw_effects(EEVEE_ViewLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
    ...
    /* Temporal Anti-Aliasing MUST come first */
	EEVEE_temporal_sampling_draw(vedata);
    ...
}
```
>
- EEVEE_draw_effects 调用 EEVEE_temporal_sampling_draw 进行  Temporal Anti-Aliasing

<br><br>

#### 2.EEVEE_temporal_sampling_draw

*eevee_temporal_sampling.c*
```
/* Special Swap */
#define SWAP_BUFFER_TAA() do { \
	SWAP(struct GPUFrameBuffer *, fbl->effect_fb, fbl->double_buffer_fb); \
	SWAP(struct GPUFrameBuffer *, fbl->effect_color_fb, fbl->double_buffer_color_fb); \
	SWAP(GPUTexture *, txl->color_post, txl->color_double_buffer); \
	effects->swap_double_buffer = false; \
	effects->source_buffer = txl->color_double_buffer; \
	effects->target_buffer = fbl->main_color_fb; \
} while (0);

void EEVEE_temporal_sampling_draw(EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & (EFFECT_TAA | EFFECT_TAA_REPROJECT)) != 0) {
		if ((effects->enabled_effects & EFFECT_TAA) != 0 && effects->taa_current_sample != 1) {
			if (DRW_state_is_image_render()) {
				/* See EEVEE_temporal_sampling_init() for more details. */
				effects->taa_alpha = 1.0f / (float)(effects->taa_render_sample);
			}
			else {
				effects->taa_alpha = 1.0f / (float)(effects->taa_current_sample);
			}

			GPU_framebuffer_bind(fbl->effect_color_fb);
			DRW_draw_pass(psl->taa_resolve);

			/* Restore the depth from sample 1. */
			if (!DRW_state_is_image_render()) {
				GPU_framebuffer_blit(fbl->double_buffer_depth_fb, 0, fbl->main_fb, 0, GPU_DEPTH_BIT);
			}

			SWAP_BUFFER_TAA();
		}
		else {
			if (!DRW_state_is_image_render()) {
				/* Do reprojection for noise reduction */
				/* TODO : do AA jitter if in only render view. */
				if ((effects->enabled_effects & EFFECT_TAA_REPROJECT) != 0 &&
				    stl->g_data->valid_double_buffer)
				{
					GPU_framebuffer_bind(fbl->effect_color_fb);
					DRW_draw_pass(psl->taa_resolve);

					SWAP_BUFFER_TAA();
				}

				/* Save the depth buffer for the next frame.
				 * This saves us from doing anything special
				 * in the other mode engines. */
				GPU_framebuffer_blit(fbl->main_fb, 0, fbl->double_buffer_depth_fb, 0, GPU_DEPTH_BIT);
			}
		}

		/* Make each loop count when doing a render. */
		if (DRW_state_is_image_render()) {
			effects->taa_render_sample += 1;
			effects->taa_current_sample += 1;
		}
		else {
			if ((effects->taa_total_sample == 0) ||
			    (effects->taa_current_sample < effects->taa_total_sample))
			{
				DRW_viewport_request_redraw();
			}
		}
	}
}
```
>
- 最主要的是，DRW_draw_pass(psl->taa_resolve) 把东西渲染到 effect_color_fb 上
<br><br>
- psl->taa_resolve 使用 effect_temporal_aa.glsl，如果是定义 EFFECT_TAA_REPROJECT 了的话，Shader 就定义 USE_REPROJECTION，否则的话，就没有定义宏 USE_REPROJECTION

<br><br>

#### 3. Shader
*effect_temporal_aa.glsl*
```
uniform sampler2D colorHistoryBuffer;
uniform sampler2D velocityBuffer;

out vec4 FragColor;

#ifdef USE_REPROJECTION

/**
 * Adapted from https://casual-effects.com/g3d/G3D10/data-files/shader/Film/Film_temporalAA.pix
 * which is adapted from https://github.com/gokselgoktas/temporal-anti-aliasing/blob/master/Assets/Resources/Shaders/TemporalAntiAliasing.cginc
 * which is adapted from https://github.com/playdeadgames/temporal
 * Optimization by Stubbesaurus and epsilon adjustment to avoid division by zero.
 *
 * This can cause 3x3 blocks of color when there is a thin edge of a similar color that
 * is varying in intensity.
 */
vec3 clip_to_aabb(vec3 color, vec3 minimum, vec3 maximum, vec3 average)
{
	/* note: only clips towards aabb center (but fast!) */
	vec3 center  = 0.5 * (maximum + minimum);
	vec3 extents = 0.5 * (maximum - minimum);
	vec3 dist = color - center;
	vec3 ts = abs(extents) / max(abs(dist), vec3(0.0001));
	float t = saturate(min_v3(ts));
	return center + dist * t;
}

/**
 * Vastly based on https://github.com/playdeadgames/temporal
 */
void main()
{
	ivec2 texel = ivec2(gl_FragCoord.xy);
	float depth = texelFetch(depthBuffer, texel, 0).r;
	vec2 motion = texelFetch(velocityBuffer, texel, 0).rg;

	/* Compute pixel position in previous frame. */
	vec2 screen_res = vec2(textureSize(colorBuffer, 0).xy);
	vec2 uv = gl_FragCoord.xy / screen_res;
	vec2 uv_history = uv - motion;

	ivec2 texel_history = ivec2(uv_history * screen_res);
	vec4 color_history = textureLod(colorHistoryBuffer, uv_history, 0.0);

	/* Color bounding box clamping. 3x3 neighborhood. */
	vec4 c02 = texelFetchOffset(colorBuffer, texel, 0, ivec2(-1,  1));
	vec4 c12 = texelFetchOffset(colorBuffer, texel, 0, ivec2( 0,  1));
	vec4 c22 = texelFetchOffset(colorBuffer, texel, 0, ivec2( 1,  1));
	vec4 c01 = texelFetchOffset(colorBuffer, texel, 0, ivec2(-1,  0));
	vec4 c11 = texelFetchOffset(colorBuffer, texel, 0, ivec2( 0,  0));
	vec4 c21 = texelFetchOffset(colorBuffer, texel, 0, ivec2( 1,  0));
	vec4 c00 = texelFetchOffset(colorBuffer, texel, 0, ivec2(-1, -1));
	vec4 c10 = texelFetchOffset(colorBuffer, texel, 0, ivec2( 0, -1));
	vec4 c20 = texelFetchOffset(colorBuffer, texel, 0, ivec2( 1, -1));

	vec4 color = c11;

	/* AABB minmax */
	vec4 min_col = min9(c02, c12, c22, c01, c11, c21, c00, c10, c20);
	vec4 max_col = max9(c02, c12, c22, c01, c11, c21, c00, c10, c20);
	vec4 avg_col = avg9(c02, c12, c22, c01, c11, c21, c00, c10, c20);

	/* bias the color aabb toward the center (rounding the shape) */
	vec4 min_center = min5(c12, c01, c11, c21, c10);
	vec4 max_center = max5(c12, c01, c11, c21, c10);
	vec4 avg_center = avg5(c12, c01, c11, c21, c10);
	min_col = (min_col + min_center) * 0.5;
	max_col = (max_col + max_center) * 0.5;
	avg_col = (avg_col + avg_center) * 0.5;

	/* Clip color toward the center of the neighborhood colors AABB box. */
	color_history.rgb = clip_to_aabb(color_history.rgb, min_col.rgb, max_col.rgb, avg_col.rgb);

	/* Luminance weighting. */
	/* TODO correct luminance */
	float lum0 = dot(color.rgb, vec3(0.333));
	float lum1 = dot(color_history.rgb, vec3(0.333));
	float diff = abs(lum0 - lum1) / max(lum0, max(lum1, 0.2));
	float weight = 1.0 - diff;
	float alpha = mix(0.04, 0.12, weight * weight);

	color_history = mix(color_history, color, alpha);

	bool out_of_view = any(greaterThanEqual(abs(uv_history - 0.5), vec2(0.5)));
	color_history = (out_of_view) ? color : color_history;

	FragColor = color_history;
}

#else

uniform float alpha;

void main()
{
	ivec2 texel = ivec2(gl_FragCoord.xy);
	vec4 color = texelFetch(colorBuffer, texel, 0);
	vec4 color_history = texelFetch(colorHistoryBuffer, texel, 0);
	FragColor = mix(color_history, color, alpha);
}
#endif
```
>
- 主要留意 USE_REPROJECTION 宏的那段，主要是用于 TAA Reprojection