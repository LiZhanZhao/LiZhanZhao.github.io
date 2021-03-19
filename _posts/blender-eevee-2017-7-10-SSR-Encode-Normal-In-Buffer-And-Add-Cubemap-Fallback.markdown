---
layout:     post
title:      "blender eevee SSR Encode Normal in buffer and add cubemap fallback"
subtitle:   ""
date:       2021-2-26 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/17  *   Eevee: SSR Encode Normal in buffer and add cubemap fallback .<br> 

> Normals can point away from the camera so we cannot just put XY in the buffer and reconstruct Z later as we would not know the sign of Z.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51

## 作用
SSR 添加 Encode Normal 和 Cubemap fallback

## 编译
- 重新生成SLN
- git 定位到  2017/7/17  *  Eevee: SSR Encode Normal in buffer and add cubemap fallback  .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 准备
先阅读[这里](http://shaderstore.cn/2021/02/24/blender-eevee-2017-7-17-SSR-Output-SSR-Datas-To-Buffers/) 

## 1. Encode Normal

*bsdf_commmon_lib.glsl*
```
/* ---- Encode / Decode Normal buffer data ---- */
/* From http://aras-p.info/texts/CompactNormalStorage.html
 * Using Method #4: Spheremap Transform */
vec2 normal_encode(vec3 n, vec3 view)
{
    float p = sqrt(n.z * 8.0 + 8.0);
    return n.xy / p + 0.5;
}

vec3 normal_decode(vec2 enc, vec3 view)
{
    vec2 fenc = enc * 4.0 - 2.0;
    float f = dot(fenc, fenc);
    float g = sqrt(1.0 - f / 4.0);
    vec3 n;
    n.xy = fenc*g;
    n.z = 1 - f / 2;
    return n;
}
```
<br>

*default_frag.glsl*
```
uniform vec3 basecol;
uniform float metallic;
uniform float specular;
uniform float roughness;

Closure nodetree_exec(void)
{
	vec3 dielectric = vec3(0.034) * specular * 2.0;
	vec3 diffuse = mix(basecol, vec3(0.0), metallic);
	vec3 f0 = mix(dielectric, basecol, metallic);
	vec3 ssr_spec;
	vec3 radiance = eevee_surface_lit((gl_FrontFacing) ? worldNormal : -worldNormal, diffuse, f0, roughness, 1.0, 0, ssr_spec);

	Closure result = Closure(radiance, 1.0, vec4(ssr_spec, roughness), normal_encode(normalize(viewNormal), viewCameraVec), 0);

#if !defined(USE_ALPHA_BLEND)
	result.opacity = length(viewPosition);
#endif

	return result;
}
```
>
- 最主要就是这里了 normal_encode(normalize(viewNormal)
- Encode Normal 在 [这里](http://aras-p.info/texts/CompactNormalStorage.html)
- Using Method #4: Spheremap Transform


## 2. Cubemap fallback

### STEP_RAYTRACE
*effect_ssr_frag.glsl*
```

#ifdef STEP_RAYTRACE

uniform sampler2D depthBuffer;
uniform sampler2D normalBuffer;
uniform sampler2D specroughBuffer;

layout(location = 0) out vec4 hitData;
layout(location = 1) out vec4 pdfData;

void main()
{
	ivec2 fullres_texel = ivec2(gl_FragCoord.xy) * 2;
	float depth = texelFetch(depthBuffer, fullres_texel, 0).r;

	/* Early discard */
	if (depth == 1.0)
		discard;

	hitData = vec4(0.2);
	pdfData = vec4(0.5);
}

#else /* STEP_RESOLVE */

...

#endif

```
>
- 深度为1的区域不进行SSR

<br>

### STEP_RESOLVE

*effect_ssr_frag.glsl*
```

#ifdef STEP_RAYTRACE

...

#else /* STEP_RESOLVE */

uniform sampler2D depthBuffer;
uniform sampler2D normalBuffer;
uniform sampler2D specroughBuffer;

uniform sampler2D hitBuffer;
uniform sampler2D pdfBuffer;

uniform int probe_count;

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

	vec3 worldPosition = get_world_space_from_depth(uvs, depth);
	vec3 V = cameraVec;
	vec3 N = mat3(ViewMatrixInverse) * normal_decode(texelFetch(normalBuffer, fullres_texel, 0).rg, V);
	vec4 speccol_roughness = texelFetch(specroughBuffer, fullres_texel, 0).rgba;
	float roughness = speccol_roughness.a;
	float roughnessSquared = roughness * roughness;

	vec4 spec_accum = vec4(0.0);

	/* Resolve SSR and compute contribution */

	/* If SSR contribution is not 1.0, blend with cubemaps */
	if (spec_accum.a < 1.0) {
		fallback_cubemap(N, V, worldPosition, roughness, roughnessSquared, spec_accum);
	}

	fragColor = vec4(spec_accum.rgb * speccol_roughness.rgb, 1.0);
}

#endif

```
>
- 深度为1的区域不进行SSR
- fallback_cubemap 这里进行计算 Probe Cubemap Reflection 和 World Probe，但是计算这两个东西的前提是 spec_accum.a < 1.0, 当spec_accum.a < 1.0的时候才会和他们进行混合。
