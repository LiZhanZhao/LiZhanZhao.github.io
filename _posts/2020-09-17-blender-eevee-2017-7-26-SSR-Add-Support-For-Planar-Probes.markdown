---
layout:     post
title:      "blender eevee SSR Add support for planar probes."
subtitle:   ""
date:       2021-3-12 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/26  *   Eevee: SSR: Add support for planar probes.<br> 

> This add the possibility to use planar probe informations to create SSR.
This has 2 advantages:
- Tracing is less expensive since the hit is found much quicker.
- We have much less artifact due to missing information.
- There is still area for improvement.


> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
*带Planar Probes SSR效果*
![](/img/Eevee/SSR/08/1.png)

<br><br>

*不带Planar Probes SSR效果*
![](/img/Eevee/SSR/08/2.png)

## 作用
SSR 考虑 Planar Probes 的影响, 假如 Planar Probes 的效果感觉还可以

## 编译
- 重新生成SLN
- git 定位到  2017/7/26  *Eevee: SSR: Add support for planar probes.<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

<br><br>

## Shader




### 1. STEP_RAYTRACE

*effect_ssr_frag.glsl*
```
void main()
{
	...
	vec3 worldPosition = transform_point(ViewMatrixInverse, viewPosition);
	vec3 wN = mat3(ViewMatrixInverse) * N;

	/* Planar Reflections */
	for (int i = 0; i < MAX_PLANAR && i < planar_count; ++i) {
		PlanarData pd = planars_data[i];

		float fade = probe_attenuation_planar(pd, worldPosition, wN);

		if (fade > 0.5) {
			/* Find view vector / reflection plane intersection. */
			/* TODO optimize, use view space for all. */
			vec3 tracePosition = line_plane_intersect(worldPosition, cameraVec, pd.pl_plane_eq);			//计算出 reflection probe 上的交点
			tracePosition = transform_point(ViewMatrix, tracePosition);
			vec3 planeNormal = mat3(ViewMatrix) * pd.pl_normal;

			/* TODO : Raytrace together if textureGather is supported. */
			hitData0 = do_planar_ssr(i, V, N, planeNormal, tracePosition, a2, rand);
			if (rayCount > 1) hitData1 = do_planar_ssr(i, V, N, planeNormal, tracePosition, a2, rand.xyz * vec3(1.0, -1.0, -1.0));
			if (rayCount > 2) hitData2 = do_planar_ssr(i, V, N, planeNormal, tracePosition, a2, rand.xzy * vec3(1.0,  1.0, -1.0));
			if (rayCount > 3) hitData3 = do_planar_ssr(i, V, N, planeNormal, tracePosition, a2, rand.xzy * vec3(1.0, -1.0,  1.0));
			return;
		}
	}

	/* TODO : Raytrace together if textureGather is supported. */
	hitData0 = do_ssr(V, N, viewPosition, a2, rand);
	if (rayCount > 1) hitData1 = do_ssr(V, N, viewPosition, a2, rand.xyz * vec3(1.0, -1.0, -1.0));
	if (rayCount > 2) hitData2 = do_ssr(V, N, viewPosition, a2, rand.xzy * vec3(1.0,  1.0, -1.0));
	if (rayCount > 3) hitData3 = do_ssr(V, N, viewPosition, a2, rand.xzy * vec3(1.0, -1.0,  1.0));
	...
}
```
>
- 先考虑有没有 Planar Reflections Probe, 优先考虑 do_planar_ssr, 如果没有才考虑 do_ssr

<br><br>

#### a. probe_attenuation_planar
*lightprobe_lib.glsl*
```
float probe_attenuation_planar(PlanarData pd, vec3 W, vec3 N)
{
	/* Normal Facing */
	float fac = saturate(dot(pd.pl_normal, N) * pd.pl_facing_scale + pd.pl_facing_bias);		// 是否在plane的正面

	/* Distance from plane */
	fac *= saturate(abs(dot(pd.pl_plane_eq, vec4(W, 1.0))) * pd.pl_fade_scale + pd.pl_fade_bias);		//是否在plane的参数distance范围内

	/* Fancy fast clipping calculation */
	vec2 dist_to_clip;
	dist_to_clip.x = dot(pd.pl_clip_pos_x, W);
	dist_to_clip.y = dot(pd.pl_clip_pos_y, W);
	fac *= step(2.0, dot(step(pd.pl_clip_edges, dist_to_clip.xxyy), vec2(-1.0, 1.0).xyxy)); /* compare and add all tests */

	return fac;
}
```
>
- probe_attenuation_planar 判断物体是否在某一个planar的有效范围内，也就是 reflection probe 的编辑范围，在 reflection probe 编辑范围内的点才会进行考虑，例如 一个 平面在的 reflection probe 范围内，那些平面上的点才会考虑

<br><br>

#### b. line_plane_intersect

*eevee_lightprobes.c*
```
static void EEVEE_planar_reflections_updates(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, EEVEE_TextureList *txl)
{
	...
	/* Compute clip plane equation / normal. */
	float refpoint[3];
	copy_v3_v3(eplanar->plane_equation, ob->obmat[2]);
	normalize_v3(eplanar->plane_equation); /* plane normal */
	eplanar->plane_equation[3] = -dot_v3v3(eplanar->plane_equation, ob->obmat[3]);
	...
}
```
>
- 这里是计算传入shader中的参数 pd.pl_plane_eq, 平面方程的代数形式公式, 
- ax + by + cz + d = 0
- (a,b,c) -> 法线 -> ob->obmat[2] -> 向上的轴 Blender 是 Z 轴 
- d = -(a * x1 + b * y1 + c * z1) -> (x1, y1, z1) 就是 ob->obmat[3]
- 如果法线是归一化的，那么平面方程中的常数表达式 d 就是原点到平面的距离, 沿着平面法线

<br>

*bsdf_common_lib.glsl*
```
float line_plane_intersect_dist(vec3 lineorigin, vec3 linedirection, vec3 planeorigin, vec3 planenormal)
{
	return dot(planenormal, planeorigin - lineorigin) / dot(planenormal, linedirection);
}

float line_plane_intersect_dist(vec3 lineorigin, vec3 linedirection, vec4 plane)
{
	vec3 plane_co = plane.xyz * (-plane.w / len_squared(plane.xyz));
	vec3 h = lineorigin - plane_co;
	return -dot(plane.xyz, h) / dot(plane.xyz, linedirection);
}

vec3 line_plane_intersect(vec3 lineorigin, vec3 linedirection, vec3 planeorigin, vec3 planenormal)
{
	float dist = line_plane_intersect_dist(lineorigin, linedirection, planeorigin, planenormal);
	return lineorigin + linedirection * dist;
}

vec3 line_plane_intersect(vec3 lineorigin, vec3 linedirection, vec4 plane)
{
	float dist = line_plane_intersect_dist(lineorigin, linedirection, plane);
	return lineorigin + linedirection * dist;
}

```
>
- line_plane_intersect 的参数 plane 是平面的代数方程 ax + by + cz + d = 0
- line_plane_intersect 主要的目的就是计算 ray 和 plan 的交点, ray 就是 lineorigin + linedirection
- 上面的理论可以参考 [这里](https://en.wikipedia.org/wiki/Line%E2%80%93plane_intersection)
- line_plane_intersect 计算出来的就是 在 ray 在 plane 上的交点，world空间的
- vec3 tracePosition = line_plane_intersect(worldPosition, cameraVec, pd.pl_plane_eq); 那么这里假设 平面在 reflection probe 范围内，这里计算的就是，平面上的点作为ray的起点，cameraVec作为 ray 的方向，这条ray和 reflection probe进行相交，计算出 reflection probe 上的交点


#### c. do_planar_ssr

```
vec4 do_planar_ssr(int index, vec3 V, vec3 N, vec3 planeNormal, vec3 viewPosition, float a2, vec3 rand)
{
	float pdf;
	vec3 R = generate_ray(V, N, a2, rand, pdf);

	R = reflect(R, planeNormal);					// 这里是判断 R 经过 plane 反射 之后, 还是不是在plane 的 正面
	pdf *= -1.0; /* Tag as planar ray. */

	/* If ray is bad (i.e. going below the plane) do not trace. */
	if (dot(R, planeNormal) > 0.0) {
		vec3 R = generate_ray(V, N, a2, rand, pdf);
	}

	float hit_dist;
	if (abs(dot(-R, V)) < 0.9999) {						// 这里其实就是判断是否R 和 V 重合,节省计算
		hit_dist = raycast(index, viewPosition, R, rand.x);			// viewPosition 是planar probe 上的点, 沿着 R 可以碰到什么物体
	}
	else {
		float z = get_view_z_from_depth(texelFetch(planarDepth, ivec3(project_point(PixelProjMatrix, viewPosition).xy, index), 0).r);
		hit_dist = (z - viewPosition.z) / R.z;
	}

	/* Since viewspace hit position can land behind the camera in this case,
	 * we save the reflected view position (visualize it as the hit position
	 * below the reflection plane). This way it's garanted that the hit will
	 * be in front of the camera. That let us tag the bad rays with a negative
	 * sign in the Z component. */
	vec3 hit_pos = viewPosition + R * abs(hit_dist);

	/* Ray did not hit anything. No backface test because it's not possible
	 * to hit a backface in this case. */
	if (hit_dist <= 0.0) {
		hit_pos.z *= -1.0;
	}

	return vec4(hit_pos, pdf);
}
```
>
- viewPosition 参数是 reflection probe 上的点



### 2. STEP_RAYTRACE

*effect_ssr_frag.glsl*
```
void main()
{
	...
	for (int i = 0; i < NUM_NEIGHBORS; i++) {
		ivec2 target_texel = halfres_texel + neighbors[i] * invert_neighbor;

		ssr_accum += get_ssr_sample(hitBuffer0, pd, planar_index, worldPosition, N, V,
		                            roughnessSquared, cone_tan, source_uvs,
		                            texture_size, target_texel, weight_acc);
		if (rayCount > 1) {
			ssr_accum += get_ssr_sample(hitBuffer1, pd, planar_index, worldPosition, N, V,
			                            roughnessSquared, cone_tan, source_uvs,
			                            texture_size, target_texel, weight_acc);
		}
		if (rayCount > 2) {
			ssr_accum += get_ssr_sample(hitBuffer2, pd, planar_index, worldPosition, N, V,
			                            roughnessSquared, cone_tan, source_uvs,
			                            texture_size, target_texel, weight_acc);
		}
		if (rayCount > 3) {
			ssr_accum += get_ssr_sample(hitBuffer3, pd, planar_index, worldPosition, N, V,
			                            roughnessSquared, cone_tan, source_uvs,
			                            texture_size, target_texel, weight_acc);
		}
	}
	...
}
```

<br><br>


#### a. get_ssr_sample

```

vec4 get_ssr_sample(
        sampler2D hitBuffer, PlanarData pd, float planar_index, vec3 worldPosition, vec3 N, vec3 V, float roughnessSquared,
        float cone_tan, vec2 source_uvs, vec2 texture_size, ivec2 target_texel,
        inout float weight_acc)
{
	vec4 hit_co_pdf = texelFetch(hitBuffer, target_texel, 0).rgba;
	bool has_hit = (hit_co_pdf.z < 0.0);
	bool is_planar = (hit_co_pdf.w < 0.0);
	hit_co_pdf.z = -abs(hit_co_pdf.z);
	hit_co_pdf.w = abs(hit_co_pdf.w);

	/* Hit position in world space. */
	vec3 hit_pos = transform_point(ViewMatrixInverse, hit_co_pdf.xyz);

	vec2 ref_uvs;
	vec3 L;
	float mask = 1.0;
	float cone_footprint;
	if (is_planar) {
		/* Reflect back the hit position to have it in non-reflected world space */
		vec3 trace_pos = line_plane_intersect(worldPosition, V, pd.pl_plane_eq);
		vec3 hit_vec = hit_pos - trace_pos;
		hit_vec = reflect(hit_vec, pd.pl_normal);
		hit_pos = hit_vec + trace_pos;
		L = normalize(hit_vec);
		ref_uvs = project_point(ProjectionMatrix, hit_co_pdf.xyz).xy * 0.5 + 0.5;
		vec2 uvs = gl_FragCoord.xy / texture_size;

		/* Compute cone footprint in screen space. */
		float homcoord = ProjectionMatrix[2][3] * hit_co_pdf.z + ProjectionMatrix[3][3];
		cone_footprint = length(hit_vec) * cone_tan * ProjectionMatrix[0][0] / homcoord;
	}
	else {
		/* Find hit position in previous frame. */
		ref_uvs = get_reprojected_reflection(hit_pos, worldPosition, N);
		L = normalize(hit_pos - worldPosition);
		mask *= view_facing_mask(V, N);
		mask *= screen_border_mask(source_uvs);

		/* Compute cone footprint Using UV distance because we are using screen space filtering. */
		cone_footprint = 1.5 * cone_tan * distance(ref_uvs, source_uvs);
	}
	mask *= screen_border_mask(ref_uvs);
	mask *= float(has_hit);

	/* Estimate a cone footprint to sample a corresponding mipmap level. */
	float mip = BRDF_BIAS * clamp(log2(cone_footprint * max(texture_size.x, texture_size.y)), 0.0, MAX_MIP);

	/* Slide 54 */
	float bsdf = bsdf_ggx(N, L, V, roughnessSquared);
	float weight = step(0.001, hit_co_pdf.w) * bsdf / hit_co_pdf.w;
	weight_acc += weight;

	vec3 sample;
	if (is_planar) {
		sample = textureLod(probePlanars, vec3(ref_uvs, planar_index), mip).rgb;
	}
	else {
		sample = textureLod(colorBuffer, ref_uvs, mip).rgb;
	}

	/* Do not add light if ray has failed. */
	sample *= float(has_hit);

	/* Firefly removal */
	sample /= 1.0 + fireflyFactor * brightness(sample);

	return vec4(sample, mask) * weight;
}

```