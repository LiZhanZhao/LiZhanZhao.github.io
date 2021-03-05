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
- 这里最主要就是下面的函数, 主要就是重新计算hit_pos点的屏幕坐标，然后再利用这个坐标进行采样colorBuffer
```
float ray_length = texelFetch(hitBuffer, halfres_texel, 0).r;
if (ray_length != -1.0) {
	vec3 hit_pos = worldPosition + R * ray_length;
	vec2 ref_uvs = get_reprojected_reflection(hit_pos, worldPosition, N);
	spec_accum.a = screen_border_mask(ref_uvs, hit_pos);
	spec_accum.xyz = textureLod(colorBuffer, ref_uvs, 0.0).rgb * spec_accum.a;
}
```
- PastViewProjectionMatrix 是上一帧的VP矩阵


## PastViewProjectionMatrix

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
