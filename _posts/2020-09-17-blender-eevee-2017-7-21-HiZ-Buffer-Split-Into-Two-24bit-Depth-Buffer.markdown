---
layout:     post
title:      "blender eevee HiZ buffer Split into two 24bit depth buffer"
subtitle:   ""
date:       2021-3-8 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/21  *   Eevee: HiZ buffer: Split into two 24bit depth buffer .<br> 

> This way we don't have float precision issue we had before and we save some bandwidth.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51


## 作用
以前的minMaxDepthTex的r记录的是 depth的min值，g保存的是depth的max值，现在单独分开两张RT单独保存 dpth的min和max


## 编译
- 重新生成SLN
- git 定位到  2017/7/21  * Eevee: HiZ buffer: Split into two 24bit depth buffer.<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## 渲染
*eevee_engine.c*

```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Create minmax texture */
	EEVEE_create_minmax_buffer(vedata, dtxl->depth);
	...
}
```
>
- 优化 minmax depth 的存储方式

## EEVEE_create_minmax_buffer

*eevee_effects.c* 
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	e_data.minz_downlevel_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, "#define MIN_PASS\n");
	e_data.maxz_downlevel_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, "#define MAX_PASS\n");
	e_data.minz_downdepth_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, "#define MIN_PASS\n"
		                                                                                          "#define INPUT_DEPTH\n");
		e_data.maxz_downdepth_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, "#define MAX_PASS\n"
		                                                                                          "#define INPUT_DEPTH\n");
	e_data.minz_copydepth_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, "#define MIN_PASS\n"
																								"#define INPUT_DEPTH\n"
																								"#define COPY_DEPTH\n");
	e_data.maxz_copydepth_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, "#define MAX_PASS\n"
																								"#define INPUT_DEPTH\n"
																								"#define COPY_DEPTH\n");
																								
	...
	/* MinMax Pyramid */
	DRWFboTexture texmin = {&stl->g_data->minzbuffer, DRW_TEX_DEPTH_24, DRW_TEX_MIPMAP | DRW_TEX_TEMP};

	DRW_framebuffer_init(&fbl->downsample_fb, &draw_engine_eevee_type,
	                    (int)viewport_size[0] / 2, (int)viewport_size[1] / 2,
	                    &texmin, 1);

	/* Cannot define 2 depth texture for one framebuffer. So allocate ourself. */
	if (txl->maxzbuffer == NULL) {
		txl->maxzbuffer = DRW_texture_create_2D((int)viewport_size[0] / 2, (int)viewport_size[1] / 2, DRW_TEX_DEPTH_24, DRW_TEX_MIPMAP, NULL);
	}
	...
}
```

<br><br>

```
void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	{
		/* Perform min/max downsample */
		psl->minz_downlevel_ps = DRW_pass_create("HiZ Min Down Level", DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_ALWAYS);
		DRWShadingGroup *grp = DRW_shgroup_create(e_data.minz_downlevel_sh, psl->minz_downlevel_ps);
		DRW_shgroup_uniform_buffer(grp, "depthBuffer", &stl->g_data->minzbuffer);
		DRW_shgroup_call_add(grp, quad, NULL);

		psl->maxz_downlevel_ps = DRW_pass_create("HiZ Max Down Level", DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_ALWAYS);
		grp = DRW_shgroup_create(e_data.maxz_downlevel_sh, psl->maxz_downlevel_ps);
		DRW_shgroup_uniform_buffer(grp, "depthBuffer", &txl->maxzbuffer);
		DRW_shgroup_call_add(grp, quad, NULL);

		/* Copy depth buffer to halfres top level of HiZ */
		psl->minz_downdepth_ps = DRW_pass_create("HiZ Min Copy Depth Halfres", DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_ALWAYS);
		grp = DRW_shgroup_create(e_data.minz_downdepth_sh, psl->minz_downdepth_ps);
		DRW_shgroup_uniform_buffer(grp, "depthBuffer", &e_data.depth_src);
		DRW_shgroup_call_add(grp, quad, NULL);

		psl->maxz_downdepth_ps = DRW_pass_create("HiZ Max Copy Depth Halfres", DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_ALWAYS);
		grp = DRW_shgroup_create(e_data.maxz_downdepth_sh, psl->maxz_downdepth_ps);
		DRW_shgroup_uniform_buffer(grp, "depthBuffer", &e_data.depth_src);
		DRW_shgroup_call_add(grp, quad, NULL);

		/* Copy depth buffer to halfres top level of HiZ */
		psl->minz_copydepth_ps = DRW_pass_create("HiZ Min Copy Depth Fullres", DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_ALWAYS);
		grp = DRW_shgroup_create(e_data.minz_copydepth_sh, psl->minz_copydepth_ps);
		DRW_shgroup_uniform_buffer(grp, "depthBuffer", &e_data.depth_src);
		DRW_shgroup_call_add(grp, quad, NULL);

		psl->maxz_copydepth_ps = DRW_pass_create("HiZ Max Copy Depth Fullres", DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_ALWAYS);
		grp = DRW_shgroup_create(e_data.maxz_copydepth_sh, psl->maxz_copydepth_ps);
		DRW_shgroup_uniform_buffer(grp, "depthBuffer", &e_data.depth_src);
		DRW_shgroup_call_add(grp, quad, NULL);
	}
	...
}
```

<br><br>

```
static void min_downsample_cb(void *vedata, int UNUSED(level))
{
	EEVEE_PassList *psl = ((EEVEE_Data *)vedata)->psl;
	DRW_draw_pass(psl->minz_downlevel_ps);
}

static void max_downsample_cb(void *vedata, int UNUSED(level))
{
	EEVEE_PassList *psl = ((EEVEE_Data *)vedata)->psl;
	DRW_draw_pass(psl->maxz_downlevel_ps);
}

void EEVEE_create_minmax_buffer(EEVEE_Data *vedata, GPUTexture *depth_src)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_TextureList *txl = vedata->txl;

	e_data.depth_src = depth_src;

	/* Copy depth buffer to min texture top level */
	DRW_framebuffer_texture_attach(fbl->downsample_fb, stl->g_data->minzbuffer, 0, 0);
	DRW_framebuffer_bind(fbl->downsample_fb);
	DRW_draw_pass(psl->minz_downdepth_ps);
	DRW_framebuffer_texture_detach(stl->g_data->minzbuffer);

	/* Create lower levels */
	DRW_framebuffer_recursive_downsample(fbl->downsample_fb, stl->g_data->minzbuffer, 8, &min_downsample_cb, vedata);

	/* Copy depth buffer to max texture top level */
	DRW_framebuffer_texture_attach(fbl->downsample_fb, txl->maxzbuffer, 0, 0);
	DRW_framebuffer_bind(fbl->downsample_fb);
	DRW_draw_pass(psl->maxz_downdepth_ps);
	DRW_framebuffer_texture_detach(txl->maxzbuffer);

	/* Create lower levels */
	DRW_framebuffer_recursive_downsample(fbl->downsample_fb, txl->maxzbuffer, 8, &max_downsample_cb, vedata);
}
```
>
- 利用 psl->minz_downdepth_ps 把东西渲染到 stl->g_data->minzbuffer RT 上, psl->minz_downdepth_ps 使用 effect_minmaxz_frag.glsl , #define MIN_PASS 和 #define INPUT_DEPTH , 这个是拿 e_data.depth_src 进行downdepth。
<br><br>
- stl->g_data->minzbuffer RT 利用 psl->minz_downlevel_ps 进行 构造 mipmap , psl->minz_downlevel_ps 使用 effect_minmaxz_frag.glsl, 定义了 #define MIN_PASS, 这个是拿 stl->g_data->minzbuffer 进行downdepth。
<br><br>
- 利用 psl->maxz_downdepth_ps 把东西渲染到 txl->maxzbuffer RT 上, psl->maxz_downdepth_ps 使用 effect_minmaxz_frag.glsl，定义了 #define MAX_PASS 和 #define INPUT_DEPTH , 这个是拿 e_data.depth_src 进行downdepth。
<br><br>
- txl->maxzbuffer RT 利用 psl->maxz_downlevel_ps 进行构造mipmap, psl->maxz_downlevel_ps 使用 effect_minmaxz_frag.glsl，定义了 #define MAX_PASS , 这个是拿 stl->g_data->maxzbuffer 进行downdepth。
<br><br>
- 总的来说就是，minzbuffer RT 先进行 downdepth，再进行dowlevel, maxzbuffer RT 也是一样的道理。

<br><br>

## Shader

*effect_minmaxz_frag.glsl*
```
/**
 * Shader that downsample depth buffer,
 * saving min and max value of each texel in the above mipmaps.
 * Adapted from http://rastergrid.com/blog/2010/10/hierarchical-z-map-based-occlusion-culling/
 **/

uniform sampler2D depthBuffer;

float sampleLowerMip(ivec2 texel)
{
	return texelFetch(depthBuffer, texel, 0).r;
}

void minmax(inout float out_val, float in_val)
{
#ifdef MIN_PASS
	out_val = min(out_val, in_val);
#else /* MAX_PASS */
	out_val = max(out_val, in_val);
#endif
}

void main()
{
	ivec2 texelPos = ivec2(gl_FragCoord.xy);
	ivec2 mipsize = textureSize(depthBuffer, 0);

#ifndef COPY_DEPTH
	texelPos *= 2;
#endif

	float val = sampleLowerMip(texelPos);
#ifndef COPY_DEPTH
	minmax(val, sampleLowerMip(texelPos + ivec2(1, 0)));
	minmax(val, sampleLowerMip(texelPos + ivec2(1, 1)));
	minmax(val, sampleLowerMip(texelPos + ivec2(0, 1)));

	/* if we are reducing an odd-width texture then fetch the edge texels */
	if (((mipsize.x & 1) != 0) && (int(gl_FragCoord.x) == mipsize.x-3)) {
		/* if both edges are odd, fetch the top-left corner texel */
		if (((mipsize.y & 1) != 0) && (int(gl_FragCoord.y) == mipsize.y-3)) {
			minmax(val, sampleLowerMip(texelPos + ivec2(-1, -1)));
		}
		minmax(val, sampleLowerMip(texelPos + ivec2(0, -1)));
		minmax(val, sampleLowerMip(texelPos + ivec2(1, -1)));
	}
	/* if we are reducing an odd-height texture then fetch the edge texels */
	else if (((mipsize.y & 1) != 0) && (int(gl_FragCoord.y) == mipsize.y-3)) {
		minmax(val, sampleLowerMip(texelPos + ivec2(0, -1)));
		minmax(val, sampleLowerMip(texelPos + ivec2(1, -1)));
	}
#endif

	gl_FragDepth = val;
}
```
>
- 上面的其实都比较好理解，minzbuffer RT 的一个像素保存就是右，下，右下，自身，4个深度值最小值，maxbuffer RT 的是右，下，右下，自身，4个深度值最大值，对于mipmap是按照同样的方式进行构造，只不过是宽高不一样而已。
<br><br>
- 这里跟之前有不太一样的就是，min和max depth 值分开了。

<br><br>

## 应用

*eevee_materials.c*
```
static void add_standard_uniforms(DRWShadingGroup *shgrp, EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata, int *ssr_id)
{
	if (ssr_id == NULL) {
		static int no_ssr = -1.0f;
		ssr_id = &no_ssr;
	}
	DRW_shgroup_uniform_block(shgrp, "probe_block", sldata->probe_ubo);
	DRW_shgroup_uniform_block(shgrp, "grid_block", sldata->grid_ubo);
	DRW_shgroup_uniform_block(shgrp, "planar_block", sldata->planar_ubo);
	DRW_shgroup_uniform_block(shgrp, "light_block", sldata->light_ubo);
	DRW_shgroup_uniform_block(shgrp, "shadow_block", sldata->shadow_ubo);
	DRW_shgroup_uniform_int(shgrp, "light_count", &sldata->lamps->num_light, 1);
	DRW_shgroup_uniform_int(shgrp, "probe_count", &sldata->probes->num_render_cube, 1);
	DRW_shgroup_uniform_int(shgrp, "grid_count", &sldata->probes->num_render_grid, 1);
	DRW_shgroup_uniform_int(shgrp, "planar_count", &sldata->probes->num_planar, 1);
	DRW_shgroup_uniform_bool(shgrp, "specToggle", &sldata->probes->specular_toggle, 1);
	DRW_shgroup_uniform_float(shgrp, "lodCubeMax", &sldata->probes->lod_cube_max, 1);
	DRW_shgroup_uniform_float(shgrp, "lodPlanarMax", &sldata->probes->lod_planar_max, 1);
	DRW_shgroup_uniform_texture(shgrp, "utilTex", e_data.util_tex);
	DRW_shgroup_uniform_buffer(shgrp, "probeCubes", &sldata->probe_pool);
	DRW_shgroup_uniform_buffer(shgrp, "probePlanars", &vedata->txl->planar_pool);
	DRW_shgroup_uniform_buffer(shgrp, "irradianceGrid", &sldata->irradiance_pool);
	DRW_shgroup_uniform_buffer(shgrp, "shadowCubes", &sldata->shadow_depth_cube_pool);
	DRW_shgroup_uniform_buffer(shgrp, "shadowCascades", &sldata->shadow_depth_cascade_pool);
	DRW_shgroup_uniform_int(shgrp, "outputSsrId", ssr_id, 1);
	if (vedata->stl->effects->use_ao) {
		DRW_shgroup_uniform_vec4(shgrp, "viewvecs[0]", (float *)&vedata->stl->g_data->viewvecs, 2);
		DRW_shgroup_uniform_buffer(shgrp, "minMaxDepthTex", &vedata->txl->maxzbuffer);
		DRW_shgroup_uniform_vec3(shgrp, "aoParameters", &vedata->stl->effects->ao_dist, 1);
	}
}
```
>
- DRW_shgroup_uniform_buffer(shgrp, "minMaxDepthTex", &vedata->txl->maxzbuffer); 主要是SSAO计算要用到，最值传入了单独的depth max RT

<br><br>

# 2. Make MinmaxZ compatible with textureArray


## 来源

- 主要看这个commit

> GIT : 2017/7/24  *   Eevee: Make MinmaxZ compatible with textureArray .<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51


## 作用
min max 可以处理 textureArray, 那数据的哪一个元素作为数据源进行采样。


## 编译
- 重新生成SLN
- git 定位到  2017/7/24  * Eevee: Make MinmaxZ compatible with textureArray.<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)



<br><br>

## 渲染前

*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	e_data.minz_downdepth_layer_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, "#define MIN_PASS\n"
																									"#define LAYERED\n"
																									"#define INPUT_DEPTH\n");
	e_data.maxz_downdepth_layer_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, "#define MAX_PASS\n"
																									"#define LAYERED\n"
																									"#define INPUT_DEPTH\n");
	...
}

void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	psl->minz_downdepth_layer_ps = DRW_pass_create("HiZ Min Copy DepthLayer Halfres", DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_ALWAYS);
	grp = DRW_shgroup_create(e_data.minz_downdepth_layer_sh, psl->minz_downdepth_layer_ps);
	DRW_shgroup_uniform_buffer(grp, "depthBuffer", &e_data.depth_src);
	DRW_shgroup_uniform_int(grp, "depthLayer", &e_data.depth_src_layer, 1);
	DRW_shgroup_call_add(grp, quad, NULL);

	psl->maxz_downdepth_layer_ps = DRW_pass_create("HiZ Max Copy DepthLayer Halfres", DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_ALWAYS);
	grp = DRW_shgroup_create(e_data.maxz_downdepth_layer_sh, psl->maxz_downdepth_layer_ps);
	DRW_shgroup_uniform_buffer(grp, "depthBuffer", &e_data.depth_src);
	DRW_shgroup_uniform_int(grp, "depthLayer", &e_data.depth_src_layer, 1);
	DRW_shgroup_call_add(grp, quad, NULL);
	...
}


void EEVEE_create_minmax_buffer(EEVEE_Data *vedata, GPUTexture *depth_src, int layer)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_TextureList *txl = vedata->txl;

	e_data.depth_src = depth_src;

	/* Copy depth buffer to min texture top level */
	DRW_framebuffer_texture_attach(fbl->downsample_fb, stl->g_data->minzbuffer, 0, 0);
	DRW_framebuffer_bind(fbl->downsample_fb);
	if (layer >= 0) {
		e_data.depth_src_layer = layer;
		DRW_draw_pass(psl->minz_downdepth_layer_ps);
	}
	else {
		DRW_draw_pass(psl->minz_downdepth_ps);
	}
	DRW_framebuffer_texture_detach(stl->g_data->minzbuffer);

	/* Create lower levels */
	DRW_framebuffer_recursive_downsample(fbl->downsample_fb, stl->g_data->minzbuffer, 8, &min_downsample_cb, vedata);

	/* Copy depth buffer to max texture top level */
	DRW_framebuffer_texture_attach(fbl->downsample_fb, txl->maxzbuffer, 0, 0);
	DRW_framebuffer_bind(fbl->downsample_fb);
	if (layer >= 0) {
		e_data.depth_src_layer = layer;
		DRW_draw_pass(psl->maxz_downdepth_layer_ps);
	}
	else {
		DRW_draw_pass(psl->maxz_downdepth_ps);
	}
	DRW_framebuffer_texture_detach(txl->maxzbuffer);

	/* Create lower levels */
	DRW_framebuffer_recursive_downsample(fbl->downsample_fb, txl->maxzbuffer, 8, &max_downsample_cb, vedata);
}
```
>
- EEVEE_create_minmax_buffer 的参数 layer 如果是大于0 的话，就表示要处理数组，那么渲染使用 maxz_downdepth_layer_ps, 那就是 effect_minmaxz_frag.glsl  和 开启 #define LAYERED
- EEVEE_create_minmax_buffer 的参数 layer 如果是小于0 的话，就不用处理数组，就不用开启 #define LAYERED

<br><br>



## Shader

*effect_minmaxz_frag.glsl*
```
/**
 * Shader that downsample depth buffer,
 * saving min and max value of each texel in the above mipmaps.
 * Adapted from http://rastergrid.com/blog/2010/10/hierarchical-z-map-based-occlusion-culling/
 **/

#ifdef LAYERED
uniform sampler2DArray depthBuffer;
uniform int depthLayer;
#else
uniform sampler2D depthBuffer;
#endif

float sampleLowerMip(ivec2 texel)
{
#ifdef LAYERED
	return texelFetch(depthBuffer, ivec3(texel, depthLayer), 0).r;
#else
	return texelFetch(depthBuffer, texel, 0).r;
#endif
}

void minmax(inout float out_val, float in_val)
{
#ifdef MIN_PASS
	out_val = min(out_val, in_val);
#else /* MAX_PASS */
	out_val = max(out_val, in_val);
#endif
}

void main()
{
	ivec2 texelPos = ivec2(gl_FragCoord.xy);
	ivec2 mipsize = textureSize(depthBuffer, 0).xy;

#ifndef COPY_DEPTH
	texelPos *= 2;
#endif

	float val = sampleLowerMip(texelPos);
#ifndef COPY_DEPTH
	minmax(val, sampleLowerMip(texelPos + ivec2(1, 0)));
	minmax(val, sampleLowerMip(texelPos + ivec2(1, 1)));
	minmax(val, sampleLowerMip(texelPos + ivec2(0, 1)));

	/* if we are reducing an odd-width texture then fetch the edge texels */
	if (((mipsize.x & 1) != 0) && (int(gl_FragCoord.x) == mipsize.x-3)) {
		/* if both edges are odd, fetch the top-left corner texel */
		if (((mipsize.y & 1) != 0) && (int(gl_FragCoord.y) == mipsize.y-3)) {
			minmax(val, sampleLowerMip(texelPos + ivec2(-1, -1)));
		}
		minmax(val, sampleLowerMip(texelPos + ivec2(0, -1)));
		minmax(val, sampleLowerMip(texelPos + ivec2(1, -1)));
	}
	/* if we are reducing an odd-height texture then fetch the edge texels */
	else if (((mipsize.y & 1) != 0) && (int(gl_FragCoord.y) == mipsize.y-3)) {
		minmax(val, sampleLowerMip(texelPos + ivec2(0, -1)));
		minmax(val, sampleLowerMip(texelPos + ivec2(1, -1)));
	}
#endif

	gl_FragDepth = val;
}
```
>
- 这里比较重要的是 LAYERED 宏，如果开启了 LAYERED 宏的话，那么处理的就是 sampler2DArray，采样的也是根据数组的第几个元素进行采样, texelFetch(depthBuffer, ivec3(texel, depthLayer), 0).r;
- depthLayer 就是决定数组的第几个元素。


## 应用

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Create minmax texture */
	EEVEE_create_minmax_buffer(vedata, dtxl->depth, -1);
	...
}
```
>
- 在主渲染管线上, EEVEE_create_minmax_buffer 的 layer 为 -1, 表示不使用 Array。

<br><br>

*eevee_lightprobes.c*
```
static void render_scene_to_planar(EEVEE_Data *vedata, int layer,
        float (*viewmat)[4], float (*persmat)[4],
        float clip_plane[4])
{
	...
	EEVEE_create_minmax_buffer(vedata, tmp_planar_depth, layer);
	...
}
```
>
- 在planar reflection 的渲染上，使用了 layer, 表示 使用 array