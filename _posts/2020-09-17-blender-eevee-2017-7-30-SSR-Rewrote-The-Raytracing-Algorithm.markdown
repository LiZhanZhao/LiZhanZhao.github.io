---
layout:     post
title:      "blender eevee SSR Rewrote the raytracing algorithm."
subtitle:   ""
date:       2021-3-16 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/30  *   Eevee: SSR: Rewrote the raytracing algorithm.<br> 

> 
It now uses a quality slider instead of stride.
Lower quality takes larger strides between samples and use lower mips when tracing rough rays.

>
Now raytracing is done entierly in homogeneous coordinate space. This run much faster.
Should be fairly optimized. We are still Bandwidth bound.

>
Add a line-line intersection refine.
Add a ray jitter between the multiple ray per pixel to fill some undersampling in mirror reflections.

>
The tracing now stops if it goes behind an object. This needs some work to allow it to continue even if behind objects.


> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/SSR/09/1.png)


## 作用
重写SSR算法, 这里还有少少的瑕疵

## 编译
- 重新生成SLN
- git 定位到  2017/7/30  *Eevee: SSR: Rewrote the raytracing algorithm.<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## 1. STEP_RAYTRACE

### Shader
```

/* Based on Stochastic Screen Space Reflections
 * https://www.ea.com/frostbite/news/stochastic-screen-space-reflections */

#ifndef UTIL_TEX
#define UTIL_TEX
uniform sampler2DArray utilTex;
#endif /* UTIL_TEX */

uniform int rayCount;

#define BRDF_BIAS 0.7

// 利用 V 和 N , 计算 反射 射线 R, 就是通过 视角 + 法线 计算出 反射线
// 这里参数 V 是  normalize(-viewPosition), 那么 -V 就是 viewPosition, 刚好符合 reflect 对入射光方向的要求
vec3 generate_ray(vec3 V, vec3 N, float a2, vec3 rand, out float pdf)
{
	float NH;
	vec3 T, B;
	make_orthonormal_basis(N, T, B); /* Generate tangent space */

	/* Importance sampling bias */
	rand.x = mix(rand.x, 0.0, BRDF_BIAS);

	/* TODO distribution of visible normals */
	vec3 H = sample_ggx(rand, a2, N, T, B, NH); /* Microfacet normal */
	pdf = min(1024e32, pdf_ggx_reflect(NH, a2)); /* Theoretical limit of 16bit float */
	return reflect(-V, H);
}

#define MAX_MIP 9.0

#ifdef STEP_RAYTRACE

uniform sampler2D normalBuffer;
uniform sampler2D specroughBuffer;

uniform int planar_count;

layout(location = 0) out vec4 hitData0;
layout(location = 1) out vec4 hitData1;
layout(location = 2) out vec4 hitData2;
layout(location = 3) out vec4 hitData3;

bool has_hit_backface(vec3 hit_pos, vec3 R, vec3 V)
{
	vec2 hit_co = project_point(ProjectionMatrix, hit_pos).xy * 0.5 + 0.5;
	vec3 hit_N = normal_decode(textureLod(normalBuffer, hit_co, 0.0).rg, V);
	return (dot(-R, hit_N) < 0.0);
}

// 这里的 V 是 normalize(-viewPosition), 也就是物体点指向相机原点
vec4 do_planar_ssr(int index, vec3 V, vec3 N, vec3 planeNormal, vec3 viewPosition, float a2, vec3 rand, float ray_nbr)
{
	// 计算出 视线 + 点法线 的反射线 R, 这里留意一下 reflect 函数的 L 和 N 的方向，下面有说
	float pdf;
	vec3 R = generate_ray(V, N, a2, rand, pdf);

	// 由于reflect函数的 L 是指向position的, 这里的意思就是, R 和 planeNormal 的 形成的 反射向量 R1 是不是 跟 planeNormal 同向 (dot(R, planeNormal) > 0.0)
	// 如果 R1 跟 planeNormal 同向, 即 (dot(R, planeNormal) > 0.0), 表示的就是 R 和 plane Normal 反向 。
	R = reflect(R, planeNormal);
	pdf *= -1.0; /* Tag as planar ray. */

	// 这里就是计算出来的R 和  planeNormal  反向, 重新生成 R
	/* If ray is bad (i.e. going below the plane) do not trace. */
	if (dot(R, planeNormal) > 0.0) {
		vec3 R = generate_ray(V, N, a2, rand * vec3(1.0, -1.0, -1.0), pdf);
	}

	vec3 hit_pos;
	if (abs(dot(-R, V)) < 0.9999) {
		/* Since viewspace hit position can land behind the camera in this case,
		 * we save the reflected view position (visualize it as the hit position
		 * below the reflection plane). This way it's garanted that the hit will
		 * be in front of the camera. That let us tag the bad rays with a negative
		 * sign in the Z component. */
		hit_pos = raycast(index, viewPosition, R, fract(rand.x + (ray_nbr / float(rayCount))), a2);
	}
	else {
		vec2 uvs = project_point(ProjectionMatrix, viewPosition).xy * 0.5 + 0.5;
		float raw_depth = textureLod(planarDepth, vec3(uvs, float(index)), 0.0).r;
		hit_pos = get_view_space_from_depth(uvs, raw_depth);
		hit_pos.z *= (raw_depth < 1.0) ? 1.0 : -1.0;
	}

	return vec4(hit_pos, pdf);
}

// 这里的 V 是 normalize(-viewPosition), 也就是物体点只想相机远点
vec4 do_ssr(vec3 V, vec3 N, vec3 viewPosition, float a2, vec3 rand, float ray_nbr)
{
	float pdf;
	vec3 R = generate_ray(V, N, a2, rand, pdf);

	vec3 hit_pos = raycast(-1, viewPosition, R, fract(rand.x + (ray_nbr / float(rayCount))), a2);

	return vec4(hit_pos, pdf);
}

void main()
{
#ifdef FULLRES
	ivec2 fullres_texel = ivec2(gl_FragCoord.xy);
	ivec2 halfres_texel = fullres_texel;
#else
	ivec2 fullres_texel = ivec2(gl_FragCoord.xy) * 2;
	ivec2 halfres_texel = ivec2(gl_FragCoord.xy);
#endif

	float depth = texelFetch(depthBuffer, fullres_texel, 0).r;

	/* Early out */
	if (depth == 1.0)
		discard;

	vec2 uvs = gl_FragCoord.xy / vec2(textureSize(depthBuffer, 0));
#ifndef FULLRES
	uvs *= 2.0;
#endif

	/* Using view space */
	vec3 viewPosition = get_view_space_from_depth(uvs, depth);

	// #define viewCameraVec  ((ProjectionMatrix[3][3] == 0.0) ? normalize(-viewPosition) : vec3(0.0, 0.0, 1.0))
	vec3 V = viewCameraVec;

	// 这个N是view空间的
	vec3 N = normal_decode(texelFetch(normalBuffer, fullres_texel, 0).rg, V);

	/* Retrieve pixel data */
	vec4 speccol_roughness = texelFetch(specroughBuffer, fullres_texel, 0).rgba;

	/* Early out */
	if (dot(speccol_roughness.rgb, vec3(1.0)) == 0.0)
		discard;

	float roughness = speccol_roughness.a;
	float roughnessSquared = max(1e-3, roughness * roughness);
	float a2 = roughnessSquared * roughnessSquared;

	// 这里lod = 0, 第一个数组
	vec3 rand = texelFetch(utilTex, ivec3(halfres_texel % LUT_SIZE, 2), 0).rba;

	vec3 worldPosition = transform_point(ViewMatrixInverse, viewPosition);
	vec3 wN = transform_direction(ViewMatrixInverse, N);

	/* Planar Reflections */
	for (int i = 0; i < MAX_PLANAR && i < planar_count; ++i) {
		PlanarData pd = planars_data[i];

		float fade = probe_attenuation_planar(pd, worldPosition, wN, 0.0);

		if (fade > 0.5) {
			/* Find view vector / reflection plane intersection. */
			/* TODO optimize, use view space for all. */
			vec3 tracePosition = line_plane_intersect(worldPosition, cameraVec, pd.pl_plane_eq);
			tracePosition = transform_point(ViewMatrix, tracePosition);
			vec3 planeNormal = transform_direction(ViewMatrix, pd.pl_normal);

			/* TODO : Raytrace together if textureGather is supported. */
			hitData0 = do_planar_ssr(i, V, N, planeNormal, tracePosition, a2, rand, 0.0);
			if (rayCount > 1) hitData1 = do_planar_ssr(i, V, N, planeNormal, tracePosition, a2, rand.xyz * vec3(1.0, -1.0, -1.0), 1.0);
			if (rayCount > 2) hitData2 = do_planar_ssr(i, V, N, planeNormal, tracePosition, a2, rand.xzy * vec3(1.0,  1.0, -1.0), 2.0);
			if (rayCount > 3) hitData3 = do_planar_ssr(i, V, N, planeNormal, tracePosition, a2, rand.xzy * vec3(1.0, -1.0,  1.0), 3.0);
			return;
		}
	}

	/* TODO : Raytrace together if textureGather is supported. */
	hitData0 = do_ssr(V, N, viewPosition, a2, rand, 0.0);
	if (rayCount > 1) hitData1 = do_ssr(V, N, viewPosition, a2, rand.xyz * vec3(1.0, -1.0, -1.0), 1.0);
	if (rayCount > 2) hitData2 = do_ssr(V, N, viewPosition, a2, rand.xzy * vec3(1.0,  1.0, -1.0), 2.0);
	if (rayCount > 3) hitData3 = do_ssr(V, N, viewPosition, a2, rand.xzy * vec3(1.0, -1.0,  1.0), 3.0);
}

#else /* STEP_RESOLVE */
...
#endif

```

#### a. normalBuffer 和 specroughBuffer 的来源

- 可以看 [Output-SSR-Datas-To-Buffers](http://shaderstore.cn/2021/02/24/blender-eevee-2017-7-17-SSR-Output-SSR-Datas-To-Buffers/) 找到具体的信息
<br><br>
- 总结来说，normalBuffer 保存的是 view 空间的 normal, specroughBuffer 中 rgb 保存的是 计算的pbr高光结果, a 保存的是 rounghness

#### b. reflect 函数
- reflect 的实现
```
float3 reflect( float3 i, float3 n )
{
  return i - 2.0 * n * dot(n,i);
}
```
- ![](/img/Eevee/SSR/09/2.png)
- 上面的就是reflection的参数的方向， L 的方向是 指向 Postion 的
- 可以参考[这里](https://zhuanlan.zhihu.com/p/152561125)


#### c. utilTex
- utilTex 是一个 TextureArray , 由3个 64 * 64 的元素组成
- 第一个数组保存了的是 ltc_mat_ggx [-1, 1]
```
void EEVEE_materials_init(EEVEE_StorageList *stl)
{
	...
	/* Textures */
	const int layers = 3;
	float (*texels)[4] = MEM_mallocN(sizeof(float[4]) * 64 * 64 * layers, "utils texels");
	float (*texels_layer)[4] = texels;

	/* Copy ltc_mat_ggx into 1st layer */
	memcpy(texels_layer, ltc_mat_ggx, sizeof(float[4]) * 64 * 64);
	texels_layer += 64 * 64;

	/* Copy bsdf_split_sum_ggx into 2nd layer red and green channels.
		Copy ltc_mag_ggx into 2nd layer blue channel. */
	for (int i = 0; i < 64 * 64; i++) {
		texels_layer[i][0] = bsdf_split_sum_ggx[i*2 + 0];
		texels_layer[i][1] = bsdf_split_sum_ggx[i*2 + 1];
		texels_layer[i][2] = ltc_mag_ggx[i];
	}
	texels_layer += 64 * 64;

	for (int i = 0; i < 64 * 64; i++) {
		texels_layer[i][0] = blue_noise[i][0];
		texels_layer[i][1] = blue_noise[i][1] * 0.5 + 0.5;
		texels_layer[i][2] = cosf(blue_noise[i][1] * 2.0 * M_PI);
		texels_layer[i][3] = sinf(blue_noise[i][1] * 2.0 * M_PI);
	}

	e_data.util_tex = DRW_texture_create_2D_array(64, 64, layers, DRW_TEX_RGBA_16, DRW_TEX_FILTER | DRW_TEX_WRAP, (float *)texels);
	MEM_freeN(texels);
	...
}
```

#### d. Planar Reflections SSR
- 可以看 [SSR-Add-Support-For-Planar-Probes](http://shaderstore.cn/2021/03/12/blender-eevee-2017-7-26-SSR-Add-Support-For-Planar-Probes/) 找到具体的信息
<br><br>
- 总结来说, 因为SSR是后处理效果, 所有的一切都是基本屏幕的, 利用屏幕的深度信息可以计算出每一个像素对应的WorldPosition 和 WorldNormal, 因为 Planar Reflections Probe 自身是保存了一个平面方程等一些信息，那么 利用 WorldPosition 和 WorldNormal 就可以判断出哪一些 屏幕的像素是 在  Planar Reflections Probe 的有效范围内
<br><br>
- 知道哪一些 屏幕的像素 在 Planar Reflections Probe 范围内, 那么就利用像素的 WorldPosition 和 cameraVec, cameraVec 是 WorldPosition 指向 CameraWorldPostion ,组成一个ray, 这个ray 和  Planar Reflections Probe 的 平面方程进行相交，计算出一个交点 tracePosition.
<br><br>


#### e. raycast
```
#define MAX_STEP 256
#define MAX_REFINE_STEP 32 /* Should be max allowed stride */

uniform vec4 ssrParameters;

uniform sampler2D depthBuffer;
uniform sampler2D maxzBuffer;
uniform sampler2D minzBuffer;
uniform sampler2DArray planarDepth;

#define ssrQuality    ssrParameters.x
#define ssrThickness  ssrParameters.y
#define ssrPixelSize  ssrParameters.zw

float sample_depth(vec2 uv, int index, float lod)
{
	if (index > -1) {
		return textureLod(planarDepth, vec3(uv, index), 0.0).r;
	}
	else {
		return textureLod(maxzBuffer, uv, lod).r;
	}
}

float sample_minz_depth(vec2 uv, int index)
{
	if (index > -1) {
		return textureLod(planarDepth, vec3(uv, index), 0.0).r;
	}
	else {
		return textureLod(minzBuffer, uv, 0.0).r;
	}
}

float sample_maxz_depth(vec2 uv, int index)
{
	if (index > -1) {
		return textureLod(planarDepth, vec3(uv, index), 0.0).r;
	}
	else {
		return textureLod(maxzBuffer, uv, 0.0).r;
	}
}

vec4 sample_depth_grouped(vec4 uv1, vec4 uv2, int index, float lod)
{
	vec4 depths;
	if (index > -1) {
		depths.x = textureLod(planarDepth, vec3(uv1.xy, index), 0.0).r;
		depths.y = textureLod(planarDepth, vec3(uv1.zw, index), 0.0).r;
		depths.z = textureLod(planarDepth, vec3(uv2.xy, index), 0.0).r;
		depths.w = textureLod(planarDepth, vec3(uv2.zw, index), 0.0).r;
	}
	else {
		depths.x = textureLod(maxzBuffer, uv1.xy, lod).r;
		depths.y = textureLod(maxzBuffer, uv1.zw, lod).r;
		depths.z = textureLod(maxzBuffer, uv2.xy, lod).r;
		depths.w = textureLod(maxzBuffer, uv2.zw, lod).r;
	}
	return depths;
}

float refine_isect(float prev_delta, float curr_delta)
{
	/**
	 * Simplification of 2D intersection :
	 * r0 = (0.0, prev_ss_ray.z);
	 * r1 = (1.0, curr_ss_ray.z);
	 * d0 = (0.0, prev_hit_depth_sample);
	 * d1 = (1.0, curr_hit_depth_sample);
	 * vec2 r = r1 - r0;
	 * vec2 d = d1 - d0;
	 * vec2 isect = ((d * cross(r1, r0)) - (r * cross(d1, d0))) / cross(r,d);
	 *
	 * We only want isect.x to know how much stride we need. So it simplifies :
	 *
	 * isect_x = (cross(r1, r0) - cross(d1, d0)) / cross(r,d);
	 * isect_x = (prev_ss_ray.z - prev_hit_depth_sample.z) / cross(r,d);
	 */
	return saturate(prev_delta / (prev_delta - curr_delta));
}

void prepare_raycast(vec3 ray_origin, vec3 ray_dir, out vec4 ss_step, out vec4 ss_ray, out float max_time)
{
	/* Negate the ray direction if it goes towards the camera.
	 * This way we don't need to care if the projected point
	 * is behind the near plane. */
	 
	 // 如果 ray_dir.z为负数，z_sign 为 1，那么 z_sign * ray_dir =  ray_dir
	 // 如果 ray_dir.z为正数, z_sign 为 -1, 那么 z_sign * ray_dir = - ray_dir

	float z_sign = -sign(ray_dir.z);
	vec3 ray_end = z_sign * ray_dir * 1e16 + ray_origin;

	/* Project into screen space. */
	vec3 ss_start = project_point(ProjectionMatrix, ray_origin);
	vec3 ss_end = project_point(ProjectionMatrix, ray_end);
	/* 4th component is current stride */
	ss_step = vec4(z_sign * normalize(ss_end - ss_start), 1.0);

	/* If the line is degenerate, make it cover at least one pixel
	 * to not have to handle zero-pixel extent as a special case later */
	ss_step.xy += vec2((dot(ss_step.xy, ss_step.xy) < 0.00001) ? 0.001 : 0.0);

	/* Make ss_step cover one pixel. */
	ss_step.xyz /= max(abs(ss_step.x), abs(ss_step.y));
	ss_step.xyz *= ((abs(ss_step.x) > abs(ss_step.y)) ? ssrPixelSize.x : ssrPixelSize.y);

	/* Clipping to frustum sides. */
	max_time = line_unit_box_intersect_dist(ss_start, ss_step.xyz) - 1.0;

	/* Convert to texture coords. Z component included
	 * since this is how it's stored in the depth buffer.
	 * 4th component how far we are on the ray */
	ss_ray = vec4(ss_start * 0.5 + 0.5, 0.0);
	ss_step.xyz *= 0.5;
}

/* See times_and_deltas. */
#define curr_time   times_and_deltas.x
#define prev_time   times_and_deltas.y
#define curr_delta  times_and_deltas.z
#define prev_delta  times_and_deltas.w

// #define GROUPED_FETCHES
/* Return the hit position, and negate the z component (making it positive) if not hit occured. */
vec3 raycast(int index, vec3 ray_origin, vec3 ray_dir, float ray_jitter, float roughness)
{
	vec4 ss_step, ss_start;
	float max_time;
	prepare_raycast(ray_origin, ray_dir, ss_step, ss_start, max_time);

#ifdef GROUPED_FETCHES
	ray_jitter *= 0.25;
#endif
	/* x : current_time, y: previous_time, z: previous_delta, w: current_delta */
	vec4 times_and_deltas = vec4(0.0, 0.0, 0.001, 0.001);

	float ray_time = 0.0;
	float depth_sample;

	float lod_fac = saturate(fast_sqrt(roughness) * 2.0 - 0.4);
	bool hit = false;
	float iter;
	for (iter = 1.0; !hit && (ray_time <= max_time) && (iter < MAX_STEP); iter++) {
		/* Minimum stride of 2 because we are using half res minmax zbuffer. */
		float stride = max(1.0, iter * ssrQuality) * 2.0;
		float lod = log2(stride * 0.5 * ssrQuality) * lod_fac;

		/* Save previous values. */
		times_and_deltas.xyzw = times_and_deltas.yxwz;

#ifdef GROUPED_FETCHES
		stride *= 4.0;
		vec4 jit_stride = mix(vec4(2.0), vec4(stride), vec4(0.0, 0.25, 0.5, 0.75) + ray_jitter);

		vec4 times = vec4(ray_time) + jit_stride;

		vec4 uv1 = ss_start.xyxy + ss_step.xyxy * times.xxyy;
		vec4 uv2 = ss_start.xyxy + ss_step.xyxy * times.zzww;

		vec4 depth_samples = sample_depth_grouped(uv1, uv2, index, lod);

		vec4 ray_z = ss_start.zzzz + ss_step.zzzz * times.xyzw;

		vec4 deltas = depth_samples - ray_z;
		/* Same as component wise (depth_samples <= ray_z) && (ray_time <= max_time). */
		bvec4 test = equal(step(deltas, vec4(0.0)) * step(times, vec4(max_time)), vec4(1.0));
		hit = any(test);
		if (hit) {
			vec2 m = vec2(1.0, 0.0); /* Mask */

			vec4 ret_times_and_deltas = times.wzzz * m.xxyy + deltas.wwwz * m.yyxx;
			ret_times_and_deltas      = (test.z) ? times.zyyy * m.xxyy + deltas.zzzy * m.yyxx : ret_times_and_deltas;
			ret_times_and_deltas      = (test.y) ? times.yxxx * m.xxyy + deltas.yyyx * m.yyxx : ret_times_and_deltas;
			times_and_deltas          = (test.x) ? times.xxxx * m.xyyy + deltas.xxxx * m.yyxy + times_and_deltas.yyww * m.yxyx : ret_times_and_deltas;

			depth_sample = depth_samples.w;
			depth_sample = (test.z) ? depth_samples.z : depth_sample;
			depth_sample = (test.y) ? depth_samples.y : depth_sample;
			depth_sample = (test.x) ? depth_samples.x : depth_sample;
			break;
		}
		curr_time = times.w;
		curr_delta = deltas.w;
		ray_time += stride;
#else
		float jit_stride = mix(2.0, stride, ray_jitter);

		curr_time = ray_time + jit_stride;
		vec4 ss_ray = ss_start + ss_step * curr_time;

		depth_sample = sample_depth(ss_ray.xy, index, lod);

		curr_delta = depth_sample - ss_ray.z;
		hit = (curr_delta <= 0.0) && (curr_time <= max_time);

		ray_time += stride;
#endif
	}

	curr_time = (hit) ? mix(prev_time, curr_time, refine_isect(prev_delta, curr_delta)) : curr_time;
	ray_time = (hit) ? curr_time : ray_time;

#if 0 /* Not needed if using refine_isect() */
	/* Binary search */
	for (float time_step = (curr_time - prev_time) * 0.5; time_step > 1.0; time_step /= 2.0) {
		ray_time -= time_step;
		vec4 ss_ray = ss_start + ss_step * ray_time;
		float depth_sample = sample_maxz_depth(ss_ray.xy, index);
		bool is_hit = (depth_sample - ss_ray.z <= 0.0);
		ray_time = (is_hit) ? ray_time : ray_time + time_step;
	}
#endif

	/* Clip to frustum. */
	ray_time = min(ray_time, max_time - 0.5);

	vec4 ss_ray = ss_start + ss_step * ray_time;
	vec3 hit_pos = get_view_space_from_depth(ss_ray.xy, ss_ray.z);

	/* Reject hit if not within threshold. */
	/* TODO do this check while tracing. Potentially higher quality */
	if (hit && (index == -1)) {
		float z = get_view_z_from_depth(depth_sample);
		hit = hit && ((z - hit_pos.z - ssrThickness) <= ssrThickness);
	}

	/* Tag Z if ray failed. */
	hit_pos.z *= (hit) ? 1.0 : -1.0;
	return hit_pos;
}

```
