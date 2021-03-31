---
layout:     post
title:      "blender eevee Shadows Add Contact Shadows"
subtitle:   ""
date:       2021-3-31 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/10/7  *   Eevee : Shadows: Add Contact Shadows <br> 

> This add the possibility to add screen space raytraced shadows to fix light leaking cause by shadows maps.
Theses inherit of the same artifacts as other screenspace methods.

> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
实现 光源的 ContactShadows 效果


## 效果
*没有开启Contact Shadows*
![](/img/Eevee/ContactShadows//1.png)
<br>
*开启Contact Shadows*
![](/img/Eevee/ContactShadows//2.png)
<br>



### Shader

*lit_surface_frag.glsl*
```
vec3 eevee_surface_lit(vec3 N, vec3 albedo, vec3 f0, float roughness, float ao, int ssr_id, out vec3 ssr_spec)
{
    ...
    vec3 l_color_vis = ld.l_color * light_visibility(ld, worldPosition, viewPosition, viewNormal, l_vector);
    ...
}

```
>
- light_visibility 函数直接计算影子

<br>

*bsdf_common_lib.glsl*
```
struct ShadowData {
	vec4 near_far_bias_exp;
	vec4 shadow_data_start_end;
	vec4 contact_shadow_data;
};

#define sh_contact_dist            contact_shadow_data.x
#define sh_contact_offset          contact_shadow_data.y
#define sh_contact_spread          contact_shadow_data.z
#define sh_contact_thickness       contact_shadow_data.w

```
>
- 这些参数是由外部传入

<br>

*lamps_lib.glsl*
```
float light_visibility(LightData ld, vec3 W, vec3 viewPosition, vec3 viewNormal, vec4 l_vector)
{
	float vis = 1.0;

	if (ld.l_type == SPOT) {
		float z = dot(ld.l_forward, l_vector.xyz);
		vec3 lL = l_vector.xyz / z;
		float x = dot(ld.l_right, lL) / ld.l_sizex;
		float y = dot(ld.l_up, lL) / ld.l_sizey;

		float ellipse = 1.0 / sqrt(1.0 + x * x + y * y);

		float spotmask = smoothstep(0.0, 1.0, (ellipse - ld.l_spot_size) / ld.l_spot_blend);

		vis *= spotmask;
		vis *= step(0.0, -dot(l_vector.xyz, ld.l_forward));
	}
	else if (ld.l_type == AREA) {
		vis *= step(0.0, -dot(l_vector.xyz, ld.l_forward));
	}

#if !defined(VOLUMETRICS) || defined(VOLUME_SHADOW)
	/* shadowing */
	if (ld.l_shadowid >= 0.0) {
		ShadowData data = shadows_data[int(ld.l_shadowid)];

		if (ld.l_type == SUN) {
			/* TODO : MSM */
			// for (int i = 0; i < MAX_MULTI_SHADOW; ++i) {
				vis *= shadow_cascade(
					data, shadows_cascade_data[int(data.sh_data_start)],
					data.sh_tex_start, W);
			// }
		}
		else {
			/* TODO : MSM */
			// for (int i = 0; i < MAX_MULTI_SHADOW; ++i) {
				vis *= shadow_cubemap(
					data, shadows_cube_data[int(data.sh_data_start)],
					data.sh_tex_start, W);
			// }
		}
#endif
        //----------------------------------------------------------------
#ifndef VOLUMETRICS
		/* Only compute if not already in shadow. */
		if ((vis > 0.001) && (data.sh_contact_dist > 0.0)) {
			vec4 L = (ld.l_type != SUN) ? l_vector : vec4(-ld.l_forward, 1.0);
			float trace_distance = (ld.l_type != SUN) ? min(data.sh_contact_dist, l_vector.w) : data.sh_contact_dist;

			vec3 T, B;
			make_orthonormal_basis(L.xyz / L.w, T, B);

			vec3 rand = texture(utilTex, vec3(gl_FragCoord.xy / LUT_SIZE, 2.0)).xzw;
			rand.yz *= rand.x * data.sh_contact_spread;

			/* We use the full l_vector.xyz so that the spread is minimize
			 * if the shading point is further away from the light source */
			vec3 ray_dir = L.xyz + T * rand.y + B * rand.z;
			ray_dir = transform_direction(ViewMatrix, ray_dir);
			ray_dir = normalize(ray_dir);
			vec3 ray_origin = viewPosition + viewNormal * data.sh_contact_offset;
			vec3 hit_pos = raycast(-1, ray_origin, ray_dir * trace_distance, data.sh_contact_thickness, rand.x, 0.75, 0.01);

			if (hit_pos.z > 0.0) {
				hit_pos = get_view_space_from_depth(hit_pos.xy, hit_pos.z);
				float hit_dist = distance(viewPosition, hit_pos);
				float dist_ratio = hit_dist / trace_distance;
				return mix(0.0, vis, dist_ratio * dist_ratio * dist_ratio);
			}
		}
	}
#endif
    //----------------------------------------------------------------

	return vis;
}
```
>
- 最主要的就是标记好的那段代码
- 主要的思路就是, 通过物体上的 position + 光源方向，来检查碰撞，如果 射线碰到物体的话，就表示 这个position 位于 阴影内


<br><br>

### Shader 参数
*eevee_lights.c*
```
static void eevee_shadow_cube_setup(Object *ob, EEVEE_LampsInfo *linfo, EEVEE_LampEngineData *led)
{
	...

	ubo_data->contact_dist = (la->mode & LA_SHAD_CONTACT) ? la->contact_dist : 0.0f;
	ubo_data->contact_bias = 0.05f * la->contact_bias;
	ubo_data->contact_spread = la->contact_spread;
	ubo_data->contact_thickness = la->contact_thickness;
}
```

