---
layout:     post
title:      "blender eevee Add Screen Space Refraction"
subtitle:   ""
date:       2021-3-18 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/8/9  *   Eevee: Add Screen Space Refraction .<br> 

> For the moment the only way to enable this is to:
- enable Screen Space REFLECTIONS.
- enable Screen Space Refraction in the SSR parameters.
- enable Screen Space Refraction in the material tab.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
*效果*
![](/img/Eevee/SSR/Refraction/1.png)
<br>

*后处理开关*
![](/img/Eevee/SSR/Refraction/2.png)
<br>

*材质开关*
![](/img/Eevee/SSR/Refraction/3.png)
<br>


## 作用
后处理折射效果实现, 目前只有透明物体才会有折射效果

<br>

## 渲染管线

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Prepare Refraction */
	EEVEE_effects_do_refraction(sldata, vedata);

	/* Restore main FB */
	DRW_framebuffer_bind(fbl->main);

	/* Transparent */
	DRW_pass_sort_shgroup_z(psl->transparent_pass);
	DRW_stats_group_start("Transparent");
	DRW_draw_pass(psl->transparent_pass);
	DRW_stats_group_end();
	...
}
```
>
- 这里先执行 EEVEE_effects_do_refraction, 再渲染 透明物体, EEVEE_effects_do_refraction 函数准备对应的渲染数据

<br><br>

#### a. EEVEE_effects_do_refraction

*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	if (BKE_collection_engine_property_value_get_bool(props, "ssr_enable")) {
		effects->enabled_effects |= EFFECT_SSR;

		if (BKE_collection_engine_property_value_get_bool(props, "ssr_refraction")) {
			effects->enabled_effects |= EFFECT_REFRACT;

			DRWFboTexture tex = {&txl->refract_color, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER | DRW_TEX_MIPMAP};

			DRW_framebuffer_init(&fbl->refract_fb, &draw_engine_eevee_type, (int)viewport_size[0], (int)viewport_size[1], &tex, 1);
		}
		...
	}
	...
}

void EEVEE_effects_do_refraction(EEVEE_SceneLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_REFRACT) != 0) {
		DRW_framebuffer_texture_attach(fbl->refract_fb, txl->refract_color, 0, 0);
		DRW_framebuffer_blit(fbl->main, fbl->refract_fb, false);
		EEVEE_downsample_buffer(vedata, fbl->downsample_fb, txl->refract_color, 9);
	}
}
```
> 
- refract_fb framebuffer 绑定了 refract_color  RT
- EEVEE_effects_do_refraction 的过程就是，先把 fbl->main blit 到 fbl->refract_fb 中，那么就是 refract_color RT 保存了 fbl->main 画面
- EEVEE_downsample_buffer 函数利用Shader effect_downsample_frag.glsl 进行 downsample, 构造 refract_color RT 的 mipmap

#### b. effect_downsample_frag.glsl
```
/**
 * Simple downsample shader. Takes the average of the 4 texels of lower mip.
 **/

uniform sampler2D source;

out vec4 FragColor;

void main()
{
#if 0
	/* Reconstructing Target uvs like this avoid missing pixels if NPO2 */
	vec2 uvs = gl_FragCoord.xy * 2.0 / vec2(textureSize(source, 0));

	FragColor = textureLod(source, uvs, 0.0);
#else
	vec2 texel_size = 1.0 / vec2(textureSize(source, 0));
	vec2 uvs = gl_FragCoord.xy * 2.0 * texel_size;
	vec4 ofs = texel_size.xyxy * vec4(0.75, 0.75, -0.75, -0.75);

	FragColor  = textureLod(source, uvs + ofs.xy, 0.0);
	FragColor += textureLod(source, uvs + ofs.xw, 0.0);
	FragColor += textureLod(source, uvs + ofs.zy, 0.0);
	FragColor += textureLod(source, uvs + ofs.zw, 0.0);
	FragColor *= 0.25;
#endif
}
```



## 渲染透明物体
*eevee_materials.c*
```
static void material_transparent(
        Material *ma, EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata,
        bool do_cull, bool use_flat_nor, struct GPUMaterial **gpumat, struct DRWShadingGroup **shgrp, struct DRWShadingGroup **shgrp_depth)
{
	...

	const bool use_refract = ((ma->blend_flag & MA_BL_REFRACTION) != 0) && ((stl->effects->enabled_effects & EFFECT_REFRACT) != 0);

	...

	if (ma->use_nodes && ma->nodetree) {
		/* Shading */
		*gpumat = EEVEE_material_mesh_get(scene, ma,
		        stl->effects->use_ao, stl->effects->use_bent_normals,
		        true, (ma->blend_method == MA_BM_MULTIPLY), use_refract);

		*shgrp = DRW_shgroup_material_create(*gpumat, psl->transparent_pass);
		if (*shgrp) {
			static int ssr_id = -1; /* TODO transparent SSR */
			static float refract_thickness = 0.0f; /* TODO Param */
			add_standard_uniforms(*shgrp, sldata, vedata, &ssr_id,
			        (use_refract) ? &refract_thickness : NULL);
		}
		else {
			/* Shader failed : pink color */
			static float col[3] = {1.0f, 0.0f, 1.0f};
			static float half = 0.5f;

			color_p = col;
			metal_p = spec_p = rough_p = &half;
		}
	}

	/* Fallback to default shader */
	...
}

```
>
- 如果物体的材质打开了 use_refract 的开关的话，Shader 就进行设置 宏 USE_REFRACTION
- add_standard_uniforms 函数也会传入 一些 refract 功能要用到的参数
- 目前材质使用了refraction，对应得 Shader 中的 eevee_surface_refraction

<br>

#### a. add_standard_uniforms
```
static void add_standard_uniforms(
        DRWShadingGroup *shgrp, EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata,
        int *ssr_id, float *refract_thickness)
{
	if (ssr_id == NULL) {
		static int no_ssr = -1.0f;
		ssr_id = &no_ssr;
	}
	DRW_shgroup_uniform_block(shgrp, "probe_block", sldata->probe_ubo);
	DRW_shgroup_uniform_block(shgrp, "grid_block", sldata->grid_ubo);
	DRW_shgroup_uniform_block(shgrp, "planar_block", sldata->planar_ubo);
	DRW_shgroup_uniform_block(shgrp, "light_block", sldata->light_ubo);
	DRW_shgroup_uniform_block(shgrp, "shadow_block", sldata->shadow_ubo);
	DRW_shgroup_uniform_int(shgrp, "light_count", &sldata->lamps->num_light, 1);
	DRW_shgroup_uniform_int(shgrp, "probe_count", &sldata->probes->num_render_cube, 1);
	DRW_shgroup_uniform_int(shgrp, "grid_count", &sldata->probes->num_render_grid, 1);
	DRW_shgroup_uniform_int(shgrp, "planar_count", &sldata->probes->num_planar, 1);
	DRW_shgroup_uniform_bool(shgrp, "specToggle", &sldata->probes->specular_toggle, 1);
	DRW_shgroup_uniform_bool(shgrp, "ssrToggle", &sldata->probes->ssr_toggle, 1);
	DRW_shgroup_uniform_float(shgrp, "lodCubeMax", &sldata->probes->lod_cube_max, 1);
	DRW_shgroup_uniform_float(shgrp, "lodPlanarMax", &sldata->probes->lod_planar_max, 1);
	DRW_shgroup_uniform_texture(shgrp, "utilTex", e_data.util_tex);
	DRW_shgroup_uniform_buffer(shgrp, "probeCubes", &sldata->probe_pool);
	DRW_shgroup_uniform_buffer(shgrp, "probePlanars", &vedata->txl->planar_pool);
	DRW_shgroup_uniform_buffer(shgrp, "irradianceGrid", &sldata->irradiance_pool);
	DRW_shgroup_uniform_buffer(shgrp, "shadowCubes", &sldata->shadow_depth_cube_pool);
	DRW_shgroup_uniform_buffer(shgrp, "shadowCascades", &sldata->shadow_depth_cascade_pool);
	DRW_shgroup_uniform_int(shgrp, "outputSsrId", ssr_id, 1);
	if (vedata->stl->effects->use_ao || refract_thickness) {
		DRW_shgroup_uniform_vec4(shgrp, "viewvecs[0]", (float *)vedata->stl->g_data->viewvecs, 2);
		DRW_shgroup_uniform_buffer(shgrp, "maxzBuffer", &vedata->txl->maxzbuffer);
	}
	if (refract_thickness) {
		DRW_shgroup_uniform_buffer(shgrp, "colorBuffer", &vedata->txl->refract_color);
		DRW_shgroup_uniform_float(shgrp, "refractionThickness", refract_thickness, 1);
		DRW_shgroup_uniform_vec4(shgrp, "ssrParameters", &vedata->stl->effects->ssr_quality, 1);
		DRW_shgroup_uniform_float(shgrp, "borderFadeFactor", &vedata->stl->effects->ssr_border_fac, 1);
		DRW_shgroup_uniform_float(shgrp, "maxRoughness", &vedata->stl->effects->ssr_max_roughness, 1);
		DRW_shgroup_uniform_vec2(shgrp, "mipRatio[0]", (float *)vedata->stl->g_data->mip_ratio, 10);
		DRW_shgroup_uniform_int(shgrp, "rayCount", &vedata->stl->effects->ssr_ray_count, 1);
	}
	if (vedata->stl->effects->use_ao) {
		DRW_shgroup_uniform_vec3(shgrp, "aoParameters", &vedata->stl->effects->ao_dist, 1);
	}
}

```

<br>

#### b. Shader  eevee_surface_refraction

*bsdf_common_lib.glsl*
```

/* --- Refraction utils --- */

float ior_from_f0(float f0)
{
	float f = sqrt(f0);
	return (-f - 1.0) / (f - 1.0);
}

float f0_from_ior(float eta)
{
	float A = (eta - 1.0) / (eta + 1.0);
	return A * A;
}

vec3 get_specular_refraction_dominant_dir(vec3 N, vec3 V, float roughness, float ior)
{
	/* TODO: This a bad approximation. Better approximation should fit
	 * the refracted vector and roughness into the best prefiltered reflection
	 * lobe. */
	/* Correct the IOR for ior < 1.0 to not see the abrupt delimitation or the TIR */
	ior = (ior < 1.0) ? mix(ior, 1.0, roughness) : ior;
	float eta = 1.0 / ior;

	float NV = dot(N, -V);

	/* Custom Refraction. */
	float k = 1.0 - eta * eta * (1.0 - NV * NV);
	k = max(0.0, k); /* Only this changes. */
	vec3 R = eta * -V - (eta * NV + sqrt(k)) * N;

	return R;
}

float get_btdf_lut(sampler2DArray btdf_lut_tex, float NV, float roughness, float ior)
{
	const vec3 lut_scale_bias_texel_size = vec3((LUT_SIZE - 1.0), 0.5, 1.5) / LUT_SIZE;

	vec3 coords;
	/* Try to compensate for the low resolution and interpolation error. */
	coords.x = (ior > 1.0)
	           ? (0.9 + lut_scale_bias_texel_size.z) + (0.1 - lut_scale_bias_texel_size.z) * f0_from_ior(ior)
	           : (0.9 + lut_scale_bias_texel_size.z) * ior * ior;
	coords.y = 1.0 - NV;
	coords.xy *= lut_scale_bias_texel_size.x;
	coords.xy += lut_scale_bias_texel_size.y;

	const float lut_lvl_ofs = 4.0; /* First texture lvl of roughness. */
	const float lut_lvl_scale = 16.0; /* How many lvl of roughness in the lut. */

	float mip = roughness * lut_lvl_scale;
	float mip_floor = floor(mip);

	coords.z = lut_lvl_ofs + mip_floor + 1.0;
	float btdf_high = textureLod(btdf_lut_tex, coords, 0.0).r;

	coords.z -= 1.0;
	float btdf_low = textureLod(btdf_lut_tex, coords, 0.0).r;

	float btdf = (ior == 1.0) ? 1.0 : mix(btdf_low, btdf_high, mip - coords.z);

	return btdf;
}
```



<br>

*ssr_lib.glsl*

```
/* ------------ Refraction ------------ */

#define BTDF_BIAS 0.85

vec4 screen_space_refraction(vec3 viewPosition, vec3 N, vec3 V, float ior, float roughnessSquared, vec3 rand, float ofs)
{
	float a2 = roughnessSquared * roughnessSquared;
	float jitter = fract(rand.x + ofs);

	/* Importance sampling bias */
	rand.x = mix(rand.x, 0.0, BTDF_BIAS);

	vec3 T, B;
	float NH;
	make_orthonormal_basis(N, T, B);
	vec3 H = sample_ggx(rand, a2, N, T, B, NH); /* Microfacet normal */
	float pdf = pdf_ggx_reflect(NH, a2);

	/* If ray is bad (i.e. going below the plane) regenerate. */
	if (F_eta(ior, dot(H, V)) < 1.0) {
		H = sample_ggx(rand * vec3(1.0, -1.0, -1.0), a2, N, T, B, NH); /* Microfacet normal */
		pdf = pdf_ggx_reflect(NH, a2);
	}

	vec3 vV = viewCameraVec;
	float eta = 1.0/ior;
	if (dot(H, V) < 0.0) {
		H = -H;
		eta = ior;
	}

	vec3 R = refract(-V, H, 1.0 / ior);

	R = transform_direction(ViewMatrix, R);

	vec3 hit_pos = raycast(-1, viewPosition, R, ssrThickness, jitter, roughnessSquared);

	if ((hit_pos.z < 0.0) && (F_eta(ior, dot(H, V)) < 1.0)) {
		float hit_dist = distance(hit_pos, viewPosition);

		float cone_cos = cone_cosine(roughnessSquared);
		float cone_tan = sqrt(1 - cone_cos * cone_cos) / cone_cos;

		/* Empirical fit for refraction. */
		/* TODO find a better fit or precompute inside the LUT. */
		cone_tan *= 0.5 * fast_sqrt(f0_from_ior((ior < 1.0) ? 1.0 / ior : ior));

		float cone_footprint = hit_dist * cone_tan;

		/* find the offset in screen space by multiplying a point
		 * in camera space at the depth of the point by the projection matrix. */
		float homcoord = ProjectionMatrix[2][3] * hit_pos.z + ProjectionMatrix[3][3];
		/* UV space footprint */
		cone_footprint = BTDF_BIAS * 0.5 * max(ProjectionMatrix[0][0], ProjectionMatrix[1][1]) * cone_footprint / homcoord;

		vec2 hit_uvs = project_point(ProjectionMatrix, hit_pos).xy * 0.5 + 0.5;

		/* Texel footprint */
		vec2 texture_size = vec2(textureSize(colorBuffer, 0).xy);
		float mip = clamp(log2(cone_footprint * max(texture_size.x, texture_size.y)), 0.0, 9.0);

		/* Correct UVs for mipmaping mis-alignment */
		float low_mip = floor(mip);
		hit_uvs *= mix(mipRatio[int(low_mip)], mipRatio[int(low_mip + 1.0)], mip - low_mip);

		vec3 spec = textureLod(colorBuffer, hit_uvs, mip).xyz;
		float mask = screen_border_mask(hit_uvs);

		return vec4(spec, mask);
	}

	return vec4(0.0);
}

```


<br>

*lit_surface_frag.glsl*

```
#ifdef USE_REFRACTION
uniform float refractionThickness;
#endif

...

vec3 eevee_surface_refraction(vec3 N, vec3 f0, float roughness, float ior, int ssr_id, out vec3 ssr_spec)
{
	/* Zero length vectors cause issues, see: T51979. */
#if 0
	N = normalize(N);
#else
	{
		float len = length(N);
		if (isnan(len)) {
			return vec3(0.0);
		}
		N /= len;
	}
#endif
	vec3 V = cameraVec;
	ior = (gl_FrontFacing) ? ior : 1.0 / ior;

	roughness = clamp(roughness, 1e-8, 0.9999);
	float roughnessSquared = roughness * roughness;

	/* ---------------- SCENE LAMPS LIGHTING ----------------- */

	/* No support for now. Supporting LTCs mean having a 3D LUT.
	 * We could support point lights easily though. */

	/* ---------------- SPECULAR ENVIRONMENT LIGHTING ----------------- */

	/* Accumulate light from all sources until accumulator is full. Then apply Occlusion and BRDF. */
	vec4 trans_accum = vec4(0.0);

#ifdef USE_REFRACTION
	/* Screen Space Refraction */
	if (ssrToggle && roughness < maxRoughness + 0.2) {
		vec3 rand = texture(utilTex, vec3(gl_FragCoord.xy / LUT_SIZE, 2.0)).xzw;

		float ray_ofs = 1.0 / float(rayCount);
		vec4 spec = screen_space_refraction(viewPosition, N, V, ior, roughnessSquared, rand, 0.0);
		if (rayCount > 1) spec += screen_space_refraction(viewPosition, N, V, ior, roughnessSquared, rand.xyz * vec3(1.0, -1.0, -1.0), 1.0 * ray_ofs);
		if (rayCount > 2) spec += screen_space_refraction(viewPosition, N, V, ior, roughnessSquared, rand.xzy * vec3(1.0,  1.0, -1.0), 2.0 * ray_ofs);
		if (rayCount > 3) spec += screen_space_refraction(viewPosition, N, V, ior, roughnessSquared, rand.xzy * vec3(1.0, -1.0,  1.0), 3.0 * ray_ofs);
		spec /= float(rayCount);
		spec.a *= smoothstep(maxRoughness + 0.2, maxRoughness, roughness);
		accumulate_light(spec.rgb, spec.a, trans_accum);
	}
#endif

	/* Specular probes */
	/* NOTE: This bias the IOR */
	vec3 spec_dir = get_specular_refraction_dominant_dir(N, V, roughness, ior);

	/* Starts at 1 because 0 is world probe */
	for (int i = 1; i < MAX_PROBE && i < probe_count && trans_accum.a < 0.999; ++i) {
		CubeData cd = probes_data[i];

		float fade = probe_attenuation_cube(cd, worldPosition);

		if (fade > 0.0) {
			vec3 spec = probe_evaluate_cube(float(i), cd, worldPosition, spec_dir, roughnessSquared);
			accumulate_light(spec, fade, trans_accum);
		}
	}

	/* World Specular */
	if (trans_accum.a < 0.999) {
		vec3 spec = probe_evaluate_world_spec(spec_dir, roughnessSquared);
		accumulate_light(spec, 1.0, trans_accum);
	}

	float btdf = get_btdf_lut(utilTex, dot(N, V), roughness, ior);

	return trans_accum.rgb * btdf;
}
```

>
- 上面得shader 也是想SSR 那套，raycast 发射线来检查碰撞