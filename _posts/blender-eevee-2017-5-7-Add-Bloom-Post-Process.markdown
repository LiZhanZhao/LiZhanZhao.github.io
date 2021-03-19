---
layout:     post
title:      "blender eevee Add Bloom Post Process"
subtitle:   ""
date:       2020-09-17 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> 2017/5/7  * Eevee: Add Bloom post process. <br>  
Based on Kino/Bloom v2 - Bloom filter for Unity


## 作用 
Bloom 后处理效果



## 编译

- 直接编译


## 效果展示


![](/img/Eevee/Bloom/1.png)



## Bloom 算法思路

```
/**  Bloom algorithm
*
* Overview :
* - Downsample the color buffer doing a small blur during each step.
* - Accumulate bloom color using previously downsampled color buffers
*   and do an upsample blur for each new accumulated layer.
* - Finally add accumulation buffer onto the source color buffer.
*
*  [1/1] is original copy resolution (can be half or quater res for performance)
*
*                                [DOWNSAMPLE CHAIN]                      [UPSAMPLE CHAIN]
*
*  Source Color ── [Blit] ──>  Bright Color Extract [1/1]                  Final Color
*                                        |                                      Λ
*                                [Downsample First]       Source Color ─> + [Resolve]
*                                        v                                      |
*                              Color Downsampled [1/2] ────────────> + Accumulation Buffer [1/2]
*                                        |                                      Λ
*                                       ───                                    ───
*                                      Repeat                                 Repeat
*                                       ───                                    ───
*                                        v                                      |
*                              Color Downsampled [1/N-1] ──────────> + Accumulation Buffer [1/N-1]
*                                        |                                      Λ
*                                   [Downsample]                            [Upsample]
*                                        v                                      |
*                              Color Downsampled [1/N] ─────────────────────────┘
**/
```


## 渲染过程
**eevee_effects.c**
```
/* Bloom */
if ((effects->enabled_effects & EFFECT_BLOOM) != 0) {
	struct GPUTexture *last;

	/* Extract bright pixels */
	copy_v2_v2(effects->unf_source_texel_size, effects->source_texel_size);
	effects->unf_source_buffer = effects->source_buffer;

	DRW_framebuffer_bind(fbl->bloom_blit_fb);
	DRW_draw_pass(psl->bloom_blit);

	/* Downsample */
	copy_v2_v2(effects->unf_source_texel_size, effects->blit_texel_size);
	effects->unf_source_buffer = txl->bloom_blit;

	DRW_framebuffer_bind(fbl->bloom_down_fb[0]);
	DRW_draw_pass(psl->bloom_downsample_first);

	last = txl->bloom_downsample[0];

	for (int i = 1; i < effects->bloom_iteration_ct; ++i) {
		copy_v2_v2(effects->unf_source_texel_size, effects->downsamp_texel_size[i-1]);
		effects->unf_source_buffer = last;

		DRW_framebuffer_bind(fbl->bloom_down_fb[i]);
		DRW_draw_pass(psl->bloom_downsample);

		/* Used in next loop */
		last = txl->bloom_downsample[i];
	}

	/* Upsample and accumulate */
	for (int i = effects->bloom_iteration_ct - 2; i >= 0; --i) {
		copy_v2_v2(effects->unf_source_texel_size, effects->downsamp_texel_size[i]);
		effects->unf_source_buffer = txl->bloom_downsample[i];
		effects->unf_base_buffer = last;

		DRW_framebuffer_bind(fbl->bloom_accum_fb[i]);
		DRW_draw_pass(psl->bloom_upsample);

		last = txl->bloom_upsample[i];
	}

	/* Resolve */
	copy_v2_v2(effects->unf_source_texel_size, effects->downsamp_texel_size[0]);
	effects->unf_source_buffer = last;
	effects->unf_base_buffer = effects->source_buffer;

	eevee_effect_framebuffer_bind(vedata);
	DRW_draw_pass(psl->bloom_resolve);
}
```

### 1. Extract bright pixels
```
/* Extract bright pixels */
copy_v2_v2(effects->unf_source_texel_size, effects->source_texel_size);
effects->unf_source_buffer = effects->source_buffer;

DRW_framebuffer_bind(fbl->bloom_blit_fb);
DRW_draw_pass(psl->bloom_blit);
```
>
- effects->source_buffer 是Bloom执行前的画面
- 调用 psl->bloom_blit 来提取亮度，渲染到 fbl->bloom_blit_fb 中，其实就是渲染到纹理 txl->bloom_blit 中


#### Shader

```


uniform sampler2D sourceBuffer; /* Buffer to filter */
uniform vec2 sourceBufferTexelSize;

/* Step Blit */
uniform vec4 curveThreshold;

/* Step Upsample */
uniform sampler2D baseBuffer; /* Previous accumulation buffer */
uniform vec2 baseBufferTexelSize;
uniform float sampleScale;

/* Step Resolve */
uniform float bloomIntensity;

in vec4 uvcoordsvar;

out vec4 FragColor;

/* -------------- Utils ------------- */

float brightness(vec3 c)
{
    return max(max(c.r, c.g), c.b);
}

/* 3-tap median filter */
vec3 median(vec3 a, vec3 b, vec3 c)
{
    return a + b + c - min(min(a, b), c) - max(max(a, b), c);
}

...

/* ----------- Steps ----------- */

vec4 step_blit(void)
{
	vec2 uv = uvcoordsvar.xy + sourceBufferTexelSize.xy * 0.5;

#ifdef HIGH_QUALITY /* Anti flicker */
	vec3 d = sourceBufferTexelSize.xyx * vec3(1, 1, 0);
	vec3 s0 = texture(sourceBuffer, uvcoordsvar.xy).rgb;
	vec3 s1 = texture(sourceBuffer, uvcoordsvar.xy - d.xz).rgb;
	vec3 s2 = texture(sourceBuffer, uvcoordsvar.xy + d.xz).rgb;
	vec3 s3 = texture(sourceBuffer, uvcoordsvar.xy - d.zy).rgb;
	vec3 s4 = texture(sourceBuffer, uvcoordsvar.xy + d.zy).rgb;
	vec3 m = median(median(s0.rgb, s1, s2), s3, s4);
#else
	vec3 s0 = texture(sourceBuffer, uvcoordsvar.xy).rgb;
	vec3 m = s0.rgb;
#endif

	/* Pixel brightness */
	float br = brightness(m);

	/* Under-threshold part: quadratic curve */
	float rq = clamp(br - curveThreshold.x, 0, curveThreshold.y);
	rq = curveThreshold.z * rq * rq;

	/* Combine and apply the brightness response curve. */
	m *= max(rq, br - curveThreshold.w) / max(br, 1e-5);

	return vec4(m, 1.0);
}

...

void main(void)
{
#if defined(STEP_BLIT)
	FragColor = step_blit();
#elif 
...
#endif
}
```


### 2. Downsample




```
/* Downsample buffers */
blitsize[0] = (int)viewport_size[0];
blitsize[1] = (int)viewport_size[1];

copy_v2_v2_int(texsize, blitsize);
for (int i = 0; i < effects->bloom_iteration_ct; ++i) {
	texsize[0] /= 2; texsize[1] /= 2;
	texsize[0] = MAX2(texsize[0], 2);
	texsize[1] = MAX2(texsize[1], 2);

	effects->downsamp_texel_size[i][0] = 1.0f / (float)texsize[0];
	effects->downsamp_texel_size[i][1] = 1.0f / (float)texsize[1];

	DRWFboTexture tex_bloom = {&txl->bloom_downsample[i], DRW_BUF_RGBA_16, DRW_TEX_FILTER};
	DRW_framebuffer_init(&fbl->bloom_down_fb[i],
						(int)texsize[0], (int)texsize[1],
						&tex_bloom, 1);
}
```
>
- 渲染前进行framebuffer和纹理大小初始化


<br>

```
const bool use_antiflicker = true;
eevee_create_bloom_pass("Bloom Downsample First", effects, e_data.bloom_downsample_sh[use_antiflicker], &psl->bloom_downsample_first, false);
eevee_create_bloom_pass("Bloom Downsample", effects, e_data.bloom_downsample_sh[0], &psl->bloom_downsample, false);
```

<br>



```
/* Downsample */
copy_v2_v2(effects->unf_source_texel_size, effects->blit_texel_size);
effects->unf_source_buffer = txl->bloom_blit;

DRW_framebuffer_bind(fbl->bloom_down_fb[0]);
DRW_draw_pass(psl->bloom_downsample_first);

last = txl->bloom_downsample[0];

for (int i = 1; i < effects->bloom_iteration_ct; ++i) {
	copy_v2_v2(effects->unf_source_texel_size, effects->downsamp_texel_size[i-1]);
	effects->unf_source_buffer = last;

	DRW_framebuffer_bind(fbl->bloom_down_fb[i]);
	DRW_draw_pass(psl->bloom_downsample);

	/* Used in next loop */
	last = txl->bloom_downsample[i];
}
```



>
- 传入第一步的纹理txl->bloom_blit，提取亮度的纹理进行downsample处理
- 每一步 downsample之后的纹理保存到 fbl->bloom_down_fb[i] 中, fbl->bloom_down_fb[i] 关联 &txl->bloom_downsample[i] 纹理


#### Shader

```


uniform sampler2D sourceBuffer; /* Buffer to filter */
uniform vec2 sourceBufferTexelSize;

/* Step Blit */
uniform vec4 curveThreshold;

/* Step Upsample */
uniform sampler2D baseBuffer; /* Previous accumulation buffer */
uniform vec2 baseBufferTexelSize;
uniform float sampleScale;

/* Step Resolve */
uniform float bloomIntensity;

in vec4 uvcoordsvar;

out vec4 FragColor;

/* -------------- Utils ------------- */

float brightness(vec3 c)
{
    return max(max(c.r, c.g), c.b);
}

/* 3-tap median filter */
vec3 median(vec3 a, vec3 b, vec3 c)
{
    return a + b + c - min(min(a, b), c) - max(max(a, b), c);
}

/* ------------- Filters ------------ */

vec3 downsample_filter_high(sampler2D tex, vec2 uv, vec2 texelSize)
{
	/* Downsample with a 4x4 box filter + anti-flicker filter */
	vec4 d = texelSize.xyxy * vec4(-1, -1, +1, +1);

	vec3 s1 = texture(tex, uv + d.xy).rgb;
	vec3 s2 = texture(tex, uv + d.zy).rgb;
	vec3 s3 = texture(tex, uv + d.xw).rgb;
	vec3 s4 = texture(tex, uv + d.zw).rgb;

	/* Karis's luma weighted average (using brightness instead of luma) */
	float s1w = 1.0 / (brightness(s1) + 1.0);
	float s2w = 1.0 / (brightness(s2) + 1.0);
	float s3w = 1.0 / (brightness(s3) + 1.0);
	float s4w = 1.0 / (brightness(s4) + 1.0);
	float one_div_wsum = 1.0 / (s1w + s2w + s3w + s4w);

	return (s1 * s1w + s2 * s2w + s3 * s3w + s4 * s4w) * one_div_wsum;
}

vec3 downsample_filter(sampler2D tex, vec2 uv, vec2 texelSize)
{
	/* Downsample with a 4x4 box filter */
	vec4 d = texelSize.xyxy * vec4(-1, -1, +1, +1);

	vec3 s;
	s  = texture(tex, uv + d.xy).rgb;
	s += texture(tex, uv + d.zy).rgb;
	s += texture(tex, uv + d.xw).rgb;
	s += texture(tex, uv + d.zw).rgb;

	return s * (1.0 / 4);
}


vec4 step_downsample(void)
{
#ifdef HIGH_QUALITY /* Anti flicker */
	vec3 sample = downsample_filter_high(sourceBuffer, uvcoordsvar.xy, sourceBufferTexelSize);
#else
	vec3 sample = downsample_filter(sourceBuffer, uvcoordsvar.xy, sourceBufferTexelSize);
#endif
	return vec4(sample, 1.0);
}


void main(void)
{
	...
	FragColor = step_downsample();
	...
}
```


### 3. Upsample
```
/* Upsample buffers */
copy_v2_v2_int(texsize, blitsize);
for (int i = 0; i < effects->bloom_iteration_ct - 1; ++i) {
	texsize[0] /= 2; texsize[1] /= 2;
	texsize[0] = MAX2(texsize[0], 2);
	texsize[1] = MAX2(texsize[1], 2);

	DRWFboTexture tex_bloom = {&txl->bloom_upsample[i], DRW_BUF_RGBA_16, DRW_TEX_FILTER};
	DRW_framebuffer_init(&fbl->bloom_accum_fb[i],
						(int)texsize[0], (int)texsize[1],
						&tex_bloom, 1);
}
```
>
- 渲染前初始化

<br>

```
/* Upsample and accumulate */
for (int i = effects->bloom_iteration_ct - 2; i >= 0; --i) {
	copy_v2_v2(effects->unf_source_texel_size, effects->downsamp_texel_size[i]);
	effects->unf_source_buffer = txl->bloom_downsample[i];
	effects->unf_base_buffer = last;

	DRW_framebuffer_bind(fbl->bloom_accum_fb[i]);
	DRW_draw_pass(psl->bloom_upsample);

	last = txl->bloom_upsample[i];
}

```
>
- 传入 上一步的 渲染好的 txl->bloom_downsample[i] 进行 Upsample
- 把结果渲染到 fbl->bloom_accum_fb[i] 中

#### Shader
```

uniform sampler2D sourceBuffer; /* Buffer to filter */
uniform vec2 sourceBufferTexelSize;

/* Step Blit */
uniform vec4 curveThreshold;

/* Step Upsample */
uniform sampler2D baseBuffer; /* Previous accumulation buffer */
uniform vec2 baseBufferTexelSize;
uniform float sampleScale;

/* Step Resolve */
uniform float bloomIntensity;

in vec4 uvcoordsvar;

out vec4 FragColor;

/* -------------- Utils ------------- */

float brightness(vec3 c)
{
    return max(max(c.r, c.g), c.b);
}

/* 3-tap median filter */
vec3 median(vec3 a, vec3 b, vec3 c)
{
    return a + b + c - min(min(a, b), c) - max(max(a, b), c);
}

/* ------------- Filters ------------ */



vec3 upsample_filter_high(sampler2D tex, vec2 uv, vec2 texelSize)
{
	/* 9-tap bilinear upsampler (tent filter) */
	vec4 d = texelSize.xyxy * vec4(1, 1, -1, 0) * sampleScale;

	vec3 s;
	s  = texture(tex, uv - d.xy).rgb;
	s += texture(tex, uv - d.wy).rgb * 2;
	s += texture(tex, uv - d.zy).rgb;

	s += texture(tex, uv + d.zw).rgb * 2;
	s += texture(tex, uv       ).rgb * 4;
	s += texture(tex, uv + d.xw).rgb * 2;

	s += texture(tex, uv + d.zy).rgb;
	s += texture(tex, uv + d.wy).rgb * 2;
	s += texture(tex, uv + d.xy).rgb;

	return s * (1.0 / 16.0);
}

vec3 upsample_filter(sampler2D tex, vec2 uv, vec2 texelSize)
{
	/* 4-tap bilinear upsampler */
	vec4 d = texelSize.xyxy * vec4(-1, -1, +1, +1) * (sampleScale * 0.5);

	vec3 s;
	s  = texture(tex, uv + d.xy).rgb;
	s += texture(tex, uv + d.zy).rgb;
	s += texture(tex, uv + d.xw).rgb;
	s += texture(tex, uv + d.zw).rgb;

	return s * (1.0 / 4.0);
}




vec4 step_upsample(void)
{
#ifdef HIGH_QUALITY
	vec3 blur = upsample_filter_high(sourceBuffer, uvcoordsvar.xy, sourceBufferTexelSize);
#else
	vec3 blur = upsample_filter(sourceBuffer, uvcoordsvar.xy, sourceBufferTexelSize);
#endif
	vec3 base = texture(baseBuffer, uvcoordsvar.xy).rgb;
	return vec4(base + blur, 1.0);
}



void main(void)
{
	...
	FragColor = step_upsample();
	...
}
```



### 4. Resolve 
```
copy_v2_v2(effects->unf_source_texel_size, effects->downsamp_texel_size[0]);
effects->unf_source_buffer = last;
effects->unf_base_buffer = effects->source_buffer;

eevee_effect_framebuffer_bind(vedata);
DRW_draw_pass(psl->bloom_resolve);
```
>
- upsample 1/2 纹理进行 Resolve 到 1/1 纹理
- effects->unf_base_buffer = effects->source_buffer : 传入bloom前的画面

#### Shader
```


uniform sampler2D sourceBuffer; /* Buffer to filter */
uniform vec2 sourceBufferTexelSize;

/* Step Blit */
uniform vec4 curveThreshold;

/* Step Upsample */
uniform sampler2D baseBuffer; /* Previous accumulation buffer */
uniform vec2 baseBufferTexelSize;
uniform float sampleScale;

/* Step Resolve */
uniform float bloomIntensity;

in vec4 uvcoordsvar;

out vec4 FragColor;

/* -------------- Utils ------------- */

float brightness(vec3 c)
{
    return max(max(c.r, c.g), c.b);
}

/* 3-tap median filter */
vec3 median(vec3 a, vec3 b, vec3 c)
{
    return a + b + c - min(min(a, b), c) - max(max(a, b), c);
}


vec3 upsample_filter_high(sampler2D tex, vec2 uv, vec2 texelSize)
{
	/* 9-tap bilinear upsampler (tent filter) */
	vec4 d = texelSize.xyxy * vec4(1, 1, -1, 0) * sampleScale;

	vec3 s;
	s  = texture(tex, uv - d.xy).rgb;
	s += texture(tex, uv - d.wy).rgb * 2;
	s += texture(tex, uv - d.zy).rgb;

	s += texture(tex, uv + d.zw).rgb * 2;
	s += texture(tex, uv       ).rgb * 4;
	s += texture(tex, uv + d.xw).rgb * 2;

	s += texture(tex, uv + d.zy).rgb;
	s += texture(tex, uv + d.wy).rgb * 2;
	s += texture(tex, uv + d.xy).rgb;

	return s * (1.0 / 16.0);
}

vec3 upsample_filter(sampler2D tex, vec2 uv, vec2 texelSize)
{
	/* 4-tap bilinear upsampler */
	vec4 d = texelSize.xyxy * vec4(-1, -1, +1, +1) * (sampleScale * 0.5);

	vec3 s;
	s  = texture(tex, uv + d.xy).rgb;
	s += texture(tex, uv + d.zy).rgb;
	s += texture(tex, uv + d.xw).rgb;
	s += texture(tex, uv + d.zw).rgb;

	return s * (1.0 / 4.0);
}






vec4 step_resolve(void)
{
#ifdef HIGH_QUALITY
	vec3 blur = upsample_filter_high(sourceBuffer, uvcoordsvar.xy, sourceBufferTexelSize);
#else
	vec3 blur = upsample_filter(sourceBuffer, uvcoordsvar.xy, sourceBufferTexelSize);
#endif
	vec4 base = texture(baseBuffer, uvcoordsvar.xy);
	vec3 cout = base.rgb + blur * bloomIntensity;
	return vec4(cout, base.a);
}

void main(void)
{
...
	FragColor = step_resolve();
...
}
```


### 传入参数到Shader中
```
{
	/* Bloom */
	EEVEE_EffectsInfo *effects = stl->effects;
	int blitsize[2], texsize[2];

	/* Blit Buffer */
	effects->source_texel_size[0] = 1.0f / viewport_size[0];
	effects->source_texel_size[1] = 1.0f / viewport_size[1];

	blitsize[0] = (int)viewport_size[0];
	blitsize[1] = (int)viewport_size[1];

	effects->blit_texel_size[0] = 1.0f / (float)blitsize[0];
	effects->blit_texel_size[1] = 1.0f / (float)blitsize[1];

	DRWFboTexture tex_blit = {&txl->bloom_blit, DRW_BUF_RGBA_16, DRW_TEX_FILTER};
	DRW_framebuffer_init(&fbl->bloom_blit_fb,
						(int)blitsize[0], (int)blitsize[1],
						&tex_blit, 1);

	/* Parameters */
	/* TODO UI Options */
	float threshold = 0.8f;
	float knee = 0.5f;
	float intensity = 0.8f;
	float radius = 8.5f;

	/* determine the iteration count */
	const float minDim = (float)MIN2(blitsize[0], blitsize[1]);
	const float maxIter = (radius - 8.0f) + log(minDim) / log(2);
	const int maxIterInt = effects->bloom_iteration_ct = (int)maxIter;

	CLAMP(effects->bloom_iteration_ct, 1, MAX_BLOOM_STEP);

	effects->bloom_sample_scale = 0.5f + maxIter - (float)maxIterInt;
	effects->bloom_curve_threshold[0] = threshold - knee;
	effects->bloom_curve_threshold[1] = knee * 2.0f;
	effects->bloom_curve_threshold[2] = 0.25f / knee;
	effects->bloom_curve_threshold[3] = threshold;
	effects->bloom_intensity = intensity;

	/* Downsample buffers */
	copy_v2_v2_int(texsize, blitsize);
	for (int i = 0; i < effects->bloom_iteration_ct; ++i) {
		texsize[0] /= 2; texsize[1] /= 2;
		texsize[0] = MAX2(texsize[0], 2);
		texsize[1] = MAX2(texsize[1], 2);

		effects->downsamp_texel_size[i][0] = 1.0f / (float)texsize[0];
		effects->downsamp_texel_size[i][1] = 1.0f / (float)texsize[1];

		DRWFboTexture tex_bloom = {&txl->bloom_downsample[i], DRW_BUF_RGBA_16, DRW_TEX_FILTER};
		DRW_framebuffer_init(&fbl->bloom_down_fb[i],
							(int)texsize[0], (int)texsize[1],
							&tex_bloom, 1);
	}

	/* Upsample buffers */
	copy_v2_v2_int(texsize, blitsize);
	for (int i = 0; i < effects->bloom_iteration_ct - 1; ++i) {
		texsize[0] /= 2; texsize[1] /= 2;
		texsize[0] = MAX2(texsize[0], 2);
		texsize[1] = MAX2(texsize[1], 2);

		DRWFboTexture tex_bloom = {&txl->bloom_upsample[i], DRW_BUF_RGBA_16, DRW_TEX_FILTER};
		DRW_framebuffer_init(&fbl->bloom_accum_fb[i],
							(int)texsize[0], (int)texsize[1],
							&tex_bloom, 1);
	}

	effects->enabled_effects |= EFFECT_BLOOM;
}



static DRWShadingGroup *eevee_create_bloom_pass(const char *name, EEVEE_EffectsInfo *effects, struct GPUShader *sh, DRWPass **pass, bool upsample)
{
	struct Batch *quad = DRW_cache_fullscreen_quad_get();

	*pass = DRW_pass_create(name, DRW_STATE_WRITE_COLOR);

	DRWShadingGroup *grp = DRW_shgroup_create(sh, *pass);
	DRW_shgroup_call_add(grp, quad, NULL);
	DRW_shgroup_uniform_buffer(grp, "sourceBuffer", &effects->unf_source_buffer, 0);
	DRW_shgroup_uniform_vec2(grp, "sourceBufferTexelSize", effects->unf_source_texel_size, 1);
	if (upsample) {
		DRW_shgroup_uniform_buffer(grp, "baseBuffer", &effects->unf_base_buffer, 1);
		DRW_shgroup_uniform_float(grp, "sampleScale", &effects->bloom_sample_scale, 1);
	}

	return grp;
}



```

### Ping Pong buffer

*eevee_engine.c*
```
DRWFboTexture tex = {&txl->color, DRW_BUF_RGBA_16, DRW_TEX_FILTER};

const float *viewport_size = DRW_viewport_size_get();
DRW_framebuffer_init(&fbl->main,
					(int)viewport_size[0], (int)viewport_size[1],
					&tex, 1);
```

*eevee_effects.c*
```
/* Ping Pong buffer */
DRWFboTexture tex = {&txl->color_post, DRW_BUF_RGBA_16, DRW_TEX_FILTER};

const float *viewport_size = DRW_viewport_size_get();
DRW_framebuffer_init(&fbl->effect_fb,
					(int)viewport_size[0], (int)viewport_size[1],
					&tex, 1);



/* Ping pong between 2 buffers */
static void eevee_effect_framebuffer_bind(EEVEE_Data *vedata)
{
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_EffectsInfo *effects = vedata->stl->effects;

	DRW_framebuffer_bind(effects->target_buffer);

	if (effects->source_buffer == txl->color) {
		effects->source_buffer = txl->color_post;
		effects->target_buffer = fbl->main;
	}
	else {
		effects->source_buffer = txl->color;
		effects->target_buffer = fbl->effect_fb;
	}
}
```
>
- fbl->effect_fb 关联 txl->color_post
- fbl->main 关联  txl->color
- eevee_effect_framebuffer_bind 其实就是在 fbl->effect_fb 和 fbl->main 这两framebuffer 来回切换