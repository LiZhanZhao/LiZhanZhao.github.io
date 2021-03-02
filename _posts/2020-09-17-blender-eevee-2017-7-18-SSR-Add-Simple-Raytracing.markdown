---
layout:     post
title:      "blender eevee SSR Add simple raytracing."
subtitle:   ""
date:       2021-3-1 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/18  *  Eevee: SSR: Add simple raytracing .<br> 

> Still imprecise

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果

![](/img/Eevee/SSR/02/1.png)


## 作用
利用 Screen Space Ray tracing 实现 SSR

## 编译
- 重新生成SLN
- git 定位到  2017/7/18  * Eevee: SSR: Add simple raytracing .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 前提
阅读本文，请先看[1](http://localhost:4000/2021/02/24/blender-eevee-2017-7-17-SSR-Output-SSR-Datas-To-Buffers/)和
[2](http://localhost:4000/2021/02/27/blender-eevee-2017-7-10-SSR-Encode-Normal-In-Buffer-And-Add-Cubemap-Fallback/)

## Shader

### 1. STEP_RAYTRACE
*effect_ssr_frag.glsl*
```
vec3 generate_ray(vec3 V, vec3 N)
{
	return -reflect(-V, N);
}

#ifdef STEP_RAYTRACE

uniform sampler2D depthBuffer;
uniform sampler2D normalBuffer;
uniform sampler2D specroughBuffer;

layout(location = 0) out vec4 hitData;
layout(location = 1) out vec4 pdfData;

void main()
{
	ivec2 fullres_texel = ivec2(gl_FragCoord.xy) * 2;
	ivec2 halfres_texel = ivec2(gl_FragCoord.xy);
	float depth = texelFetch(depthBuffer, halfres_texel, 0).r;

	/* Early discard */
	if (depth == 1.0)
		discard;

	vec2 uvs = gl_FragCoord.xy / vec2(textureSize(depthBuffer, 0));

	/* Using view space */
	vec3 viewPosition = get_view_space_from_depth(uvs, depth);
	vec3 V = viewCameraVec;
	vec3 N = normal_decode(texelFetch(normalBuffer, fullres_texel, 0).rg, V);

	/* Retrieve pixel data */
	vec4 speccol_roughness = texelFetch(specroughBuffer, fullres_texel, 0).rgba;
	float roughness = speccol_roughness.a;
	float roughnessSquared = roughness * roughness;

	/* Generate Ray */
	vec3 R = generate_ray(V, N);

	/* Search for the planar reflection affecting this pixel */
	/* If no planar is found, fallback to screen space */

	/* Raycast over planar reflection */
	/* Potentially lots of time waisted here for pixels
	 * that does not have planar reflections. TODO Profile it. */
	/* TODO: Idea, rasterize boxes around planar
	 * reflection volumes (frontface culling to avoid overdraw)
	 * and do the raycasting, discard pixel that are not in influence.
	 * Add stencil test to discard the main SSR.
	 * Cons: - Potentially raytrace multiple times
	 *         if Planar Influence overlaps. */
	//float hit_dist = raycast(depthBuffer, W, R);

	/* Raycast over screen */
	float hit = raycast(depthBuffer, viewPosition, R);

	hitData = vec4(hit, hit, hit, 1.0);
	pdfData = vec4(0.5);
}

#else /* STEP_RESOLVE */
...

#endif

```
>
- vec3 V = viewCameraVec;  在bsdf_common_lib.glsl种可以找到 <br>
#define viewCameraVec  ((ProjectionMatrix[3][3] == 0.0) ? normalize(viewPosition) : vec3(0.0, 0.0, -1.0)), <br>
那么这里就是 viewCameraVec 就是相机空间 相机->物体点 的向量。
- 这里最最最重要的，其实就是 raycast 函数，raycast函数定义在 raytrace_lib.glsl 中。





<br><br>
### 2. RayTrace
*raytrace_lib.glsl*
```
/* Based on work from Morgan McGuire and Michael Mara at Williams College 2014
 * Released as open source under the BSD 2-Clause License
 * http://opensource.org/licenses/BSD-2-Clause
 * http://casual-effects.blogspot.fr/2014/08/screen-space-ray-tracing.html */

#define MAX_STEP 256

uniform mat4 PixelProjMatrix; /* View > NDC > Texel : maps view coords to texel coord */

void swapIfBigger(inout float a, inout float b)
{
	if (a > b) {
		float temp = a;
		a = b;
		b = temp;
	}
}

/* Return the length of the ray if there is a hit, and -1.0 if not hit occured */
float raycast(sampler2D depth_texture, vec3 ray_origin, vec3 ray_dir)
{
	float near = get_view_z_from_depth(0.0); /* TODO optimize */

	/* Clip ray to a near plane in 3D */
	float ray_length = 1e16;
	if ((ray_origin.z + ray_dir.z * ray_length) > near)
		ray_length = (near - ray_origin.z) / ray_dir.z;

	vec3 ray_end = ray_dir * ray_length + ray_origin;

	/* Project into screen space */	//	PixelProjMatrix 矩阵是 把ViewPos -> NDC -> Screen UV -> Screen Pixels
	vec4 H0 = PixelProjMatrix * vec4(ray_origin, 1.0);
	vec4 H1 = PixelProjMatrix * vec4(ray_end, 1.0);

	/* There are a lot of divisions by w that can be turned into multiplications
	* at some minor precision loss...and we need to interpolate these 1/w values
	* anyway. */
	float k0 = 1.0 / H0.w;
	float k1 = 1.0 / H1.w;

	/* Switch the original points to values that interpolate linearly in 2D */	//为什么要/w, 原因就是透视矫正插值 
	vec3 Q0 = ray_origin * k0;
	vec3 Q1 = ray_end * k1;

	/* Screen-space endpoints */		//P0,P1 计算出来的是屏幕空间的像素坐标[0, with], [0, height]
	vec2 P0 = H0.xy * k0;
	vec2 P1 = H1.xy * k1;

	/* [Optional clipping to frustum sides here] */

	/* Initialize to off screen */
	vec2 hitpixel = vec2(-1.0, -1.0);

	/* If the line is degenerate, make it cover at least one pixel
	 * to not have to handle zero-pixel extent as a special case later */
	P1 += vec2((distance_squared(P0, P1) < 0.0001) ? 0.01 : 0.0);

	vec2 delta = P1 - P0;

	/* Permute so that the primary iteration is in x to reduce large branches later.
	 * After this, "x" is the primary iteration direction and "y" is the secondary one
	 * If it is a more-vertical line, create a permutation that swaps x and y in the output
	 * and directly swizzle the inputs. */
	bool permute = false;
	if (abs(delta.x) < abs(delta.y)) {
		permute = true;
		delta = delta.yx;
		P1 = P1.yx;
		P0 = P0.yx;
	}

	/* Track the derivatives */
	float step_sign = sign(delta.x);
	float invdx = step_sign / delta.x;
	vec2 dP = vec2(step_sign, invdx * delta.y);
	vec3 dQ = (Q1 - Q0) * invdx;
	float dk = (k1 - k0) * invdx;

	/* Slide each value from the start of the ray to the end */
	vec4 pqk = vec4(P0, Q0.z, k0);				// 这个其实是开始点

	/* Scale derivatives by the desired pixel stride */
	vec4 dPQK = vec4(dP, dQ.z, dk) * 4.0;		// 这里其实就是步长

	/* We track the ray depth at +/- 1/2 pixel to treat pixels as clip-space solid
	 * voxels. Because the depth at -1/2 for a given pixel will be the same as at
	 * +1/2 for the previous iteration, we actually only have to compute one value
	 * per iteration. */
	float prev_zmax = ray_origin.z;
	float zmax, zmin;

	/* P1.x is never modified after this point, so pre-scale it by
	 * the step direction for a signed comparison */
	float end = P1.x * step_sign;

	bool hit = false;
	float hitstep, refinestep, raw_depth, view_depth;
	for (hitstep = 0.0; hitstep < MAX_STEP && !hit; hitstep++) {
		/* Ray finished & no hit*/
		if ((pqk.x * step_sign) > end) break;

		/* step through current cell */
		pqk += dPQK;

		hitpixel = permute ? pqk.yx : pqk.xy;
		zmin = prev_zmax;
		zmax = (dPQK.z * 0.5 + pqk.z) / (dPQK.w * 0.5 + pqk.w);
		prev_zmax = zmax;
		swapIfBigger(zmin, zmax);

		raw_depth = texelFetch(depth_texture, ivec2(hitpixel), 0).r;
		view_depth = get_view_z_from_depth(raw_depth);

		if (zmax < view_depth) {
			/* Below surface, cannot trace further */
			hit = true;
		}
	}

	if (hit) {
		/* Rewind back a step */
		pqk -= dPQK;

		/* And do a finer trace over this segment */
		dPQK /= 4.0;

		for (refinestep = 0.0; refinestep < 4.0; refinestep++) {
			/* step through current cell */
			pqk += dPQK;

			hitpixel = permute ? pqk.yx : pqk.xy;
			zmin = prev_zmax;
			zmax = (dPQK.z * 0.5 + pqk.z) / (dPQK.w * 0.5 + pqk.w);
			prev_zmax = zmax;
			swapIfBigger(zmin, zmax);

			raw_depth = texelFetch(depth_texture, ivec2(hitpixel), 0).r;
			view_depth = get_view_z_from_depth(raw_depth);

			if (zmax < view_depth) {
				/* Below surface, cannot trace further */
				break;
			}
		}
	}

	/* Check if we are somewhere near the surface. */
	/* TODO user threshold */
	float threshold = 0.05 / pqk.w; /* In view space */
	if (zmax < (view_depth - threshold)) {
		hit = false;
	}

	/* Check failure cases (out of screen, hit background) */
	if (hit && (raw_depth != 1.0) && (raw_depth != 0.0)) {
		/* Return length */
		return (zmax - ray_origin.z) / ray_dir.z;
	}

	/* Failure, return no hit */
	return -1.0;
}

```
>
- 理论参考在[这里](http://casual-effects.blogspot.com/2014/08/screen-space-ray-tracing.html)
- 一些细节的在注释里面
- 透视矫正插值 可以参考 [这里](https://zhuanlan.zhihu.com/p/105639446), 结论就是
- 如果是在 屏幕空间 进行插值计算，需要插件计算的属性都需要乘以 Z 的倒数才能是线性插值。

<br>
<br>

### 3. PixelProjMatrix

*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	/* Compute pixel projection matrix */
	{
		float uvpix[4][4], ndcuv[4][4], tmp[4][4], winmat[4][4];
		DRW_viewport_matrix_get(winmat, DRW_MAT_WIN);

		/* NDC to UVs */
		unit_m4(ndcuv);
		ndcuv[0][0] = ndcuv[1][1] = ndcuv[3][0] = ndcuv[3][1] = 0.5f;

		/* UVs to pixels */
		unit_m4(uvpix);
		uvpix[0][0] = tracing_res[0];
		uvpix[1][1] = tracing_res[1];

		mul_m4_m4m4(tmp, uvpix, ndcuv);
		mul_m4_m4m4(e_data.pixelprojmat, tmp, winmat);
	}
	...
}

```
>
- PixelProjMatrix 矩阵是 把ViewPos -> NDC -> Screen UV -> Screen Pixels .
- 大多数都是处理了 NDC-> Screen UV -> Screen Pixels 是处理了 X, Y 坐标.





<br><br>

### 4. STEP_RESOLVE
*effect_ssr_frag.glsl*
```
vec3 generate_ray(vec3 V, vec3 N)
{
	return -reflect(-V, N);
}

#ifdef STEP_RAYTRACE

...

#else /* STEP_RESOLVE */

uniform sampler2D depthBuffer;
uniform sampler2D normalBuffer;
uniform sampler2D specroughBuffer;

uniform sampler2D hitBuffer;
uniform sampler2D pdfBuffer;

uniform int probe_count;

uniform mat4 ViewProjectionMatrix;

out vec4 fragColor;

void fallback_cubemap(vec3 N, vec3 V, vec3 W, float roughness, float roughnessSquared, inout vec4 spec_accum)
{
	/* Specular probes */
	vec3 spec_dir = get_specular_dominant_dir(N, V, roughnessSquared);

	/* Starts at 1 because 0 is world probe */
	for (int i = 1; i < MAX_PROBE && i < probe_count && spec_accum.a < 0.999; ++i) {
		CubeData cd = probes_data[i];

		float fade = probe_attenuation_cube(cd, W);

		if (fade > 0.0) {
			vec3 spec = probe_evaluate_cube(float(i), cd, W, spec_dir, roughness);
			accumulate_light(spec, fade, spec_accum);
		}
	}

	/* World Specular */
	if (spec_accum.a < 0.999) {
		vec3 spec = probe_evaluate_world_spec(spec_dir, roughness);
		accumulate_light(spec, 1.0, spec_accum);
	}
}

void main()
{
	ivec2 halfres_texel = ivec2(gl_FragCoord.xy / 2.0);
	ivec2 fullres_texel = ivec2(gl_FragCoord.xy);
	vec2 uvs = gl_FragCoord.xy / vec2(textureSize(depthBuffer, 0));

	float depth = textureLod(depthBuffer, uvs, 0.0).r;

	/* Early discard */
	if (depth == 1.0)
		discard;

	/* Using world space */
	vec3 worldPosition = get_world_space_from_depth(uvs, depth);
	vec3 V = cameraVec;
	vec3 N = mat3(ViewMatrixInverse) * normal_decode(texelFetch(normalBuffer, fullres_texel, 0).rg, V);
	vec4 speccol_roughness = texelFetch(specroughBuffer, fullres_texel, 0).rgba;
	float roughness = speccol_roughness.a;
	float roughnessSquared = roughness * roughness;

	vec4 spec_accum = vec4(0.0);

	/* Resolve SSR and compute contribution */

	/* We generate the same rays that has been genearted in the raycast step.
	 * But we add this ray from our resolve pixel position, increassing accuracy. */
	vec3 R = generate_ray(-V, N);
	float ray_length = texelFetch(hitBuffer, halfres_texel, 0).r;

	if (ray_length != -1.0) {
		vec3 hit_pos = worldPosition + R * ray_length;
		vec4 hit_co = ViewProjectionMatrix * vec4(hit_pos, 1.0);
		spec_accum.xy = (hit_co.xy / hit_co.w) * 0.5 + 0.5;
		spec_accum.xyz = vec3(mod(dot(floor(hit_pos.xyz * 5.0), vec3(1.0)), 2.0));
		spec_accum.a = 1.0;
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
- 由于在 STEP_RAYTRACE 的时候，输出了 raycast(depthBuffer, viewPosition, R); 的结果，如果没有hit到物体的话，就是返回的是-1，如果hit到物体的话，返回的是ray的长度
- 这里比较简单，主要
```
float ray_length = texelFetch(hitBuffer, halfres_texel, 0).r;
if (ray_length != -1.0) {
	vec3 hit_pos = worldPosition + R * ray_length;
	vec4 hit_co = ViewProjectionMatrix * vec4(hit_pos, 1.0);
	spec_accum.xy = (hit_co.xy / hit_co.w) * 0.5 + 0.5;
	spec_accum.xyz = vec3(mod(dot(floor(hit_pos.xyz * 5.0), vec3(1.0)), 2.0));
	spec_accum.a = 1.0;
}
```
这里就是直接debug，为了验证是否hit到物体。