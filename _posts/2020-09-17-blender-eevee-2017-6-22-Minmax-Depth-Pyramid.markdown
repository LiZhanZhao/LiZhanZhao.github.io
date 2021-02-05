---
layout:     post
title:      "blender eevee Minmax Depth Pyramid."
subtitle:   ""
date:       2021-2-3 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/22  * Eevee : Minmax Depth Pyramid. .<br> 

>
- This commit introduce the computation of a depth pyramid containing min and max depth values of the original depth buffer.
This is useful for Clustered Light Culling but also for raytracing on the depth buffer (SSR).
It's also usefull to have to fetch higher mips in order to improve texture cache usage.<br><br><br>
- As of now, 1st mip (highest res) is half the resolution of the depth buffer, but everything is already done to be able to make a fullres copy of the depth buffer in the 1st mip instead of downsampling.
Also, the texture used is RG_32F which is a too much but enough to cover the 24bits of the depth buffer. Reducing the texture size would make things quite faster.


> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		

## 作用 
*computation of a depth pyramid containing min and max depth values of the original depth buffer*<br><br>
个人理解就是计算 利用 original depth buffer 计算 一个 depth pyramid, depth pyramid 其实就是 mipmap 的 depth texture，texture 每一层level的每一个像素的r保存 min depth，g 保存的是 max depth，这些 min depth，max depth 是在 downsample 的时候，拿自身和(+1,0), (0,+1), (+1,+1)的像素进行对比出来的。


## 编译
- 重新生成SLN
- git 定位到  2017/6/22  * Eevee : Minmax Depth Pyramid .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## 渲染前
*eevee_effects.c*

```

void EEVEE_effects_init(EEVEE_Data *vedata)
{
	...
	e_data.minmaxz_downlevel_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, NULL);
	e_data.minmaxz_downdepth_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, "#define INPUT_DEPTH\n");
	e_data.minmaxz_copydepth_sh = DRW_shader_create_fullscreen(datatoc_effect_minmaxz_frag_glsl, "#define INPUT_DEPTH\n"
																									"#define COPY_DEPTH\n");
	...
}


void EEVEE_effects_cache_init(EEVEE_Data *vedata)
{
	...
	struct Gwn_Batch *quad = DRW_cache_fullscreen_quad_get();

	{
		psl->minmaxz_downlevel = DRW_pass_create("HiZ Down Level", DRW_STATE_WRITE_COLOR);
		DRWShadingGroup *grp = DRW_shgroup_create(e_data.minmaxz_downlevel_sh, psl->minmaxz_downlevel);
		DRW_shgroup_uniform_buffer(grp, "depthBuffer", &stl->g_data->minmaxz);
		DRW_shgroup_call_add(grp, quad, NULL);

		psl->minmaxz_downdepth = DRW_pass_create("HiZ Down Depth", DRW_STATE_WRITE_COLOR);
		grp = DRW_shgroup_create(e_data.minmaxz_downdepth_sh, psl->minmaxz_downdepth);
		DRW_shgroup_uniform_buffer(grp, "depthBuffer", &dtxl->depth);
		DRW_shgroup_call_add(grp, quad, NULL);

		psl->minmaxz_copydepth = DRW_pass_create("HiZ Copy Depth", DRW_STATE_WRITE_COLOR);
		grp = DRW_shgroup_create(e_data.minmaxz_copydepth_sh, psl->minmaxz_copydepth);
		DRW_shgroup_uniform_buffer(grp, "depthBuffer", &dtxl->depth);
		DRW_shgroup_call_add(grp, quad, NULL);
	}
	...
}

```
>
- 这里是为了之后的渲染做准备的，这里是用 DRW_cache_fullscreen_quad_get 作为模型，做后处理，psl->minmaxz_downlevel, psl->minmaxz_downdepth, psl->minmaxz_copydepth 使用的Shader 都是 effect_minmaxz_frag.glsl，但是开启不一样的宏
- e_data.minmaxz_downlevel_sh 没有开启特殊的宏
- e_data.minmaxz_downdepth_sh 开启INPUT_DEPTH
- e_data.minmaxz_copydepth_sh 开启了 INPUT_DEPTH 和 COPY_DEPTH


## 渲染

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Create minmax texture */
	EEVEE_create_minmax_buffer(vedata);
	...
}

```


*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_Data *vedata)
{
	...
	/* MinMax Pyramid */
	/* TODO reduce precision */
	DRWFboTexture tex = {&stl->g_data->minmaxz, DRW_TEX_RG_32, DRW_TEX_MIPMAP | DRW_TEX_TEMP};

	DRW_framebuffer_init(&fbl->minmaxz_fb, &draw_engine_eevee_type,
	                    (int)viewport_size[0] / 2, (int)viewport_size[1] / 2,
	                    &tex, 1);
	...
}


static void minmax_downsample_cb(void *vedata, int UNUSED(level))
{
	EEVEE_PassList *psl = ((EEVEE_Data *)vedata)->psl;
	DRW_draw_pass(psl->minmaxz_downlevel);
}

void EEVEE_create_minmax_buffer(EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;

	/* Copy depth buffer to minmax texture top level */
	DRW_framebuffer_texture_attach(fbl->minmaxz_fb, stl->g_data->minmaxz, 0, 0);
	DRW_framebuffer_bind(fbl->minmaxz_fb);
	DRW_draw_pass(psl->minmaxz_downdepth);
	DRW_framebuffer_texture_detach(stl->g_data->minmaxz);

	/* Create lower levels */
	DRW_framebuffer_recursive_downsample(fbl->minmaxz_fb, stl->g_data->minmaxz, 6, &minmax_downsample_cb, vedata);
}
```
>
- 在渲染管线多了一个步骤，多了 DRW_draw_pass(psl->minmaxz_downdepth); 和 DRW_framebuffer_recursive_downsample(fbl->minmaxz_fb, stl->g_data->minmaxz, 6, &minmax_downsample_cb, vedata); 分别使用了 minmaxz_downdepth 和 psl->minmaxz_downlevel
- psl->minmaxz_downdepth 使用了 e_data.minmaxz_downdepth_sh, 开启INPUT_DEPTH
- psl->minmaxz_downlevel 使用了 e_data.minmaxz_downlevel_sh, 没有开启特殊的宏
- minmax depth texture 就是通过上面两个步骤渲染出来的, minmax depth 第0个level使用 psl->minmaxz_downdepth，之后level的都使用psl->minmaxz_downlevel

## Shader

*effect_minmaxz_frag.glsl*
```
/**
 * Shader that downsample depth buffer,
 * saving min and max value of each texel in the above mipmaps.
 * Adapted from http://rastergrid.com/blog/2010/10/hierarchical-z-map-based-occlusion-culling/
 **/

uniform sampler2D depthBuffer;

out vec4 FragMinMax;

vec2 sampleLowerMip(ivec2 texel)
{
#ifdef INPUT_DEPTH   // 当level = 0 的时候执行
	return texelFetch(depthBuffer, texel, 0).rr;
#else
	return texelFetch(depthBuffer, texel, 0).rg;
#endif
}

void minmax(inout vec2 val[2])
{
	val[0].x = min(val[0].x, val[1].x);
	val[0].y = max(val[0].y, val[1].y);
}

void main()
{
	vec2 val[2];
	ivec2 texelPos = ivec2(gl_FragCoord.xy);
	ivec2 mipsize = textureSize(depthBuffer, 0);
#ifndef COPY_DEPTH
	texelPos *= 2;
#endif

	val[0] = sampleLowerMip(texelPos);
#ifndef COPY_DEPTH
	val[1] = sampleLowerMip(texelPos + ivec2(1, 0));
	minmax(val);
	val[1] = sampleLowerMip(texelPos + ivec2(1, 1));
	minmax(val);
	val[1] = sampleLowerMip(texelPos + ivec2(0, 1));
	minmax(val);

	/* if we are reducing an odd-width texture then fetch the edge texels */
	if (((mipsize.x & 1) != 0) && (int(gl_FragCoord.x) == mipsize.x-3)) {
		/* if both edges are odd, fetch the top-left corner texel */
		if (((mipsize.y & 1) != 0) && (int(gl_FragCoord.y) == mipsize.y-3)) {
			val[1] = sampleLowerMip(texelPos + ivec2(-1, -1));
			minmax(val);
		}
		val[1] = sampleLowerMip(texelPos + ivec2(0, -1));
		minmax(val);
		val[1] = sampleLowerMip(texelPos + ivec2(1, -1));
		minmax(val);
	}
	/* if we are reducing an odd-height texture then fetch the edge texels */
	else if (((mipsize.y & 1) != 0) && (int(gl_FragCoord.y) == mipsize.y-3)) {
		val[1] = sampleLowerMip(texelPos + ivec2(0, -1));
		minmax(val);
		val[1] = sampleLowerMip(texelPos + ivec2(1, -1));
		minmax(val);
	}
#endif

	FragMinMax = vec4(val[0], 0.0, 1.0);
}
```
>
- 当渲染level = 0 的时候，开启INPUT_DEPTH宏，传入depthbuffer是origin depth buffer，通过上面的代码可以发现，通过降维，大小变一半的方式，得出的texture的每一个像素是 自身 和(+1,0), (0,+1), (+1,+1)的像素进行对比出来 min，max 出来，然后输出的像素的 r 保存的是 min 值，g 保存的是max.<br><br>
- 剩下的level渲染的时候，没有开启任何的宏，传入depthbuffer是保存了min，max的 level = 0 的depth texture, 渲染的思路跟第一步基本一致，最终保存也是min和max 的depth value.
