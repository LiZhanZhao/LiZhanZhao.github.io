---
layout:     post
title:      "blender eevee SSR Add double buffer so we can read previous frame color."
subtitle:   ""
date:       2021-3-5 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/19  *   Eevee: SSR: Add double buffer so we can read previous frame color .<br> 

> Also add simple reprojection and screen fade to the SSR resolve pass.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/SSR/03/1.png)


## 作用
SSR 的初始版本, 可以反射正常的物体


## 编译
- 重新生成SLN
- git 定位到  2017/7/19  * Eevee: SSR: Add double buffer so we can read previous frame color .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 准备
请先阅读[SSR Add simple raytracing](http://localhost:4000/2021/03/01/blender-eevee-2017-7-18-SSR-Add-Simple-Raytracing/)

## Shader
*effect_ssr_frag.glsl*

```

vec3 generate_ray(vec3 V, vec3 N)
{
	return -reflect(-V, N);
}

#ifdef STEP_RAYTRACE
...

#else /* STEP_RESOLVE */

uniform sampler2D colorBuffer; /* previous frame */
uniform sampler2D depthBuffer;
uniform sampler2D normalBuffer;
uniform sampler2D specroughBuffer;

uniform sampler2D hitBuffer;
uniform sampler2D pdfBuffer;

uniform int probe_count;

uniform mat4 ViewProjectionMatrix;
uniform mat4 PastViewProjectionMatrix;

out vec4 fragColor;

void fallback_cubemap(vec3 N, vec3 V, vec3 W, float roughness, float roughnessSquared, inout vec4 spec_accum)
{
	...
}

#if 0 /* Finish reprojection with motion vectors */
vec3 get_motion_vector(vec3 pos)
{
}

/* http://bitsquid.blogspot.fr/2017/06/reprojecting-reflections_22.html */
vec3 find_reflection_incident_point(vec3 cam, vec3 hit, vec3 pos, vec3 N)
{
	...
}
#endif

vec2 get_reprojected_reflection(vec3 hit, vec3 pos, vec3 N)
{
	/* TODO real motion vectors */
	/* Transform to viewspace */
	// vec4(get_view_space_from_depth(uvcoords, depth), 1.0);
	// vec4(get_view_space_from_depth(uvcoords, depth), 1.0);

	/* Reproject */
	// vec3 hit_reprojected = find_reflection_incident_point(cameraPos, hit, pos, N);

	vec4 hit_co = PastViewProjectionMatrix * vec4(hit, 1.0);
	return (hit_co.xy / hit_co.w) * 0.5 + 0.5;
}

float screen_border_mask(vec2 past_hit_co, vec3 hit)
{
	/* Fade on current and past screen edges */
	vec4 hit_co = ViewProjectionMatrix * vec4(hit, 1.0);
	hit_co.xy = (hit_co.xy / hit_co.w) * 0.5 + 0.5;
	hit_co.zw = past_hit_co;

	const float margin = 0.002;
	const float atten = 0.05 + margin; /* Screen percentage */
	hit_co = smoothstep(margin, atten, hit_co) * (1 - smoothstep(1.0 - atten, 1.0 - margin, hit_co));
	vec2 atten_fac = min(hit_co.xy, hit_co.zw);

	float screenfade = atten_fac.x * atten_fac.y;

	return screenfade;
}

void main()
{
	...

	vec4 spec_accum = vec4(0.0);

	/* Resolve SSR and compute contribution */

	/* We generate the same rays that has been generated in the raycast step.
	 * But we add this ray from our resolve pixel position, increassing accuracy. */
	vec3 R = generate_ray(-V, N);
	float ray_length = texelFetch(hitBuffer, halfres_texel, 0).r;

	if (ray_length != -1.0) {
		vec3 hit_pos = worldPosition + R * ray_length;
		vec2 ref_uvs = get_reprojected_reflection(hit_pos, worldPosition, N);
		spec_accum.a = screen_border_mask(ref_uvs, hit_pos);
		spec_accum.xyz = textureLod(colorBuffer, ref_uvs, 0.0).rgb * spec_accum.a;
	}

	/* If SSR contribution is not 1.0, blend with cubemaps */
	if (spec_accum.a < 1.0) {
		fallback_cubemap(N, V, worldPosition, roughness, roughnessSquared, spec_accum);
	}

	fragColor = vec4(spec_accum.rgb * speccol_roughness.rgb, 1.0);
	// fragColor = vec4(texelFetch(hitBuffer, halfres_texel, 0).rgb, 1.0);
	// fragColor = vec4(R, 1.0);
}

#endif

```
>
- 这里最主要就是下面的函数
```
float ray_length = texelFetch(hitBuffer, halfres_texel, 0).r;
if (ray_length != -1.0) {
	vec3 hit_pos = worldPosition + R * ray_length;
	vec2 ref_uvs = get_reprojected_reflection(hit_pos, worldPosition, N);
	spec_accum.a = screen_border_mask(ref_uvs, hit_pos);
	spec_accum.xyz = textureLod(colorBuffer, ref_uvs, 0.0).rgb * spec_accum.a;
}
```
- 先计算ray碰撞到的物体，交点 hit_pos, 这个hit_pos是世界坐标位置。<br><br>
- get_reprojected_reflection 函数 利用 PastViewProjectionMatrix ，上一帧的 VP矩阵 进行变换 hit_pos, 得到 hit_pos 在上一帧 屏幕空间UV 坐标 ref_uvs 。<br><br>
- screen_border_mask 函数 利用  ViewProjectionMatrix 当前帧的VP矩阵进行变换hit_pos, 得到 hit_pos 在当前帧的 屏幕空间UV 坐标，所以现在得到了 hit_pos 的上一帧和当前帧 的 屏幕空间UV 坐标。<br><br>
- screen_border_mask 函数 里面的 
```
hit_co = smoothstep(margin, atten, hit_co) * (1 - smoothstep(1.0 - atten, 1.0 - margin, hit_co));
vec2 atten_fac = min(hit_co.xy, hit_co.zw);
```
hit_co 在 [0.002, 0.052] 范围是 [0, 1], 少于 0.002 是0, 大于 0.052 是 1, 然后在 [1 - 0.052, 1 - 0.002] 范围 是 [1, 0], 然后 小于 1 - 0.052 的时候是 1， 大于 1 - 0.002 的时候是 0, 也就是 像素在图片中是 1，在图片外是 0，边界是 1 过渡到 0, 所以 atten_fac 得到的 数据就是，考虑到上一帧和当前帧的屏幕UV坐标，uv坐标在屏幕边界是0-1，屏幕内部是1，屏幕外是0 。



## Shader参数

*eevee_effects.c*

```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	/* Setup double buffer so we can access last frame as it was before post processes */
	if ((effects->enabled_effects & EFFECT_DOUBLE_BUFFER) != 0) {
		DRWFboTexture tex_double_buffer = {&txl->color_double_buffer, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER};

		DRW_framebuffer_init(&fbl->double_buffer, &draw_engine_eevee_type,
		                    (int)viewport_size[0], (int)viewport_size[1],
		                    &tex_double_buffer, 1);

		copy_m4_m4(stl->g_data->prev_persmat, stl->g_data->next_persmat);
		DRW_viewport_matrix_get(stl->g_data->next_persmat, DRW_MAT_PERS);
	}
	else {
		/* Cleanup to release memory */
		DRW_TEXTURE_FREE_SAFE(txl->color_double_buffer);
		DRW_FRAMEBUFFER_FREE_SAFE(fbl->double_buffer);
	}
	...
}
```
>
- 初始化 double buffer
- prev_persmat = next_persmat, next_persmat 就是 投影矩阵, DRW_viewport_matrix_get(stl->g_data->next_persmat, DRW_MAT_PERS);

<br><br>

```
void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	DRW_shgroup_uniform_buffer(grp, "colorBuffer", &txl->color_double_buffer);
	...
	DRW_shgroup_uniform_mat4(grp, "PastViewProjectionMatrix", (float *)stl->g_data->prev_persmat);
}

```
>
- 传入参数到Shader中

<br><br>

# 2. Add roughness random rays

## 来源

- 主要看这个commit

> GIT : 2017/7/19  *   Eevee: SSR: Add roughness random rays. .<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/SSR/03/2.png)


## Shader
*bsdf_sampling_lib.glsl*
```
vec3 sample_ggx(vec3 rand, float a2, vec3 N, vec3 T, vec3 B)
{
	/* Theta is the aperture angle of the cone */
	float z = sqrt( (1.0 - rand.x) / ( 1.0 + a2 * rand.x - rand.x ) ); /* cos theta */
	float r = sqrt( 1.0 - z * z ); /* sin theta */
	float x = r * rand.y;
	float y = r * rand.z;

	/* Microfacet Normal */
	vec3 Ht = vec3(x, y, z);
	return tangent_to_world(Ht, N, T, B);
}
```
<br><br>

*effect_ssr_frag.glsl*

```
#ifndef UTIL_TEX
#define UTIL_TEX
uniform sampler2DArray utilTex;
#endif /* UTIL_TEX */

vec3 generate_ray(ivec2 pix, vec3 V, vec3 N, float roughnessSquared)
{
	vec3 T, B;
	make_orthonormal_basis(N, T, B); /* Generate tangent space */
	vec3 rand = texelFetch(utilTex, ivec3(pix % LUT_SIZE, 2), 0).rba;
	vec3 H = sample_ggx(rand, roughnessSquared, N, T, B); /* Microfacet normal */
	return -reflect(-V, H);
}

void main()
{
	...
	/* Generate Ray */
	vec3 R = generate_ray(halfres_texel, V, N, roughnessSquared);
	...
}


```
>
- 这个 commit 最主要的修改就是generate_ray，生成 ray 的方式, 个人理解就是 是利用ggx ,生成不一样的法线出来然后就和view进行 reflect 得到 反射 ray 。

<br><br>

# 3. Add per pixel resolve of multiple rays

## 来源

- 主要看这个commit

> GIT : 2017/7/20  *   Eevee: SSR: Add per pixel resolve of multiple rays. .<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/SSR/03/3.png)


## Shader

### STEP_RAYTRACE 生成 pdf

*bsdf_common_lib.glsl*
```
float D_ggx_opti(float NH, float a2)
{
	float tmp = (NH * a2 - NH) * NH + 1.0;
	return M_PI * tmp*tmp; /* Doing RCP and mul a2 at the end */
}
```
<br><br>

*bsdf_sampling_lib.glsl*
```
float pdf_ggx_reflect(float NH, float a2)
{
	return NH * a2 / D_ggx_opti(NH, a2);
}
```

<br><br>

*effect_ssr_frag.glsl*
```
vec3 generate_ray(ivec2 pix, vec3 V, vec3 N, float roughnessSquared, out float pdf)
{
	float NH;
	vec3 T, B;
	make_orthonormal_basis(N, T, B); /* Generate tangent space */
	vec3 rand = texelFetch(utilTex, ivec3(pix % LUT_SIZE, 2), 0).rba;
	vec3 H = sample_ggx(rand, roughnessSquared, N, T, B, NH); /* Microfacet normal */
	pdf = max(32e32, pdf_ggx_reflect(NH, roughnessSquared)); /* Theoretical limit of 10bit float (not in practice?) */
	return reflect(-V, H);
}

#ifdef STEP_RAYTRACE
	...
	void main()
	{
		...
		/* Generate Ray */
		float pdf;
		vec3 R = generate_ray(halfres_texel, V, N, roughnessSquared, pdf);
		...


		/* Raycast over screen */
		float hit_dist = raycast(depthBuffer, viewPosition, R);

		/* TODO Check if has hit a backface */

		vec2 hit_co = project_point(ProjectionMatrix, viewPosition + R * hit_dist).xy * 0.5 + 0.5;
		hitData = hit_co.xyxy;
		pdfData = vec4(pdf) * step(-0.1, hit_dist);
	}
	...
#else
```
>
- 这里最主要的通过 pdf_ggx_reflect 函数生成 pdf
- STEP_RAYTRACE Pass 生成完的pdf 会传入到 STEP_RESOLVE Pass中

<br><br>

### STEP_RESOLVE

*bsdf_common_lib.glsl*
```
float bsdf_ggx(vec3 N, vec3 L, vec3 V, float roughness)
{
	float a = roughness;
	float a2 = a * a;

	vec3 H = normalize(L + V);
	float NH = max(dot(N, H), 1e-8);
	float NL = max(dot(N, L), 1e-8);
	float NV = max(dot(N, V), 1e-8);

	float G = G1_Smith_GGX(NV, a2) * G1_Smith_GGX(NL, a2); /* Doing RCP at the end */
	float D = D_ggx_opti(NH, a2);

	/* Denominator is canceled by G1_Smith */
	/* bsdf = D * G / (4.0 * NL * NV); /* Reference function */
	return NL * a2 / (D * G); /* NL to Fit cycles Equation : line. 345 in bsdf_microfacet.h */
}
```
<br><br>

*effect_ssr_frag.glsl*
```
#else /* STEP_RESOLVE */

#define NUM_NEIGHBORS 4
void main(){
	...
	/* We generate the same rays that has been generated in the raycast step.
	 * But we add this ray from our resolve pixel position, increassing accuracy. */
	vec3 ssr_accum = vec3(0.0);
	float weight_acc = 0.0;
	float mask = 0.0;
	const ivec2 neighbors[7] = ivec2[7](ivec2(0, 0), ivec2(2, 1), ivec2(-2, 1), ivec2(0, -2), ivec2(-2, -1), ivec2(2, -1), ivec2(0, 2));
	int invert_neighbor = ((((fullres_texel.x & 0x1) + (fullres_texel.y & 0x1)) & 0x1) == 0) ? 1 : -1;
	for (int i = 0; i < NUM_NEIGHBORS; i++) {
		ivec2 target_texel = halfres_texel + neighbors[i] * invert_neighbor;

		float pdf = texelFetch(pdfBuffer, target_texel, 0).r;

		/* Check if there was a hit */
		if (pdf != 0.0) {
			vec2 hit_co = texelFetch(hitBuffer, target_texel, 0).rg;

			/* Reconstruct ray */
			float hit_depth = textureLod(depthBuffer, hit_co, 0.0).r;
			vec3 hit_pos = get_world_space_from_depth(hit_co, hit_depth);

			/* Find hit position in previous frame */
			vec2 ref_uvs = get_reprojected_reflection(hit_pos, worldPosition, N);

			/* Evaluate BSDF */
			vec3 L = normalize(hit_pos - worldPosition);
			float bsdf = bsdf_ggx(N, L, V, max(1e-3, roughness));

			float weight = bsdf / max(1e-8, pdf);
			mask += screen_border_mask(ref_uvs, hit_pos);
			ssr_accum += textureLod(colorBuffer, ref_uvs, 0.0).rgb * weight;
			weight_acc += weight;
		}
	}

	if (weight_acc > 0.0) {
		accumulate_light(ssr_accum / weight_acc, mask / float(NUM_NEIGHBORS), spec_accum);
	}
	...

	fragColor = vec4(spec_accum.rgb * speccol_roughness.rgb, 1.0);
}

#endif
```
>
- float weight = bsdf / max(1e-8, pdf); 利用 pdf 计算出 权重, 有点类似于 Ray tracing 的 蒙特卡罗的方式。
- ssr_accum += textureLod(colorBuffer, ref_uvs, 0.0).rgb * weight; 采样出colorBuffer的颜色作为高光，求和，之后再平均