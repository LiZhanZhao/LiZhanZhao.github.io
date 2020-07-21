---
layout:     post
title:      "blender eevee Make default shaders works"
subtitle:   ""
date:       2020-07-21 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> 2017/4/20   Eevee: Make default shaders works.


## 编译

- 直接编译


## 效果展示
![](/img/Eevee/DefaultShader/default-Shader.png)



## 渲染


### CPU

```c

DRW_draw_pass(psl->material_pass);

```

> 
- 渲染 material_pass


```c

DRWState state = DRW_STATE_WRITE_COLOR | DRW_STATE_DEPTH_EQUAL;
psl->material_pass = DRW_pass_create("Material Shader Pass", state);

/* NOTE : this shading grp does not contain any geom, it's just here to setup uniforms & textures. */
stl->g_data->material_lit_grp = DRW_shgroup_create(e_data.default_lit, psl->material_pass);

```
> 
-  material_pass 与 default_lit shader代码相关



```c

if (!e_data.default_lit) {
		char *frag_str = NULL;

		DynStr *ds_frag = BLI_dynstr_new();
		BLI_dynstr_append(ds_frag, datatoc_bsdf_common_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_ltc_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_bsdf_direct_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_lit_surface_frag_glsl);
		BLI_dynstr_append(ds_frag, datatoc_default_frag_glsl);
		frag_str = BLI_dynstr_get_cstring(ds_frag);
		BLI_dynstr_free(ds_frag);

		e_data.default_lit = DRW_shader_create(
		        datatoc_lit_surface_vert_glsl, NULL, frag_str,
		        "#define MAX_LIGHT 128\n"
		        "#define MAX_SHADOW_CUBE 42\n"
		        "#define MAX_SHADOW_MAP 64\n"
		        "#define MAX_SHADOW_CASCADE 8\n"
		        "#define MAX_CASCADE_NUM 4\n");

		MEM_freeN(frag_str);
	}


```
> 
-  拼接 default_lit 代码，可以看到使用了 datatoc_default_frag_glsl, 也就是 default_frag.glsl



### Shader

*default_frag.glsl*

```glsl
uniform vec3 diffuse_col;
uniform vec3 specular_col;
uniform int hardness;

void main()
{
	float roughness = 1.0 - float(hardness) / 511.0;
	fragColor = vec4(eevee_surface_lit(worldNormal, diffuse_col, specular_col, roughness), 1.0);
}

```
> 
- 添加 default_frag.glsl 作为默认材质的面片着色器。

*lit_surface_frag.glsl*
```glsl


vec3 eevee_surface_lit(vec3 world_normal, vec3 albedo, vec3 f0, float roughness)
{
	ShadingData sd;
	sd.N = normalize(world_normal);
	sd.V = (ProjectionMatrix[3][3] == 0.0) /* if perspective */
	            ? normalize(cameraPos - worldPosition)
	            : normalize(eye);
	sd.W = worldPosition;
	sd.R = reflect(-sd.V, sd.N);
	sd.spec_dominant_dir = get_specular_dominant_dir(sd.N, sd.R, roughness);

	vec3 radiance = vec3(0.0);

	/* Analitic Lights */
	for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
		LightData ld = lights_data[i];

		sd.l_vector = ld.l_position - worldPosition;
		sd.l_distance = length(sd.l_vector);

		light_common(ld, sd);

		float vis = light_visibility(ld, sd);
		float spec = light_specular(ld, sd, roughness);
		float diff = light_diffuse(ld, sd);
		vec3 fresnel = light_fresnel(ld, sd, f0);

		radiance += vis * (albedo * diff + fresnel * spec) * ld.l_color;
	}

	/* Envmaps */
	vec2 uv = ltc_coords(dot(sd.N, sd.V), sqrt(roughness));
	vec2 brdf_lut = texture(brdfLut, uv).rg;
	vec3 Li = textureLod(probeFiltered, sd.spec_dominant_dir, roughness * lodMax).rgb;
	radiance += Li * brdf_lut.y + f0 * Li * brdf_lut.x;

	radiance += spherical_harmonics(sd.N, shCoefs) * albedo;

	return radiance;
}
```
> 
- 之前都是使用 lit_surface_frag 来进行frag 来进行面片着色器，现在只是封装一下，添加 eevee_surface_lit 函数来计算光照，外部其他文件进行调用。



### Shader 数据

```c
DRWState state = DRW_STATE_WRITE_COLOR | DRW_STATE_DEPTH_EQUAL;
psl->material_pass = DRW_pass_create("Material Shader Pass", state);

/* NOTE : this shading grp does not contain any geom, it's just here to setup uniforms & textures. */
stl->g_data->material_lit_grp = DRW_shgroup_create(e_data.default_lit, psl->material_pass);
DRW_shgroup_uniform_block(stl->g_data->material_lit_grp, "light_block", stl->light_ubo, 0);
DRW_shgroup_uniform_block(stl->g_data->material_lit_grp, "shadow_block", stl->shadow_ubo, 1);
DRW_shgroup_uniform_int(stl->g_data->material_lit_grp, "light_count", &stl->lamps->num_light, 1);
DRW_shgroup_uniform_float(stl->g_data->material_lit_grp, "lodMax", &stl->probes->lodmax, 1);
DRW_shgroup_uniform_vec3(stl->g_data->material_lit_grp, "shCoefs[0]", (float *)stl->probes->shcoefs, 9);
DRW_shgroup_uniform_vec3(stl->g_data->material_lit_grp, "cameraPos", e_data.camera_pos, 1);
DRW_shgroup_uniform_texture(stl->g_data->material_lit_grp, "ltcMat", e_data.ltc_mat, 0);
DRW_shgroup_uniform_texture(stl->g_data->material_lit_grp, "brdfLut", e_data.brdf_lut, 1);
DRW_shgroup_uniform_texture(stl->g_data->material_lit_grp, "probeFiltered", txl->probe_pool, 2);
/* NOTE : Adding Shadow Map textures uniform in EEVEE_cache_finish */
```



```c
/* Get per-material split surface */
struct Batch **mat_geom = DRW_cache_object_surface_material_get(ob);
for (int i = 0; i < MAX2(1, ob->totcol); ++i) {
	Material *ma = give_current_material(ob, i + 1);

	if (ma == NULL)
		ma = &defmaterial;

	DRWShadingGroup *shgrp = DRW_shgroup_create(e_data.default_lit, psl->material_pass);
	DRW_shgroup_uniform_vec3(shgrp, "diffuse_col", &ma->r, 1);
	DRW_shgroup_uniform_vec3(shgrp, "specular_col", &ma->specr, 1);
	DRW_shgroup_uniform_short(shgrp, "hardness", &ma->har, 1);
	DRW_shgroup_call_add(shgrp, mat_geom[i], ob->obmat);
}
```
> 
- 传入 各种参数到 Shader中，diffuse_col, specular_col, hardness 都是在材质面板中取，也就是提供给用户调整得。
