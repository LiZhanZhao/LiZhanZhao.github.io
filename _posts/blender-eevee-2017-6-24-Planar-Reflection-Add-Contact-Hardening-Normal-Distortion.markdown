---
layout:     post
title:      "blender eevee Planar Reflection Add Contact Hardening Normal Distortion."
subtitle:   ""
date:       2021-2-8 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/24  * Eevee : Planar Reflection: Add contact hardening normal distortion.

> Save radial distance to camera in alpha channel of the planar probe.
Use this distance to modulate distortion intensity when shading the surface.<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/PlanarReflections/3.png)

## 作用 
优化镜面反射效果

## 编译
- 重新生成SLN
- git 定位到  2017/6/24  * Eevee : Planar Reflection: Add contact hardening normal distortion .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## Shader


*default_frag.glsl*
```
uniform vec3 basecol;
uniform float metallic;
uniform float specular;
uniform float roughness;

out vec4 FragColor;

void main()
{
	vec3 dielectric = vec3(0.034) * specular * 2.0;
	vec3 diffuse = mix(basecol, vec3(0.0), metallic);
	vec3 f0 = mix(dielectric, basecol, metallic);
	FragColor = vec4(eevee_surface_lit((gl_FrontFacing) ? worldNormal : -worldNormal, diffuse, f0, roughness, 1.0), length(viewPosition));
}
```
>
- 输出FragColor的Alpha是length(viewPosition)

*gup_shader_material.glsl*
```
...
void node_output_eevee_material(vec4 surface, out vec4 result)
{
	result = vec4(surface.rgb, length(viewPosition));
}
...

```
>
- 输出FragColor的Alpha是length(viewPosition)

*bsdf_common_lib.glsl*
```
float len_squared(vec3 a) { return dot(a, a); }
...
float line_plane_intersect_dist(vec3 lineorigin, vec3 linedirection, vec4 plane)
{
	vec3 plane_co = plane.xyz * (-plane.w / len_squared(plane.xyz));
	vec3 h = lineorigin - plane_co;
	return -dot(plane.xyz, h) / dot(plane.xyz, linedirection);
}
```

*lit_surface_frag.glsl*
```
vec3 eevee_surface_lit(vec3 world_normal, vec3 albedo, vec3 f0, float roughness, float ao)
{
	...
	/* Planar Reflections */
	for (int i = 0; i < MAX_PLANAR && i < planar_count && spec_accum.a < 0.999; ++i) {
		PlanarData pd = planars_data[i];

		float influence = planar_attenuation(sd.W, sd.N, pd);

		if (influence > 0.0) {
			float influ_spec = min(influence, (1.0 - spec_accum.a));

			/* Sample reflection depth. */
			vec4 refco = pd.reflectionmat * vec4(sd.W, 1.0);
			refco.xy /= refco.w;
			float ref_depth = textureLod(probePlanars, vec3(refco.xy, i), 0.0).a;

			/* Find view vector / reflection plane intersection. (dist_to_plane is negative) */
			float dist_to_plane = line_plane_intersect_dist(cameraPos, sd.V, pd.pl_plane_eq);
			vec3 point_on_plane = cameraPos + sd.V * dist_to_plane;

			/* How far the pixel is from the plane. */
			ref_depth = ref_depth + dist_to_plane;

			/* Compute distorded reflection vector based on the distance to the reflected object.
			 * In other words find intersection between reflection vector and the sphere center
			 * around point_on_plane. */
			vec3 proj_ref = reflect(R * ref_depth, pd.pl_normal);

			/* Final point in world space. */
			vec3 ref_pos = point_on_plane + proj_ref;

			/* Reproject to find texture coords. */
			refco = pd.reflectionmat * vec4(ref_pos, 1.0);
			refco.xy /= refco.w;

			vec3 sample = textureLod(probePlanars, vec3(refco.xy, i), 0.0).rgb;

			spec_accum.rgb += sample * influ_spec;
			spec_accum.a += influ_spec;
		}
	}
	...

	vec3 indirect_radiance =
	        spec_accum.rgb * F_ibl(f0, brdf_lut) * float(specToggle) * specular_occlusion(dot(sd.N, sd.V), final_ao, roughness) +
	        diff_accum.rgb * albedo * gtao_multibounce(final_ao, albedo);
}
```
>
- float ref_depth = textureLod(probePlanars, vec3(refco.xy, i), 0.0).a; 由于以上输出alpha为 length(viewPosition), 所有这里取得alpha就是length(viewPosition)


