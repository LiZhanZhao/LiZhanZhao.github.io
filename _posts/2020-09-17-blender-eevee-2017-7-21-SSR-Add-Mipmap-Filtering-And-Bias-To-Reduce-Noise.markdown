---
layout:     post
title:      "blender eevee SSR Add mipmap filtering and bias to reduce noise"
subtitle:   ""
date:       2021-3-6 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/21  *   Eevee: SSR: Add mipmap filtering and bias to reduce noise .<br> 

> Also fix the roughness factors.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/SSR/04/1.png)



## 作用
SSR 添加 反射的 color texture 带 mipmap，减少bias 和 noise

## 编译
- 重新生成SLN
- git 定位到  2017/7/21  * Eevee: SSR: Add mipmap filtering and bias to reduce noise .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## SSR Color Buffer RT DownSample

### 渲染
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	e_data.downsample_sh = DRW_shader_create_fullscreen(datatoc_effect_downsample_frag_glsl, NULL);
	...
}

void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	DRWShadingGroup *grp = DRW_shgroup_create(e_data.downsample_sh, psl->color_downsample_ps);
	...

	{
		psl->color_downsample_ps = DRW_pass_create("Downsample", DRW_STATE_WRITE_COLOR);
		DRWShadingGroup *grp = DRW_shgroup_create(e_data.downsample_sh, psl->color_downsample_ps);
		DRW_shgroup_uniform_buffer(grp, "source", &e_data.color_src);
		DRW_shgroup_call_add(grp, quad, NULL);
	}
}


static void simple_downsample_cb(void *vedata, int UNUSED(level))
{
	EEVEE_PassList *psl = ((EEVEE_Data *)vedata)->psl;
	DRW_draw_pass(psl->color_downsample_ps);
}


/**
 * Simple downsampling algorithm. Reconstruct mip chain up to mip level.
 **/
void EEVEE_downsample_buffer(EEVEE_Data *vedata, struct GPUFrameBuffer *fb_src, GPUTexture *texture_src, int level)
{
	e_data.color_src = texture_src;

	/* Create lower levels */
	DRW_framebuffer_recursive_downsample(fb_src, texture_src, level, &simple_downsample_cb, vedata);
}

void EEVEE_effects_do_ssr(EEVEE_SceneLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	...
	if ((effects->enabled_effects & EFFECT_SSR) != 0 && stl->g_data->valid_double_buffer) {
		...
		EEVEE_downsample_buffer(vedata, fbl->minmaxz_fb, txl->color_double_buffer, 5);
		...
	}
	...
}
```
>
- 对 color_double_buffer RT 进行mipmap的构建， DownSample, 使用 effect_downsample_frag.glsl
```
uniform sampler2D source;
out vec4 FragColor;
void main()
{
	/* Reconstructing Target uvs like this avoid missing pixels if NPO2 */
	vec2 uvs = gl_FragCoord.xy * 2.0 / vec2(textureSize(source, 0));
	FragColor = texture(source, uvs);
}
```

<br><br>

## Shader
上面直接把 color_double_buffer  构造了 mipmap，mipmap都是进行DownSample, 下面的就是用法

*effect_ssr_frag.glsl*
```
#define BRDF_BIAS 0.7

vec3 generate_ray(ivec2 pix, vec3 V, vec3 N, float a2, out float pdf)
{
	float NH;
	vec3 T, B;
	make_orthonormal_basis(N, T, B); /* Generate tangent space */
	vec3 rand = texelFetch(utilTex, ivec3(pix % LUT_SIZE, 2), 0).rba;

	/* Importance sampling bias */
	rand.x = mix(rand.x, 0.0, BRDF_BIAS);

	vec3 H = sample_ggx(rand, a2, N, T, B, NH); /* Microfacet normal */
	pdf = min(1024e32, pdf_ggx_reflect(NH, a2)); /* Theoretical limit of 16bit float */
	return reflect(-V, H);
}

```
>
- 优化生成ray算法，Importance sampling bias

<br><br>

*bsdf_common_lib.glsl*
```

/* ----------- Cone Apperture Approximation --------- */

/* Return a fitted cone angle given the input roughness */
float cone_cosine(float r)
{
	/* Using phong gloss
	 * roughness = sqrt(2/(gloss+2)) */
	float gloss = -2 + 2 / (r * r);
	/* Drobot 2014 in GPUPro5 */
	// return cos(2.0 * sqrt(2.0 / (gloss + 2)));
	/* Uludag 2014 in GPUPro5 */
	// return pow(0.244, 1 / (gloss + 1));
	/* Jimenez 2016 in Practical Realtime Strategies for Accurate Indirect Occlusion*/
	return exp2(-3.32193 * r * r);
}
```

<br><br>

*effect_ssr_frag.glsl*
```
#else /* STEP_RESOLVE
...

void main()
{
	...
	float roughness = speccol_roughness.a;
	float roughnessSquared = max(1e-3, roughness * roughness);

	vec4 spec_accum = vec4(0.0);

	/* Resolve SSR */
	float cone_cos = cone_cosine(roughness);
	float cone_tan = sqrt(1 - cone_cos * cone_cos) / cone_cos;
	cone_tan *= mix(saturate(dot(N, V) * 2.0), 1.0, sqrt(roughness)); /* Elongation fit */

	vec3 ssr_accum = vec3(0.0);
	float weight_acc = 0.0;
	float mask_acc = 0.0;
	float dist_acc = 0.0;
	float hit_acc = 0.0;
	const ivec2 neighbors[4] = ivec2[4](ivec2(0, 0), ivec2(1, 1), ivec2(0, 1), ivec2(1, 0));
	ivec2 invert_neighbor;
	invert_neighbor.x = ((fullres_texel.x & 0x1) == 0) ? 1 : -1;
	invert_neighbor.y = ((fullres_texel.y & 0x1) == 0) ? 1 : -1;
	for (int i = 0; i < NUM_NEIGHBORS; i++) {
		ivec2 target_texel = halfres_texel + neighbors[i] * invert_neighbor;

		float pdf = texelFetch(pdfBuffer, target_texel, 0).r;

		/* Check if there was a hit */
		if (pdf > 0.001) {
			vec2 hit_co = texelFetch(hitBuffer, target_texel, 0).rg;

			/* Reconstruct ray */
			float hit_depth = textureLod(depthBuffer, hit_co, 0.0).r;
			vec3 hit_pos = get_world_space_from_depth(hit_co, hit_depth);

			/* Evaluate BSDF */
			vec3 L = normalize(hit_pos - worldPosition);
			float bsdf = bsdf_ggx(N, L, V, roughnessSquared);

			/* Find hit position in previous frame */
			vec2 ref_uvs = get_reprojected_reflection(hit_pos, worldPosition, N);
			vec2 source_uvs = project_point(PastViewProjectionMatrix, worldPosition).xy * 0.5 + 0.5;

			/* Estimate a cone footprint to sample a corresponding mipmap level */
			/* compute cone footprint Using UV distance because we are using screen space filtering */
			float cone_footprint = cone_tan * distance(ref_uvs, source_uvs);
			float mip = BRDF_BIAS * clamp(log2(cone_footprint * max(texture_size.x, texture_size.y)), 0.0, MAX_MIP);

			float border_mask = screen_border_mask(ref_uvs, hit_pos);
			float weight = border_mask * bsdf / max(1e-8, pdf);
			ssr_accum += textureLod(colorBuffer, ref_uvs, mip).rgb * weight;
			weight_acc += weight;
			dist_acc += distance(hit_pos, worldPosition);
			mask_acc += border_mask;
			hit_acc += 1.0;
		}
	}

	/* Compute SSR contribution */
	if (weight_acc > 0.0) {
		/* Fade intensity based on roughness and average distance to hit */
		float fade = saturate(2.0 - roughness * 2.0); /* fade between 0.5 and 1.0 roughness */
		fade *= mask_acc / hit_acc;
		fade *= mask_acc / hit_acc;
		accumulate_light(ssr_accum / weight_acc, fade, spec_accum);
	}
	...
}

#endif
```
>
- 注意的是怎么计算 mipmap level
