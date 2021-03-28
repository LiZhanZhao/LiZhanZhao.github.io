---
layout:     post
title:      "blender eevee Implement Temporal Anti Aliasing  Super Sampling"
subtitle:   ""
date:       2021-3-29 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/9/26  *   Eevee : Implement Temporal Anti Aliasing  Super Sampling.<br> 

> This adds TAA to eevee. The only thing important to note is that we need to keep the unjittered depth buffer so that the other engines are composited correctly.


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
实现 TAA 渲染流程，目前没有真正实现TAA

<br>

### Swap Double Buffers

#### 1. 初始化

*eevee_engine.c*

```
static void EEVEE_engine_init(void *ved)
{
    ...
    DRWFboTexture tex = {&txl->color, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER | DRW_TEX_MIPMAP};

	const float *viewport_size = DRW_viewport_size_get();
	DRW_framebuffer_init(&fbl->main, &draw_engine_eevee_type,
	                    (int)viewport_size[0], (int)viewport_size[1],
	                    &tex, 1);

	/* EEVEE_effects_init needs to go first for TAA */
	EEVEE_effects_init(sldata, vedata);
    ...

}

```

<br>

*eevee_effects.c*
```

void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
    ...
    /* Setup double buffer so we can access last frame as it was before post processes */
	if ((effects->enabled_effects & EFFECT_DOUBLE_BUFFER) != 0) {
		DRWFboTexture tex_double_buffer = {&txl->color_double_buffer, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER | DRW_TEX_MIPMAP};

		DRW_framebuffer_init(&fbl->double_buffer, &draw_engine_eevee_type,
		                    (int)viewport_size[0], (int)viewport_size[1],
		                    &tex_double_buffer, 1);
	}
	else {
		/* Cleanup to release memory */
		DRW_TEXTURE_FREE_SAFE(txl->color_double_buffer);
		DRW_FRAMEBUFFER_FREE_SAFE(fbl->double_buffer);
	}

    ...

    /* Only allocate if at least one effect is activated */
	if (effects->enabled_effects != 0) {
		/* Ping Pong buffer */
		DRWFboTexture tex = {&txl->color_post, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER};

		DRW_framebuffer_init(&fbl->effect_fb, &draw_engine_eevee_type,
		                    (int)viewport_size[0], (int)viewport_size[1],
		                    &tex, 1);
	}
}
```
>
- 上面可以看到 
<br>
- fbl->main 绑定了 txl->color
<br>
- fbl->double_buffer 绑定了 txl->color_double_buffer
<br>
- fbl->effect_fb buffer 绑定了 txl->color_post

<br>


#### 2. Swap

```
#define SWAP_DOUBLE_BUFFERS() {                                       \
	if (swap_double_buffer) {                                         \
		SWAP(struct GPUFrameBuffer *, fbl->main, fbl->double_buffer); \
		SWAP(GPUTexture *, txl->color, txl->color_double_buffer);     \
		swap_double_buffer = false;                                   \
	}                                                                 \
} ((void)0)

#define SWAP(type, a, b)  {    \
	type sw_ap;                \
	CHECK_TYPE(a, type);       \
	CHECK_TYPE(b, type);       \
	sw_ap = (a);               \
	(a) = (b);                 \
	(b) = sw_ap;               \
} (void)0


#define SWAP_BUFFERS() {                           \
	if (effects->source_buffer == txl->color) {    \
		SWAP_DOUBLE_BUFFERS();                     \
		effects->source_buffer = txl->color_post;  \
		effects->target_buffer = fbl->main;        \
	}                                              \
	else {                                         \
		SWAP_DOUBLE_BUFFERS();                     \
		effects->source_buffer = txl->color;       \
		effects->target_buffer = fbl->effect_fb;   \
	}                                              \
} ((void)0)

```
<br>

*eevee_effects.c*
```
void EEVEE_draw_effects(EEVEE_Data *vedata)
{
    ...
	/* only once per frame after the first post process */
	bool swap_double_buffer = ((effects->enabled_effects & EFFECT_DOUBLE_BUFFER) != 0);

	...

	/* Init pointers */
	effects->source_buffer = txl->color; /* latest updated texture */
	effects->target_buffer = fbl->effect_fb; /* next target to render to */

    ...

	/* Motion Blur */
	if ((effects->enabled_effects & EFFECT_MOTION_BLUR) != 0) {
		DRW_framebuffer_bind(effects->target_buffer);
		DRW_draw_pass(psl->motion_blur);
		SWAP_BUFFERS();
	}

	/* Detach depth for effects to use it */
	DRW_framebuffer_texture_detach(dtxl->depth);

	...

	/* Restore default framebuffer */
	DRW_framebuffer_texture_attach(dfbl->default_fb, dtxl->depth, 0, 0);
	DRW_framebuffer_bind(dfbl->default_fb);

	/* Tonemapping */
	DRW_transform_to_display(effects->source_buffer);

	...

	/* If no post processes is enabled, buffers are still not swapped, do it now. */
	SWAP_DOUBLE_BUFFERS();

	if (!stl->g_data->valid_double_buffer &&
		((effects->enabled_effects & EFFECT_DOUBLE_BUFFER) != 0) &&
		(DRW_state_is_image_render() == false))
	{
		/* If history buffer is not valid request another frame.
		 * This fix black reflections on area resize. */
		DRW_viewport_request_redraw();
	}

	/* Record pers matrix for the next frame. */
	DRW_viewport_matrix_get(stl->g_data->prev_persmat, DRW_MAT_PERS);

	/* Update double buffer status if render mode. */
	if (DRW_state_is_image_render()) {
		stl->g_data->valid_double_buffer = (txl->color_double_buffer != NULL);
	}
}
```
>
- EEVEE_draw_effects 函数一开始 之前，也就是进行后处理之前，所有的物体已经渲染到 fbl->main 中
<br><br>
- EEVEE_draw_effects 函数一开始 都会先判断是否使用 swap_double_buffer
<br>

> 执行 EEVEE_draw_effects 
- effects->source_buffer = txl->color (fbl->main)
<br><br>
- effects->target_buffer = fbl->effect_fb (txl->color_post)
<br><br>
- 假设 没有任何的 后处理需要执行的话，进行 DRW_transform_to_display(effects->source_buffer)，把effects->source_buffer进行输出到屏幕上
<br><br>
- 然后再进行 SWAP_DOUBLE_BUFFERS，需要注意的是，SWAP_DOUBLE_BUFFERS 如果之前执行过一次，就不会再执行，因为执行前要判断 swap_double_buffer 变量，也就是 EEVEE_draw_effects 只会执行一次 SWAP_DOUBLE_BUFFERS
<br><br>
- SWAP_DOUBLE_BUFFERS 的作用就是 fbl->main 和 fbl->double_buffer 进行交换，txl->color 和 txl->color_double_buffer 也进行交换，其实理解就是 fbl->main 指向两个不同的 framebuffer，每一次执行EEVEE_draw_effects 都让 fbl->main 指向不同的framebuffer，也就是下一帧渲染的时候，物体就渲染到 B FrameBuffer 上，再下一帧，就渲染回 A FrameBuffer 上.

<br>
> SWAP_BUFFERS
- 看看 Motion Blur 的逻辑，先把东西渲染到 target_buffer 上，再进行 SWAP_BUFFERS
<br><br>
- SWAP_BUFFERS 是先执行 SWAP_DOUBLE_BUFFERS ，然后再交换 effects->source_buffer 和 effects->target_buffer, 所以 执行完 SWAP_BUFFERS 之后，effects->source_buffer 就指向了 effects->target_buffer ，那么在最后 DRW_transform_to_display(effects->source_buffer) 的时候，就可以把Motion Blur 渲染的东西进行输出。
<br><br>
- 如果执行完 Motion Blur 之后，还要有其他的 后处理逻辑，其实也是拿 effects->source_buffer 作为输入，把东西渲染到 effects->target_buffer 上，然后再交换 effects->source_buffer 和 effects->target_buffer，再输出 effects->source_buffer，以此类推
<br><br> 
- 但是值得注意的是，SWAP_BUFFERS 中调用 SWAP_DOUBLE_BUFFERS ，SWAP_DOUBLE_BUFFERS只会执行一次

<br><br> 

### TAA 渲染入口
*eevee_engine.c*
```
void EEVEE_draw_effects(EEVEE_Data *vedata)
{
    ...
    // 对于TAA来说，如果打开了TAA，那么同时也会进行 EFFECT_DOUBLE_BUFFER
	/* only once per frame after the first post process */
	bool swap_double_buffer = ((effects->enabled_effects & EFFECT_DOUBLE_BUFFER) != 0);

	...

	/* Init pointers */
	effects->source_buffer = txl->color; /* latest updated texture */
	effects->target_buffer = fbl->effect_fb; /* next target to render to */

	/* Motion Blur */
	if ((effects->enabled_effects & EFFECT_TAA) != 0) {
		if (effects->taa_current_sample != 1) {
			effects->source_buffer = txl->color_double_buffer;
			DRW_framebuffer_bind(fbl->main);
			DRW_draw_pass(psl->taa_resolve);
			effects->source_buffer = txl->color;

			/* Restore the depth from sample 1. */
			DRW_framebuffer_blit(fbl->depth_double_buffer_fb, fbl->main, true);
		}
		else {
			/* Save the depth buffer for the next frame.
			 * This saves us from doing anything special
			 * in the other mode engines. */
			DRW_framebuffer_blit(fbl->main, fbl->depth_double_buffer_fb, true);
		}

		if (effects->taa_current_sample < effects->taa_total_sample) {
			DRW_viewport_request_redraw();
		}
	}

	/* Detach depth for effects to use it */
	DRW_framebuffer_texture_detach(dtxl->depth);

	...

	/* Restore default framebuffer */
	DRW_framebuffer_texture_attach(dfbl->default_fb, dtxl->depth, 0, 0);
	DRW_framebuffer_bind(dfbl->default_fb);

	/* Tonemapping */
	DRW_transform_to_display(effects->source_buffer);

	...

	/* If no post processes is enabled, buffers are still not swapped, do it now. */
	SWAP_DOUBLE_BUFFERS();

	if (!stl->g_data->valid_double_buffer &&
		((effects->enabled_effects & EFFECT_DOUBLE_BUFFER) != 0) &&
		(DRW_state_is_image_render() == false))
	{
		/* If history buffer is not valid request another frame.
		 * This fix black reflections on area resize. */
		DRW_viewport_request_redraw();
	}

	/* Record pers matrix for the next frame. */
	DRW_viewport_matrix_get(stl->g_data->prev_persmat, DRW_MAT_PERS);

	/* Update double buffer status if render mode. */
	if (DRW_state_is_image_render()) {
		stl->g_data->valid_double_buffer = (txl->color_double_buffer != NULL);
	}
}
```
>
- DRW_draw_pass(psl->taa_resolve); 进行TAA的渲染，渲染前绑定了 fbl->main，然后设置了 effects->source_buffer = txl->color_double_buffer;，这个主要是因为 taa_resolve 传入要把 effects->source_buffer传入到Shader中，color_double_buffer 其实也可以考虑为保存了上一帧的渲染结果。
<br><br>
- 执行后 DRW_draw_pass(psl->taa_resolve);之后还会进行，effects->source_buffer = txl->color; 还原 effects->source_buffer 的值，因为下面的后处理效果可能需要的不是上一帧的结果

<br><br>

### taa_resolve Pass

#### 1. 初始化

```

void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
    ...
    e_data.taa_resolve_sh = DRW_shader_create_fullscreen(datatoc_effect_temporal_aa_glsl, NULL);
    ...
}

void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
    ...
    if ((effects->enabled_effects & EFFECT_TAA) != 0) {
		psl->taa_resolve = DRW_pass_create("Temporal AA Resolve", DRW_STATE_WRITE_COLOR | DRW_STATE_BLEND);
		DRWShadingGroup *grp = DRW_shgroup_create(e_data.taa_resolve_sh, psl->taa_resolve);

		DRW_shgroup_uniform_buffer(grp, "colorBuffer", &effects->source_buffer);
		DRW_shgroup_uniform_float(grp, "alpha", &effects->taa_alpha, 1);
		DRW_shgroup_call_add(grp, quad, NULL);
	}
    ...
}
```
>
- taa_resolve Pass 使用了 effect_temporal_aa.glsl
- Shader 参数 colorBuffer 是传入 effects->source_buffer, 

<br>

#### 2. Shader
*effect_temporal_aa.glsl*
```
uniform sampler2D colorBuffer;
uniform float alpha;

out vec4 FragColor;

void main()
{
	/* TODO History buffer Reprojection */
	FragColor = vec4(texelFetch(colorBuffer, ivec2(gl_FragCoord.xy), 0).rgb, alpha);
}
```
>
- Shader目前比较简单，直接采样 传入进来的 colorBuffer，再进行输出

<br>

#### 3. Shader 参数
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
    ...
    if (BKE_collection_engine_property_value_get_bool(props, "taa_enable")) {
		float persmat[4][4], viewmat[4][4];

		effects->enabled_effects |= EFFECT_TAA | EFFECT_DOUBLE_BUFFER;

		/* Until we support reprojection, we need to make sure
		 * that the history buffer contains correct information. */
		bool view_is_valid = stl->g_data->valid_double_buffer;

		view_is_valid = view_is_valid && (stl->g_data->view_updated == false);

		int taa_pref_samples = BKE_collection_engine_property_value_get_int(props, "taa_samples");
		CLAMP(taa_pref_samples, 1, 32);
		view_is_valid = view_is_valid && (effects->taa_total_sample == taa_pref_samples);

		if (effects->taa_total_sample != taa_pref_samples) {
			effects->taa_total_sample = taa_pref_samples;
			BLI_jitter_init(effects->taa_jit_ofs, effects->taa_total_sample);
		}

		DRW_viewport_matrix_get(persmat, DRW_MAT_PERS);
		DRW_viewport_matrix_get(viewmat, DRW_MAT_VIEW);
		DRW_viewport_matrix_get(effects->overide_winmat, DRW_MAT_WIN);
		view_is_valid = view_is_valid && compare_m4m4(persmat, effects->prev_drw_persmat, 0.0001f);
		copy_m4_m4(effects->prev_drw_persmat, persmat);


		if (view_is_valid && (effects->taa_current_sample < effects->taa_total_sample)) {
			effects->taa_current_sample += 1;

			effects->taa_alpha = 1.0f - (1.0f / (float)(effects->taa_current_sample));

			window_translate_m4(
			        effects->overide_winmat, persmat,
			        (effects->taa_jit_ofs[effects->taa_current_sample][0] * 2.0f) / viewport_size[0],
			        (effects->taa_jit_ofs[effects->taa_current_sample][1] * 2.0f) / viewport_size[1]);

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
- 打开TAA的时候，会自带 EFFECT_DOUBLE_BUFFER 的功能
<br><br>
- 这里是计算 effects->overide_persmat, effects->overide_persinv, effects->overide_winmat， effects->overide_wininv

<br><br>

#### 4. 渲染流程的一些改变

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