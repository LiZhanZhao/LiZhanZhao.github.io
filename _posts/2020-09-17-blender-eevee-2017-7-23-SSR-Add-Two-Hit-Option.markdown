---
layout:     post
title:      "blender eevee SSR Add two hit option"
subtitle:   ""
date:       2021-3-10 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/23  *   Eevee: SSR: Add two hit option.<br> 

> This option add another raytrace per pixel, clearing some noise. But multiplying the raytrace cost.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/SSR/06/1.png)


## 作用
给SSR添加参数 Double Trace 参数

## 编译
- 重新生成SLN
- git 定位到  2017/7/23  *Eevee: SSR: Add two hit option.<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## 渲染前

*eevee_effects.c*
```

static struct GPUShader *eevee_effects_ssr_shader_get(int options)
{
	if (e_data.ssr_sh[options] == NULL) {
		DynStr *ds_frag = BLI_dynstr_new();
		BLI_dynstr_append(ds_frag, datatoc_bsdf_common_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_bsdf_sampling_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_octahedron_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_lightprobe_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_raytrace_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_effect_ssr_frag_glsl);
		char *ssr_shader_str = BLI_dynstr_get_cstring(ds_frag);
		BLI_dynstr_free(ds_frag);

		DynStr *ds_defines = BLI_dynstr_new();
		BLI_dynstr_appendf(ds_defines, SHADER_DEFINES);
		if (options & SSR_RESOLVE) {
			BLI_dynstr_appendf(ds_defines, "#define STEP_RESOLVE\n");
		}
		else {
			BLI_dynstr_appendf(ds_defines, "#define STEP_RAYTRACE\n");
		}
		if (options & SSR_TWO_HIT) {
			BLI_dynstr_appendf(ds_defines, "#define TWO_HIT\n");
		}
		if (options & SSR_FULL_TRACE) {
			BLI_dynstr_appendf(ds_defines, "#define FULLRES\n");
		}
		if (options & SSR_NORMALIZE) {
			BLI_dynstr_appendf(ds_defines, "#define USE_NORMALIZATION\n");
		}
		char *ssr_define_str = BLI_dynstr_get_cstring(ds_defines);
		BLI_dynstr_free(ds_defines);

		e_data.ssr_sh[options] = DRW_shader_create_fullscreen(ssr_shader_str, ssr_define_str);

		MEM_freeN(ssr_shader_str);
		MEM_freeN(ssr_define_str);
	}

	return e_data.ssr_sh[options];
}

void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	if ((effects->enabled_effects & EFFECT_SSR) != 0) {
		int options = 0;
		options |= (effects->ssr_use_two_hit) ? SSR_TWO_HIT : 0;
		options |= (effects->reflection_trace_full) ? SSR_FULL_TRACE : 0;
		options |= (effects->ssr_use_normalization) ? SSR_NORMALIZE : 0;

		struct GPUShader *trace_shader = eevee_effects_ssr_shader_get(options);
		struct GPUShader *resolve_shader = eevee_effects_ssr_shader_get(SSR_RESOLVE | options);
		...
	}
}
```
>
- 目前的Shader会根据option进行开启宏
- 如果选项和宏一一对应
- 选项Double Trace 对应宏 TWO_HIT
- Half Res Trace 对应 宏 FULLRES
- trace_shader 一定会开启 STEP_RAYTRACE
- resolve_shader 一定会开启 STEP_RESOLVE



## Shader
*effect_ssr_frag.glsl*
```

#ifndef UTIL_TEX
#define UTIL_TEX
uniform sampler2DArray utilTex;
#endif /* UTIL_TEX */

#define BRDF_BIAS 0.7

vec3 generate_ray(vec3 V, vec3 N, float a2, vec3 rand, out float pdf)
{
	float NH;
	vec3 T, B;
	make_orthonormal_basis(N, T, B); /* Generate tangent space */

	/* Importance sampling bias */
	rand.x = mix(rand.x, 0.0, BRDF_BIAS);

	vec3 H = sample_ggx(rand, a2, N, T, B, NH); /* Microfacet normal */
	pdf = min(1024e32, pdf_ggx_reflect(NH, a2)); /* Theoretical limit of 16bit float */
	return reflect(-V, H);
}

#define MAX_MIP 9.0

#ifdef STEP_RAYTRACE

uniform sampler2D depthBuffer;
uniform sampler2D normalBuffer;
uniform sampler2D specroughBuffer;

layout(location = 0) out vec4 hitData;
layout(location = 1) out vec4 pdfData;

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
	vec3 V = viewCameraVec;
	vec3 N = normal_decode(texelFetch(normalBuffer, fullres_texel, 0).rg, V);

	/* Retrieve pixel data */
	vec4 speccol_roughness = texelFetch(specroughBuffer, fullres_texel, 0).rgba;

	/* Early out */
	if (dot(speccol_roughness.rgb, vec3(1.0)) == 0.0)
		discard;

	float roughness = speccol_roughness.a;
	float roughnessSquared = max(1e-3, roughness * roughness);
	float a2 = roughnessSquared * roughnessSquared;

	/* Generate Ray */
	float pdf;
	vec3 rand = texelFetch(utilTex, ivec3(halfres_texel % LUT_SIZE, 2), 0).rba;
	vec3 R = generate_ray(V, N, a2, rand, pdf);
#ifdef TWO_HIT
	float pdf2;
	vec3 R2 = generate_ray(V, N, a2, rand * vec3(1.0, -1.0, -1.0), pdf2);
#endif

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
	float hit_dist = -1.0;
#ifdef TWO_HIT
	float hit_dist2 = -1.0;
#endif
	/* Only raytrace if ray is above the surface normal */
	/* Note : this still fails in some cases like with normal map.
	 * We should check against the geometric normal but we don't have it at this stage. */
	if (dot(R, N) > 0.0001) {
		hit_dist = raycast(depthBuffer, viewPosition, R, rand.x);
	}
#ifdef TWO_HIT
	/* TODO do double raytrace at the same time */
	if (dot(R2, N) > 0.0001) {
		hit_dist2 = raycast(depthBuffer, viewPosition, R2, rand.x);
	}
#endif

	/* TODO Do reprojection here */
	vec2 hit_co = project_point(ProjectionMatrix, viewPosition + R * hit_dist).xy * 0.5 + 0.5;
#ifdef TWO_HIT
	vec2 hit_co2 = project_point(ProjectionMatrix, viewPosition + R2 * hit_dist2).xy * 0.5 + 0.5;
#endif

	/* Check if has hit a backface */
	vec3 hit_N = normal_decode(textureLod(normalBuffer, hit_co, 0.0).rg, V);
	hit_dist *= step(0.0, dot(-R, hit_N));
#ifdef TWO_HIT
	hit_N = normal_decode(textureLod(normalBuffer, hit_co2, 0.0).rg, V);
	hit_dist2 *= step(0.0, dot(-R2, hit_N));
#endif

	if (hit_dist > 0.0) {
		hitData = hit_co.xyxy;
	}
	else {
		hitData = vec4(-1.0);
	}
#ifdef TWO_HIT
	if (hit_dist2 > 0.0) {
		hitData.zw = hit_co2;
	}
	else {
		hitData.zw = vec2(-1.0);
	}
#endif

#ifdef TWO_HIT
	pdfData = vec4(pdf, pdf2, 0.0, 0.0);
#else
	pdfData = vec4(pdf);
#endif
}

#else /* STEP_RESOLVE */

uniform sampler2D colorBuffer; /* previous frame */
uniform sampler2D depthBuffer;
uniform sampler2D normalBuffer;
uniform sampler2D specroughBuffer;

uniform sampler2D hitBuffer;
uniform sampler2D pdfBuffer;

uniform int probe_count;

uniform float borderFadeFactor;

uniform mat4 ViewProjectionMatrix;
uniform mat4 PastViewProjectionMatrix;

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

#if 0 /* Finish reprojection with motion vectors */
vec3 get_motion_vector(vec3 pos)
{
}

/* http://bitsquid.blogspot.fr/2017/06/reprojecting-reflections_22.html */
vec3 find_reflection_incident_point(vec3 cam, vec3 hit, vec3 pos, vec3 N)
{
	float d_cam = point_plane_projection_dist(cam, pos, N);
	float d_hit = point_plane_projection_dist(hit, pos, N);

	if (d_hit < d_cam) {
		/* Swap */
		float tmp = d_cam;
		d_cam = d_hit;
		d_hit = tmp;
	}

	vec3 proj_cam = cam - (N * d_cam);
	vec3 proj_hit = hit - (N * d_hit);

	return (proj_hit - proj_cam) * d_cam / (d_cam + d_hit) + proj_cam;
}
#endif

float brightness(vec3 c)
{
	return max(max(c.r, c.g), c.b);
}

vec2 get_reprojected_reflection(vec3 hit, vec3 pos, vec3 N)
{
	/* TODO real motion vectors */
	/* Transform to viewspace */
	// vec4(get_view_space_from_depth(uvcoords, depth), 1.0);
	// vec4(get_view_space_from_depth(uvcoords, depth), 1.0);

	/* Reproject */
	// vec3 hit_reprojected = find_reflection_incident_point(cameraPos, hit, pos, N);

	return project_point(PastViewProjectionMatrix, hit).xy * 0.5 + 0.5;
}

float screen_border_mask(vec2 past_hit_co, vec3 hit)
{
	/* Fade on current and past screen edges */
	vec4 hit_co = ViewProjectionMatrix * vec4(hit, 1.0);
	hit_co.xy = (hit_co.xy / hit_co.w) * 0.5 + 0.5;
	hit_co.zw = past_hit_co;

	const float margin = 0.003;
	float atten = borderFadeFactor + margin; /* Screen percentage */
	hit_co = smoothstep(margin, atten, hit_co) * (1 - smoothstep(1.0 - atten, 1.0 - margin, hit_co));
	vec2 atten_fac = min(hit_co.xy, hit_co.zw);

	float screenfade = atten_fac.x * atten_fac.y;

	return screenfade;
}

float view_facing_mask(vec3 V, vec3 R)
{
	/* Fade on viewing angle (strange deformations happens at R == V) */
	return smoothstep(0.95, 0.80, dot(V, R));
}

vec4 get_ssr_sample(
        vec2 hit_co, vec3 worldPosition, vec3 N, vec3 V, float roughnessSquared,
        float cone_tan, vec2 source_uvs, vec2 texture_size, ivec2 target_texel,
        out float weight)
{
	/* Reconstruct ray */
	float hit_depth = textureLod(depthBuffer, hit_co, 0.0).r;
	vec3 hit_pos = get_world_space_from_depth(hit_co, hit_depth);

	/* Find hit position in previous frame */
	vec2 ref_uvs = get_reprojected_reflection(hit_pos, worldPosition, N);

	/* Estimate a cone footprint to sample a corresponding mipmap level */
	/* compute cone footprint Using UV distance because we are using screen space filtering */
	float cone_footprint = 1.5 * cone_tan * distance(ref_uvs, source_uvs);
	float mip = BRDF_BIAS * clamp(log2(cone_footprint * max(texture_size.x, texture_size.y)), 0.0, MAX_MIP);

	vec3 L = normalize(hit_pos - worldPosition);
#ifdef USE_NORMALIZATION
	/* Evaluate BSDF */
	float bsdf = bsdf_ggx(N, L, V, roughnessSquared);
	float pdf = texelFetch(pdfBuffer, target_texel, 0).r;

	weight = step(0.001, pdf) * bsdf / pdf;
#else
	weight = 1.0;
#endif

	vec3 sample = textureLod(colorBuffer, ref_uvs, mip).rgb ;

	/* Firefly removal */
	sample /= 1.0 + brightness(sample);

	float mask = screen_border_mask(ref_uvs, hit_pos);
	mask *= view_facing_mask(V, N);

	/* Check if there was a hit */
	return vec4(sample, mask) * weight * step(0.0, hit_co.x);
}

#define NUM_NEIGHBORS 9

void main()
{
	ivec2 fullres_texel = ivec2(gl_FragCoord.xy);
#ifdef FULLRES
	ivec2 halfres_texel = fullres_texel;
#else
	ivec2 halfres_texel = ivec2(gl_FragCoord.xy / 2.0);
#endif
	vec2 texture_size = vec2(textureSize(depthBuffer, 0));
	vec2 uvs = gl_FragCoord.xy / texture_size;
	vec3 rand = texelFetch(utilTex, ivec3(fullres_texel % LUT_SIZE, 2), 0).rba;

	float depth = textureLod(depthBuffer, uvs, 0.0).r;

	/* Early out */
	if (depth == 1.0)
		discard;

	/* Using world space */
	vec3 viewPosition = get_view_space_from_depth(uvs, depth); /* Needed for viewCameraVec */
	vec3 worldPosition = transform_point(ViewMatrixInverse, viewPosition);
	vec3 V = cameraVec;
	vec3 N = mat3(ViewMatrixInverse) * normal_decode(texelFetch(normalBuffer, fullres_texel, 0).rg, viewCameraVec);
	vec4 speccol_roughness = texelFetch(specroughBuffer, fullres_texel, 0).rgba;

	/* Early out */
	if (dot(speccol_roughness.rgb, vec3(1.0)) == 0.0)
		discard;

	float roughness = speccol_roughness.a;
	float roughnessSquared = max(1e-3, roughness * roughness);

	vec4 spec_accum = vec4(0.0);

	/* Resolve SSR */
	float cone_cos = cone_cosine(roughnessSquared);
	float cone_tan = sqrt(1 - cone_cos * cone_cos) / cone_cos;
	cone_tan *= mix(saturate(dot(N, V) * 2.0), 1.0, roughness); /* Elongation fit */

	vec2 source_uvs = project_point(PastViewProjectionMatrix, worldPosition).xy * 0.5 + 0.5;

	vec4 ssr_accum = vec4(0.0);
	float weight_acc = 0.0;
	const ivec2 neighbors[9] = ivec2[9](
		ivec2(0, 0),
		ivec2(-1,  1), ivec2(0,  1), ivec2(1,  1),
		ivec2(-1,  0),               ivec2(1,  0),
		ivec2(-1, -1), ivec2(0, -1), ivec2(1, -1)
	);
	ivec2 invert_neighbor;
	invert_neighbor.x = ((fullres_texel.x & 0x1) == 0) ? 1 : -1;
	invert_neighbor.y = ((fullres_texel.y & 0x1) == 0) ? 1 : -1;
	for (int i = 0; i < NUM_NEIGHBORS; i++) {
		ivec2 target_texel = halfres_texel + neighbors[i] * invert_neighbor;

#ifdef TWO_HIT
		vec4 hit_co = texelFetch(hitBuffer, target_texel, 0).rgba;
#else
		vec2 hit_co = texelFetch(hitBuffer, target_texel, 0).rg;
#endif

		float weight;
		ssr_accum += get_ssr_sample(hit_co.xy, worldPosition, N, V,
		                            roughnessSquared, cone_tan, source_uvs,
		                            texture_size, target_texel, weight);
		weight_acc += weight;

#ifdef TWO_HIT
		ssr_accum += get_ssr_sample(hit_co.zw, worldPosition, N, V,
		                            roughnessSquared, cone_tan, source_uvs,
		                            texture_size, target_texel, weight);
		weight_acc += weight;
#endif
	}

	/* Compute SSR contribution */
	if (weight_acc > 0.0) {
		ssr_accum /= weight_acc;
		/* fade between 0.5 and 1.0 roughness */
		ssr_accum.a *= saturate(2.0 - roughness * 2.0);
		accumulate_light(ssr_accum.rgb, ssr_accum.a, spec_accum);
	}

	/* If SSR contribution is not 1.0, blend with cubemaps */
	if (spec_accum.a < 1.0) {
		fallback_cubemap(N, V, worldPosition, roughness, roughnessSquared, spec_accum);
	}

	fragColor = vec4(spec_accum.rgb * speccol_roughness.rgb, 1.0);
	// vec2 _uvs = project_point(PastViewProjectionMatrix, worldPosition).xy * 0.5 + 0.5;
	// fragColor = vec4(textureLod(colorBuffer, _uvs, roughness * MAX_MIP).rgb, 1.0);
}

#endif

```
>
- 留意 TWO_HIT 宏，其实 TWO_HIT 的思路就是计算2次 ray tracing 来提供精度，但是性能也会多一倍出来