---
layout:     post
title:      "blender eevee Add parallax correction to probe reflections"
subtitle:   ""
date:       2021-1-15 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/10  * Eevee: Add parallax correction to probe reflections. <br>  

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/ParallexCorrectionToProbeReflections/Sphere_par.png)
![](/img/Eevee/ParallexCorrectionToProbeReflections/Sphere_res.png)
![](/img/Eevee/ParallexCorrectionToProbeReflections/Box_par.png)
![](/img/Eevee/ParallexCorrectionToProbeReflections/Box_res.png)

## 作用 
Probe Reflections 添加 视差矫正 (parallax correction)

## 编译
- 重新生成SLN
- git 定位到  2017/6/10  * Eevee: Add parallax correction to probe reflections  
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## Shader

```

struct ProbeData {
	vec4 position_type;
	vec4 shcoefs[7];
	vec4 attenuation_fac_type;
	mat4 influencemat;
	mat4 parallaxmat;
};

#define PROBE_PARALLAX_BOX    1.0
#define PROBE_ATTENUATION_BOX 1.0

#define p_position      position_type.xyz
#define p_parallax_type position_type.w
#define p_atten_fac     attenuation_fac_type.x
#define p_atten_type    attenuation_fac_type.y

float line_unit_sphere_intersect_dist(vec3 lineorigin, vec3 linedirection)
{
	float a = dot(linedirection, linedirection);
	float b = dot(linedirection, lineorigin);
	float c = dot(lineorigin, lineorigin) - 1;

	float dist = 1e15;
	float determinant = b * b - a * c;
	if (determinant >= 0)
		dist = (sqrt(determinant) - b) / a;

	return dist;
}

float line_unit_box_intersect_dist(vec3 lineorigin, vec3 linedirection)
{
	/* https://seblagarde.wordpress.com/2012/09/29/image-based-lighting-approaches-and-parallax-corrected-cubemap/ */
	vec3 firstplane  = (vec3( 1.0) - lineorigin) / linedirection;
	vec3 secondplane = (vec3(-1.0) - lineorigin) / linedirection;
	vec3 furthestplane = max(firstplane, secondplane);

	return min_v3(furthestplane);
}


vec3 probe_parallax_correction(vec3 W, vec3 spec_dir, ProbeData pd, inout float roughness)
{
	vec3 localpos = (pd.parallaxmat * vec4(W, 1.0)).xyz;
	vec3 localray = (pd.parallaxmat * vec4(spec_dir, 0.0)).xyz;

	float dist;
	if (pd.p_parallax_type == PROBE_PARALLAX_BOX) {
		dist = line_unit_box_intersect_dist(localpos, localray);
	}
	else {
		dist = line_unit_sphere_intersect_dist(localpos, localray);
	}

	/* Use Distance in WS directly to recover intersection */
	vec3 intersection = W + spec_dir * dist - pd.p_position;

	/* From Frostbite PBR Course
	 * Distance based roughness
	 * http://www.frostbite.com/wp-content/uploads/2014/11/course_notes_moving_frostbite_to_pbr.pdf */
	float original_roughness = roughness;
	float linear_roughness = sqrt(roughness);
	float distance_roughness = saturate(dist * linear_roughness / length(intersection));
	linear_roughness = mix(distance_roughness, linear_roughness, linear_roughness);
	roughness = linear_roughness * linear_roughness;

	float fac = saturate(original_roughness * 2.0 - 1.0);
	return mix(intersection, spec_dir, fac * fac);
}

float probe_attenuation(vec3 W, ProbeData pd)
{
	vec3 localpos = (pd.influencemat * vec4(W, 1.0)).xyz;

	float fac;
	if (pd.p_atten_type == PROBE_ATTENUATION_BOX) {
		vec3 axes_fac = saturate(pd.p_atten_fac - pd.p_atten_fac * abs(localpos));
		fac = min_v3(axes_fac);
	}
	else {
		fac = saturate(pd.p_atten_fac - pd.p_atten_fac * length(localpos));
	}

	return fac;
}



vec3 eevee_surface_lit(vec3 world_normal, vec3 albedo, vec3 f0, float roughness, float ao)
{
	roughness = clamp(roughness, 1e-8, 0.9999);
	float roughnessSquared = roughness * roughness;

	ShadingData sd;
	sd.N = normalize(world_normal);
	sd.V = (ProjectionMatrix[3][3] == 0.0) /* if perspective */
	            ? normalize(cameraPos - worldPosition)
	            : cameraForward;
	sd.W = worldPosition;

	vec3 radiance = vec3(0.0);
	/* Analitic Lights */
	for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
		LightData ld = lights_data[i];
		vec3 diff, spec;
		float vis;

		sd.l_vector = ld.l_position - worldPosition;

		light_visibility(ld, sd, vis);
		light_shade(ld, sd, albedo, roughnessSquared, f0, diff, spec);

		radiance += vis * (diff + spec) * ld.l_color;
	}


	/* Envmaps */
	vec3 R = reflect(-sd.V, sd.N);
	vec3 spec_dir = get_specular_dominant_dir(sd.N, R, roughnessSquared);
	vec2 uv = lut_coords(dot(sd.N, sd.V), roughness);
	vec2 brdf_lut = texture(utilTex, vec3(uv, 1.0)).rg;

	vec4 spec_accum = vec4(0.0);
	vec4 diff_accum = vec4(0.0);

	/* Specular probes */
	/* Start at 1 because 0 is world probe */
	for (int i = 1; i < MAX_PROBE && i < probe_count; ++i) {
		ProbeData pd = probes_data[i];

		vec3 sample_vec = probe_parallax_correction(sd.W, spec_dir, pd, roughness);
		vec4 sample = textureLod_octahedron(probeCubes, vec4(sample_vec, i), roughness * lodMax).rgba;

		float dist_attenuation = probe_attenuation(sd.W, pd);
		float influ_spec = min(dist_attenuation, (1.0 - spec_accum.a));

		spec_accum.rgb += sample.rgb * influ_spec;
		spec_accum.a += influ_spec;
	}

	/* World probe */
	if (spec_accum.a < 1.0 || diff_accum.a < 1.0) {
		ProbeData pd = probes_data[0];

		vec3 spec = textureLod_octahedron(probeCubes, vec4(spec_dir, 0), roughness * lodMax).rgb;
		vec3 diff = spherical_harmonics(sd.N, pd.shcoefs);

		diff_accum.rgb += diff * (1.0 - diff_accum.a);
		spec_accum.rgb += spec * (1.0 - spec_accum.a);
	}

	vec3 indirect_radiance =
	        spec_accum.rgb * F_ibl(f0, brdf_lut) +
	        diff_accum.rgb * albedo;

	return radiance + indirect_radiance * ao;
}

```
>
- 最主要的函数就是 probe_parallax_correction，本中最主要就是记录 probe_parallax_correction 是怎么做的


## Shader 参数

*eevee_private.h*
```
typedef struct EEVEE_Probe {
	float position[3], parallax_type;
	float shcoefs[9][3], pad2;
	float attenuation_fac;
	float attenuation_type;
	float pad3[2];
	float attenuationmat[4][4];
	float parallaxmat[4][4];
} EEVEE_Probe;

```

*eevee_probe.c*
```
static void EEVEE_probes_updates(EEVEE_SceneLayerData *sldata)
{
	EEVEE_ProbesInfo *pinfo = sldata->probes;
	Object *ob;

	for (int i = 1; (ob = pinfo->probes_ref[i]) && (i < MAX_PROBE); i++) {
		Probe *probe = (Probe *)ob->data;
		EEVEE_Probe *eprobe = &pinfo->probe_data[i];


		/* Attenuation */
		eprobe->attenuation_type = probe->attenuation_type;
		eprobe->attenuation_fac = 1.0f / max_ff(1e-8f, probe->falloff);

		unit_m4(eprobe->attenuationmat);
		if (probe->attenuation_type == PROBE_BOX) {
			BoundBox bb;
			float bb_center[3], bb_size[3];

			BKE_boundbox_init_from_minmax(&bb, probe->mininf, probe->maxinf);
			BKE_boundbox_calc_center_aabb(&bb, bb_center);
			BKE_boundbox_calc_size_aabb(&bb, bb_size);

			eprobe->attenuationmat[0][0] = bb_size[0];
			eprobe->attenuationmat[1][1] = bb_size[1];
			eprobe->attenuationmat[2][2] = bb_size[2];
			copy_v3_v3(eprobe->attenuationmat[3], bb_center);
			mul_m4_m4m4(eprobe->attenuationmat, ob->obmat, eprobe->attenuationmat);
		}
		else {
			scale_m4_fl(eprobe->attenuationmat, probe->distinf);
			mul_m4_m4m4(eprobe->attenuationmat, ob->obmat, eprobe->attenuationmat);
		}
		invert_m4(eprobe->attenuationmat);

		/* Parallax */
		BoundBox parbb;
		float dist;
		if ((probe->flag & PRB_CUSTOM_PARALLAX) != 0) {
			eprobe->parallax_type = probe->parallax_type;
			BKE_boundbox_init_from_minmax(&parbb, probe->minpar, probe->maxpar);
			dist = probe->distpar;
		}
		else {
			eprobe->parallax_type = probe->attenuation_type;
			BKE_boundbox_init_from_minmax(&parbb, probe->mininf, probe->maxinf);
			dist = probe->distinf;
		}

		unit_m4(eprobe->parallaxmat);
		if (eprobe->parallax_type == PROBE_BOX) {
			float bb_center[3], bb_size[3];

			BKE_boundbox_calc_center_aabb(&parbb, bb_center);
			BKE_boundbox_calc_size_aabb(&parbb, bb_size);

			eprobe->parallaxmat[0][0] = bb_size[0];
			eprobe->parallaxmat[1][1] = bb_size[1];
			eprobe->parallaxmat[2][2] = bb_size[2];
			copy_v3_v3(eprobe->parallaxmat[3], bb_center);
			mul_m4_m4m4(eprobe->parallaxmat, ob->obmat, eprobe->parallaxmat);
		}
		else {
			scale_m4_fl(eprobe->parallaxmat, dist);
			mul_m4_m4m4(eprobe->parallaxmat, ob->obmat, eprobe->parallaxmat);
		}
		invert_m4(eprobe->parallaxmat);
	}
}

```
>
- 参数都是在这个函数进行计算