---
layout:     post
title:      "blender eevee SSS Add Translucency support"
subtitle:   ""
date:       2021-4-12 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/11/22  *   Eevee : SSS : Add Translucency support. <br> 

> 
This adds the possibility to simulate things like red ears with strong backlight or material with high scattering distances.

>
To enable it you need to turn on the "Subsurface Translucency" option in the "Options" tab of the Material Panel (and of course to have "regular" SSS enabled in both render settings and material options).
Since the effect is adding another overhead I prefer to make it optional. But this is open to discussion.

>
Be aware that the effect only works for direct lights (so no indirect/world lighting) that have shadowmaps, and is affected by the "softness" of the shadowmap and resolution.

> Technical notes:
This is inspired by http://www.iryoku.com/translucency/ but goes a bit beyond that.
We do not use a sum of gaussian to apply in regards to the object thickness but we precompute a 1D kernel texture.
This texture stores the light transmited to a point at the back of an infinite slab of material of variying thickness.
We make the assumption that the slab is perpendicular to the light so that no fresnel or diffusion term is taken into account.
The light is considered constant.
If the setup is similar to the one assume during the profile baking, the realtime render matches cycles reference.
Due to these assumptions the computed transmitted light is in most cases too bright for curvy objects.

>
Finally we jitter the shadow map sample per pixel so we can simulate dispersion inside the medium.
Radius of the dispersion is in world space and derived by from the "soft" shadowmap parameter.
Idea for this come from this presentation http://www.iryoku.com/stare-into-the-future (slide 164).


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
实现SSS Translucency 效果
<br><br>

## 效果

![](/img/Eevee/SSS/02/1.png)


<br><br>


## 准备
先看看 [这里](http://shaderstore.cn/2021/04/09/blender-eevee-2017-11-14-Initial-Separable-Subsurface-Scattering-Implementation/),

<br><br>

### Shader 

#### 1.node_subsurface_scattering 
*gpu_shader_material.glsl*
```
void node_subsurface_scattering(
        vec4 color, float scale, vec3 radius, float sharpen, float texture_blur, vec3 N, float sss_id,
        out Closure result)
{
#if defined(EEVEE_ENGINE) && defined(USE_SSS)
	vec3 vN = normalize(mat3(ViewMatrix) * N);
	result = CLOSURE_DEFAULT;
	result.ssr_data = vec4(0.0);
	result.ssr_normal = normal_encode(vN, viewCameraVec);
	result.ssr_id = -1;
	result.sss_data.rgb = eevee_surface_translucent_lit(N, color.rgb, scale);
	result.sss_data.rgb += eevee_surface_diffuse_lit(N, color.rgb, 1.0);
	result.sss_data.a = scale;
#else
	node_bsdf_diffuse(color, 0.0, N, result);
#endif
}
```
>
- 这里可以看到 在 node tree 的 subsurface scattering 节点上，多了 eevee_surface_translucent_lit 的计算

<br><br>

#### 2.eevee_surface_translucent_lit

*lit_surface_frag.glsl*
```
/* ----------- Translucency -----------  */

vec3 eevee_surface_translucent_lit(vec3 N, vec3 albedo, float sss_scale)
{
#ifndef USE_TRANSLUCENCY
	return vec3(0.0);
#endif

	vec3 V = cameraVec;

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

	/* We only enlit the backfaces */
	N = -N;

	/* ---------------- SCENE LAMPS LIGHTING ----------------- */

#ifdef HAIR_SHADER
	vec3 norm_view = cross(V, N);
	norm_view = normalize(cross(norm_view, N)); /* Normal facing view */
#endif

	vec3 diff = vec3(0.0);
	for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
		LightData ld = lights_data[i];

		vec4 l_vector; /* Non-Normalized Light Vector with length in last component. */
		l_vector.xyz = ld.l_position - worldPosition;
		l_vector.w = length(l_vector.xyz);

#ifdef HAIR_SHADER
		vec3 norm_lamp, view_vec;
		float occlu_trans, occlu;
		light_hair_common(ld, N, V, l_vector, norm_view, occlu_trans, occlu, norm_lamp, view_vec);

		diff += ld.l_color * light_translucent(ld, worldPosition, norm_lamp, l_vector, sss_scale) * occlu_trans;
#else
		diff += ld.l_color * light_translucent(ld, worldPosition, N, l_vector, sss_scale);
#endif
	}

	/* Accumulate outgoing radiance */
	vec3 out_light = diff * albedo;

	return out_light;
}
```
>
- 主要的效果都是在 light_translucent 里计算


<br><br>


#### 3.light_translucent
*lamps_lib.glsl*
```

#define MAX_SSS_SAMPLES 65
#define SSS_LUT_SIZE 64.0
#define SSS_LUT_SCALE ((SSS_LUT_SIZE - 1.0) / float(SSS_LUT_SIZE))
#define SSS_LUT_BIAS (0.5 / float(SSS_LUT_SIZE))
layout(std140) uniform sssProfile {
	vec4 kernel[MAX_SSS_SAMPLES];
	vec4 radii_max_radius;
	int sss_samples;
};

uniform sampler1D sssTexProfile;

vec3 sss_profile(float s) {
	s /= radii_max_radius.w;
	return texture(sssTexProfile, saturate(s) * SSS_LUT_SCALE + SSS_LUT_BIAS).rgb;
}

vec3 light_translucent(LightData ld, vec3 W, vec3 N, vec4 l_vector, float scale)
{
	vec3 vis = vec3(1.0);

	/* Only shadowed light can produce translucency */
	if (ld.l_shadowid >= 0.0) {
		ShadowData data = shadows_data[int(ld.l_shadowid)];
		float delta;

		vec4 L = (ld.l_type != SUN) ? l_vector : vec4(-ld.l_forward, 1.0);

		vec3 T, B;
		make_orthonormal_basis(L.xyz / L.w, T, B);

		vec3 rand = texture(utilTex, vec3(gl_FragCoord.xy / LUT_SIZE, 2.0)).xzw;
		/* XXX This is a hack to not have noise correlation artifacts.
		 * A better solution to have better noise is welcome. */
		rand.yz *= fast_sqrt(fract(rand.x * 7919.0)) * data.sh_blur;

		/* We use the full l_vector.xyz so that the spread is minimize
		 * if the shading point is further away from the light source */
		W = W + T * rand.y + B * rand.z;

		if (ld.l_type == SUN) {
			ShadowCascadeData scd = shadows_cascade_data[int(data.sh_data_start)];
			vec4 view_z = vec4(dot(W - cameraPos, cameraForward));

			vec4 weights = step(scd.split_end_distances, view_z);
			float id = abs(4.0 - dot(weights, weights));

			if (id > 3.0) {
				return vec3(0.0);
			}

			float range = abs(data.sh_far - data.sh_near); /* Same factor as in get_cascade_world_distance(). */

			vec4 shpos = scd.shadowmat[int(id)] * vec4(W, 1.0);
			float dist = shpos.z * range;

			if (shpos.z > 1.0 || shpos.z < 0.0) {
				return vec3(0.0);
			}

#if defined(SHADOW_VSM)
			vec2 moments = texture(shadowTexture, vec3(shpos.xy, data.sh_tex_start + id)).rg;
			delta = dist - moments.x;
#else
			float z = texture(shadowTexture, vec3(shpos.xy, data.sh_tex_start + id)).r;
			delta = dist - z;
#endif
		}
		else {
			vec3 cubevec = W - shadows_cube_data[int(data.sh_data_start)].position.xyz;
			float dist = length(cubevec);

			/* If fragment is out of shadowmap range, do not occlude */
			/* XXX : we check radial distance against a cubeface distance.
			 * We loose quite a bit of valid area. */
			if (dist < data.sh_far) {
				cubevec /= dist;

#if defined(SHADOW_VSM)
				vec2 moments = texture_octahedron(shadowTexture, vec4(cubevec, data.sh_tex_start)).rg;
				delta = dist - moments.x;
#else
				float z = texture_octahedron(shadowTexture, vec4(cubevec, data.sh_tex_start)).r;
				delta = dist - z;
#endif
			}
		}

		/* XXX : Removing Area Power. */
		/* TODO : put this out of the shader. */
		float falloff;
		if (ld.l_type == AREA) {
			vis *= 0.0962 * (ld.l_sizex * ld.l_sizey * 4.0 * M_PI);
			vis /= (l_vector.w * l_vector.w);
			falloff = dot(N, l_vector.xyz / l_vector.w);
		}
		else if (ld.l_type == SUN) {
			falloff = dot(N, -ld.l_forward);
		}
		else {
			vis *= 0.0248 * (4.0 * ld.l_radius * ld.l_radius * M_PI * M_PI);
			vis /= (l_vector.w * l_vector.w);
			falloff = dot(N, l_vector.xyz / l_vector.w);
		}
		vis *= M_1_PI; /* Normalize */

		/* Applying profile */
		vis *= sss_profile(abs(delta) / scale);

		/* No transmittance at grazing angle (hide artifacts) */
		vis *= saturate(falloff * 2.0);

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
	}
	else {
		vis = vec3(0.0);
	}

	return vis;
}

```
>
- 这里需要留意的是 Applying profile : vis *= sss_profile(abs(delta) / scale);
<br><br>
- sss_profile 里面主要采样  sampler1D sssTexProfile


<br><br>


#### 4.sampler1D sssTexProfile
*eevee_materials.c*
```
static void material_opaque(
        Material *ma, GHash *material_hash, EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata,
        bool do_cull, bool use_flat_nor, struct GPUMaterial **gpumat, struct GPUMaterial **gpumat_depth,
        struct DRWShadingGroup **shgrp, struct DRWShadingGroup **shgrp_depth, struct DRWShadingGroup **shgrp_depth_clip)
{
	...
	const bool use_translucency = ((ma->blend_flag & MA_BL_TRANSLUCENCY) != 0) && ((stl->effects->enabled_effects & EFFECT_SSS) != 0);

	...

	/* Shading */
	*gpumat = EEVEE_material_mesh_get(scene, ma, vedata, false, false, use_refract,
										use_sss, use_translucency, linfo->shadow_method);

	...
	if (use_sss) {
		struct GPUTexture *sss_tex_profile = NULL;
		struct GPUUniformBuffer *sss_profile = GPU_material_sss_profile_get(*gpumat,
																			stl->effects->sss_sample_count,
																			&sss_tex_profile);

		if (sss_profile) {
			if (use_translucency) {
				DRW_shgroup_uniform_block(*shgrp, "sssProfile", sss_profile);
				DRW_shgroup_uniform_texture(*shgrp, "sssTexProfile", sss_tex_profile);
			}

			DRW_shgroup_stencil_mask(*shgrp, e_data.sss_count + 1);
			EEVEE_subsurface_add_pass(vedata, e_data.sss_count + 1, sss_profile);
			e_data.sss_count++;
		}
	}					
}
```
>
- 这里可以看到 sss_tex_profile 的计算是在 GPU_material_sss_profile_get 里面进行的
<br><br>
- 如果是使用了 Translucency 的话，Shader会进行定义宏 #define USE_TRANSLUCENCY

<br><br>

#### 5.GPU_material_sss_profile_get

*gpu_material.c*

```

#define INTEGRAL_RESOLUTION 512
static void compute_sss_translucence_kernel(
        const GPUSssKernelData *kd, int resolution, short falloff_type, float sharpness, float **output)
{
	float (*texels)[4];
	texels = MEM_callocN(sizeof(float) * 4 * resolution, "compute_sss_translucence_kernel");
	*output = (float *)texels;

	/* Last texel should be black, hence the - 1. */
	for (int i = 0; i < resolution - 1; ++i) {
		/* Distance from surface. */
		float d = kd->max_radius * ((float)i + 0.00001f) / ((float)resolution);

		/* For each distance d we compute the radiance incomming from an hypothetic parallel plane. */
		/* Compute radius of the footprint on the hypothetic plane */
		float r_fp = sqrtf(kd->max_radius * kd->max_radius - d * d);
		float r_step = r_fp / INTEGRAL_RESOLUTION;
		float area_accum = 0.0f;
		for (float r = 0.0f; r < r_fp; r += r_step) {
			/* Compute distance to the "shading" point through the medium. */
			/* r_step * 0.5f to put sample between the area borders */
			float dist = hypotf(r + r_step * 0.5f, d);

			float profile[3];
			profile[0] = eval_profile(dist, falloff_type, sharpness, kd->param[0]);
			profile[1] = eval_profile(dist, falloff_type, sharpness, kd->param[1]);
			profile[2] = eval_profile(dist, falloff_type, sharpness, kd->param[2]);

			/* Since the profile and configuration are radially symetrical we
			 * can just evaluate it once and weight it accordingly */
			float r_next = r + r_step;
			float disk_area = (M_PI * r_next * r_next) - (M_PI * r * r);

			mul_v3_fl(profile, disk_area);
			add_v3_v3(texels[i], profile);
			area_accum += disk_area;
		}
		/* Normalize over the disk. */
		mul_v3_fl(texels[i], 1.0f / (area_accum));
	}

	/* Normalize */
	for (int j = resolution - 2; j > 0; j--) {
		texels[j][0] /= (texels[0][0] > 0.0f) ? texels[0][0] : 1.0f;
		texels[j][1] /= (texels[0][1] > 0.0f) ? texels[0][1] : 1.0f;
		texels[j][2] /= (texels[0][2] > 0.0f) ? texels[0][2] : 1.0f;
	}

	/* First texel should be white */
	texels[0][0] = (texels[0][0] > 0.0f) ? 1.0f : 0.0f;
	texels[0][1] = (texels[0][1] > 0.0f) ? 1.0f : 0.0f;
	texels[0][2] = (texels[0][2] > 0.0f) ? 1.0f : 0.0f;

	/* dim the last few texels for smoother transition */
	mul_v3_fl(texels[resolution - 2], 0.25f);
	mul_v3_fl(texels[resolution - 3], 0.5f);
	mul_v3_fl(texels[resolution - 4], 0.75f);
}
#undef INTEGRAL_RESOLUTION

struct GPUUniformBuffer *GPU_material_sss_profile_get(GPUMaterial *material, int sample_ct, GPUTexture **tex_profile)
{
	if (material->sss_radii == NULL)
		return NULL;

	if (material->sss_dirty || (material->sss_samples != sample_ct)) {
		GPUSssKernelData kd;

		float sharpness = (material->sss_sharpness != NULL) ? *material->sss_sharpness : 0.0f;

		/* XXX Black magic but it seems to fit. Maybe because we integrate -1..1 */
		sharpness *= 0.5f;

		compute_sss_kernel(&kd, material->sss_radii, sample_ct, *material->sss_falloff, sharpness);

		/* Update / Create UBO */
		GPU_uniformbuffer_update(material->sss_profile, &kd);

		/* Update / Create Tex */
		float *translucence_profile;
		compute_sss_translucence_kernel(&kd, 64, *material->sss_falloff, sharpness, &translucence_profile);

		if (material->sss_tex_profile != NULL) {
			GPU_texture_free(material->sss_tex_profile);
		}

		material->sss_tex_profile = GPU_texture_create_1D_custom(64, 4, GPU_RGBA16F, translucence_profile, NULL);

		MEM_freeN(translucence_profile);

		material->sss_samples = sample_ct;
		material->sss_dirty = false;
	}

	if (tex_profile != NULL) {
		*tex_profile = material->sss_tex_profile;
	}
	return material->sss_profile;
}
```
>
- 主要是留意 translucence_profile 的计算
<br><br>
- 理论todo
