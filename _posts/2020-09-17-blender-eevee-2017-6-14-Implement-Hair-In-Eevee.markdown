---
layout:     post
title:      "blender eevee Implement hair in eevee"
subtitle:   ""
date:       2021-1-25 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/14  * Implement hair in eevee  <br>  
New implementation of hair for Eevee. <br>  
Note: A hard coded "transmission" property is being used. This should  <br>  
eventually be exposed to the UI, possibly in the form of SSS  <br>  
properties. <br>  

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/Hair/1.png)


## 作用 
添加Hair的渲染

## 编译
- 重新生成SLN
- git 定位到  2017/6/14  * Implement hair in eevee  
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## Hair 的验证
![](/img/Eevee/Hair/2.png)


## 渲染

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	DRW_draw_pass(psl->default_hair_pass);
	...
}
```

>
- 个人理解，现在渲染管线多了 psl->default_hair_pass，但是这个 psl->default_hair_pass 只是针对 ParticleSystem 得Hair 而来得，如果没有ParticleSystem 得 Hair 得话，是不会执行得

*eevee_materials.c*
```
void EEVEE_materials_init(void)
{
	...
	e_data.default_lit_hair = DRW_shader_create(
		        datatoc_lit_surface_vert_glsl, NULL, frag_str,
		        SHADER_DEFINES
		        "#define MESH_SHADER\n"
		        "#define HAIR_SHADER\n");
	...
}

struct GPUMaterial *EEVEE_material_hair_get(struct Scene *scene, Material *ma)
{
	return GPU_material_from_nodetree(
	    scene, ma->nodetree, &ma->gpumaterial, &DRW_engine_viewport_eevee_type,
	    VAR_MAT_HAIR,
	    datatoc_lit_surface_vert_glsl, NULL, e_data.frag_shader_lib,
	    SHADER_DEFINES "#define MESH_SHADER\n" "#define HAIR_SHADER\n");
}


void EEVEE_materials_cache_init(EEVEE_Data *vedata)
{
	...
	psl->default_hair_pass = DRW_pass_create("Default Hair Lit Pass", state);
	shgrp = DRW_shgroup_create(e_data.default_lit_hair, psl->default_hair_pass);
	add_standard_uniforms(shgrp, sldata);
	...
}
```

>
- default_lit_hair 和 default_lit 相差的就是 #define HAIR_SHADER，也就是是否开启宏

## Shader

*lit_surface_frag.glsl*

```

#ifdef HAIR_SHADER
vec3 light_diffuse(LightData ld, ShadingData sd, vec3 albedo)
{
       if (ld.l_type == SUN) {
               return direct_diffuse_sun(ld, sd) * albedo;
       }
       else if (ld.l_type == AREA) {
               return direct_diffuse_rectangle(ld, sd) * albedo;
       }
       else {
               return direct_diffuse_sphere(ld, sd) * albedo;
       }
}

vec3 light_specular(LightData ld, ShadingData sd, float roughness, vec3 f0)
{
       if (ld.l_type == SUN) {
               return direct_ggx_sun(ld, sd, roughness, f0);
       }
       else if (ld.l_type == AREA) {
               return direct_ggx_rectangle(ld, sd, roughness, f0);
       }
       else {
               return direct_ggx_sphere(ld, sd, roughness, f0);
       }
}

void light_shade(
        LightData ld, ShadingData sd, vec3 albedo, float roughness, vec3 f0,
        out vec3 diffuse, out vec3 specular)
{
       const float transmission = 0.3; /* Uniform internal scattering factor */
       ShadingData sd_new = sd;

       vec3 lamp_vec;

      if (ld.l_type == SUN || ld.l_type == AREA) {
               lamp_vec = ld.l_forward;
       }
       else {
               lamp_vec = -sd.l_vector;
       }

       vec3 norm_view = cross(sd.V, sd.N);
       norm_view = normalize(cross(norm_view, sd.N)); /* Normal facing view */

       vec3 norm_lamp = cross(lamp_vec, sd.N);
       norm_lamp = normalize(cross(sd.N, norm_lamp)); /* Normal facing lamp */

       /* Rotate view vector onto the cross(tangent, light) plane */
       vec3 view_vec = normalize(norm_lamp * dot(norm_view, sd.V) + sd.N * dot(sd.N, sd.V));

       float occlusion = (dot(norm_view, norm_lamp) * 0.5 + 0.5);
       float occltrans = transmission + (occlusion * (1.0 - transmission)); /* Includes transmission component */

       sd_new.N = -norm_lamp;

       diffuse = light_diffuse(ld, sd_new, albedo) * occltrans;

       sd_new.V = view_vec;

       specular = light_specular(ld, sd_new, roughness, f0) * occlusion;
}
#else
void light_shade(
        LightData ld, ShadingData sd, vec3 albedo, float roughness, vec3 f0,
        out vec3 diffuse, out vec3 specular)
{
#ifdef USE_LTC
	if (ld.l_type == SUN) {
		/* TODO disk area light */
		diffuse = direct_diffuse_sun(ld, sd) * albedo;
		specular = direct_ggx_sun(ld, sd, roughness, f0);
	}
	else if (ld.l_type == AREA) {
		diffuse =  direct_diffuse_rectangle(ld, sd) * albedo;
		specular =  direct_ggx_rectangle(ld, sd, roughness, f0);
	}
	else {
		diffuse =  direct_diffuse_sphere(ld, sd) * albedo;
		specular =  direct_ggx_sphere(ld, sd, roughness, f0);
	}
#else
	if (ld.l_type == SUN) {
		diffuse = direct_diffuse_sun(ld, sd) * albedo;
		specular = direct_ggx_sun(ld, sd, roughness, f0);
	}
	else {
		diffuse = direct_diffuse_point(ld, sd) * albedo;
		specular = direct_ggx_point(sd, roughness, f0);
	}
#endif
}
#endif



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

#ifdef HAIR_SHADER
       /* View facing normal */
       vec3 norm_view = cross(sd.V, sd.N);
       norm_view = normalize(cross(norm_view, sd.N)); /* Normal facing view */
#endif


	/* Analitic Lights */
	for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
		LightData ld = lights_data[i];
		vec3 diff, spec;
		float vis = 1.0;

		sd.l_vector = ld.l_position - worldPosition;

#ifndef HAIR_SHADER
		light_visibility(ld, sd, vis);
#endif
		light_shade(ld, sd, albedo, roughnessSquared, f0, diff, spec);

		radiance += vis * (diff + spec) * ld.l_color;
	}

#ifdef HAIR_SHADER
	sd.N = -norm_view;
#endif

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

		float dist_attenuation = probe_attenuation(sd.W, pd);

		if (dist_attenuation > 0.0) {
			float roughness_copy = roughness;

			vec3 sample_vec = probe_parallax_correction(sd.W, spec_dir, pd, roughness_copy);
			vec4 sample = textureLod_octahedron(probeCubes, vec4(sample_vec, i), roughness_copy * lodMax).rgba;

			float influ_spec = min(dist_attenuation, (1.0 - spec_accum.a));

			spec_accum.rgb += sample.rgb * influ_spec;
			spec_accum.a += influ_spec;
		}
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
- Shader中最主要的 HAIR_SHADER 的宏，如果是Hair的话，就会进行HAIR_SHADER的宏里面