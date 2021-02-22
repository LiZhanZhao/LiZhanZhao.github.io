---
layout:     post
title:      "blender eevee Transparency Add support for Clip and Stochastic shadows."
subtitle:   ""
date:       2021-2-22 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/11  *  Eevee: Transparency: Add support for Clip and Stochastic shadows .<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
*Opaque* 
![](/img/Eevee/AlphaClipShadow/1.png)
<br>
*Alpha Clip*
![](/img/Eevee/AlphaClipShadow/2.png)
<br>
*Alpha Blend*
![](/img/Eevee/AlphaClipShadow/3.png)


## 作用
Alpha Clip 影响 Shadow

## 编译
- 重新生成SLN
- git 定位到  2017/7/11  * Eevee: Transparency: Add support for Clip and Stochastic shadows .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 渲染前
*eevee_materials.c*
```
void EEVEE_materials_cache_populate(EEVEE_Data *vedata, EEVEE_SceneLayerData *sldata, Object *ob)
{
	...
	/* Get per-material split surface */
	struct Gwn_Batch **mat_geom = DRW_cache_object_surface_material_get(ob, gpumat_array, materials_len);
	if (mat_geom) {
		for (int i = 0; i < materials_len; ++i) {
			Material *ma = give_current_material(ob, i + 1);

			if (ma == NULL)
				ma = &defmaterial;

			/* Shading pass */
			ADD_SHGROUP_CALL(shgrp_array[i], ob, mat_geom[i]);

			/* Depth Prepass */
			ADD_SHGROUP_CALL_SAFE(shgrp_depth_array[i], ob, mat_geom[i]);
			ADD_SHGROUP_CALL_SAFE(shgrp_depth_clip_array[i], ob, mat_geom[i]);

			/* Shadow Pass */
			if (ma->blend_method == MA_BM_SOLID)
				EEVEE_lights_cache_shcaster_add(sldata, psl, mat_geom[i], ob->obmat);
			else if (ma->blend_method == MA_BM_HASHED) {
				struct GPUMaterial *gpumat = EEVEE_material_mesh_depth_get(scene, ma, true, true);
				EEVEE_lights_cache_shcaster_material_add(sldata, psl, gpumat, mat_geom[i], ob->obmat, NULL);
			}
			else if (ma->blend_method == MA_BM_CLIP) {
				struct GPUMaterial *gpumat = EEVEE_material_mesh_depth_get(scene, ma, false, true);
				EEVEE_lights_cache_shcaster_material_add(sldata, psl, gpumat, mat_geom[i], ob->obmat, &ma->alpha_threshold);
			}
		}
	}
	...
}
```
>
- 关注 Shadow Pass 的代码部分
- MA_BM_SOLID 对应 EEVEE_lights_cache_shcaster_add
- MA_BM_HASHED 对应 EEVEE_lights_cache_shcaster_material_add(sldata, psl, gpumat, mat_geom[i], ob->obmat, NULL);
- MA_BM_CLIP 对应 EEVEE_lights_cache_shcaster_material_add(sldata, psl, gpumat, mat_geom[i], ob->obmat, &ma->alpha_threshold);
- 上面的函数都是说明材质和 shadow_cube_pass 进行关联


## MA_BM_SOLID

*eevee_lights.c*
```
void EEVEE_lights_init(EEVEE_SceneLayerData *sldata)
{
	...
	e_data.shadow_sh = DRW_shader_create(
		        datatoc_shadow_vert_glsl, datatoc_shadow_geom_glsl, datatoc_shadow_frag_glsl, NULL);
	...
}

/* Add a shadow caster to the shadowpasses */
void EEVEE_lights_cache_shcaster_add(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, struct Gwn_Batch *geom, float (*obmat)[4])
{
	DRWShadingGroup *grp = DRW_shgroup_instance_create(e_data.shadow_sh, psl->shadow_cube_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	for (int i = 0; i < 6; ++i)
		DRW_shgroup_call_dynamic_add_empty(grp);

	grp = DRW_shgroup_instance_create(e_data.shadow_sh, psl->shadow_cascade_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	for (int i = 0; i < MAX_CASCADE_NUM; ++i)
		DRW_shgroup_call_dynamic_add_empty(grp);
}
```
>
- MA_BM_SOLID 的材质关联 shadow_cube_pass
- MA_BM_SOLID 的材质的Shader使用 shadow_vert.glsl, shadow_geom.glsl, shadow_frag.glsl ，没有定义SHADOW_SHADER



## MA_BM_HASHED 和 MA_BM_CLIP

*eevee_materials.c*
```
struct GPUMaterial *EEVEE_material_mesh_depth_get(
        struct Scene *scene, Material *ma,
        bool use_hashed_alpha, bool is_shadow)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	int options = VAR_MAT_MESH;

	if (use_hashed_alpha) {
		options |= VAR_MAT_HASH;
	}
	else {
		options |= VAR_MAT_CLIP;
	}

	if (is_shadow)
		options |= VAR_MAT_SHADOW;

	GPUMaterial *mat = GPU_material_from_nodetree_find(&ma->gpumaterial, engine, options);
	if (mat) {
		return mat;
	}

	char *defines = eevee_get_defines(options);

	DynStr *ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, e_data.frag_shader_lib);
	BLI_dynstr_append(ds_frag, datatoc_prepass_frag_glsl);
	char *frag_str = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	mat = GPU_material_from_nodetree(
	        scene, ma->nodetree, &ma->gpumaterial, engine, options,
	        (is_shadow) ? datatoc_shadow_vert_glsl : datatoc_lit_surface_vert_glsl,
	        (is_shadow) ? datatoc_shadow_geom_glsl : NULL,
	        frag_str,
	        defines);

	MEM_freeN(frag_str);
	MEM_freeN(defines);

	return mat;
}
```
<br><br>

*eevee_lights.c*
```
void EEVEE_lights_cache_shcaster_material_add(
	EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, struct GPUMaterial *gpumat, struct Gwn_Batch *geom, float (*obmat)[4], float *alpha_threshold)
{
	DRWShadingGroup *grp = DRW_shgroup_material_instance_create(gpumat, psl->shadow_cube_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	if (alpha_threshold != NULL)
		DRW_shgroup_uniform_float(grp, "alphaThreshold", alpha_threshold, 1);

	for (int i = 0; i < 6; ++i)
		DRW_shgroup_call_dynamic_add_empty(grp);

	grp = DRW_shgroup_material_instance_create(gpumat, psl->shadow_cascade_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	if (alpha_threshold != NULL)
		DRW_shgroup_uniform_float(grp, "alphaThreshold", alpha_threshold, 1);

	for (int i = 0; i < MAX_CASCADE_NUM; ++i)
		DRW_shgroup_call_dynamic_add_empty(grp);
}
```

>
- 
```
struct GPUMaterial *gpumat = EEVEE_material_mesh_depth_get(scene, ma, true, true);
EEVEE_lights_cache_shcaster_material_add(sldata, psl, gpumat, mat_geom[i], ob->obmat, NULL);
```
- MA_BM_HASHED 的材质关联 shadow_cube_pass
- MA_BM_HASHED 的材质的Shader使用 shadow_vert.glsl, shadow_geom.glsl, prepass_frag.glsl，这里并且定义了宏 USE_ALPHA_HASH  和 SHADOW_SHADER
<br><br><br>
```
struct GPUMaterial *gpumat = EEVEE_material_mesh_depth_get(scene, ma, false, true);
EEVEE_lights_cache_shcaster_material_add(sldata, psl, gpumat, mat_geom[i], ob->obmat, &ma->alpha_threshold);
```
- MA_BM_CLIP 的材质关联 shadow_cube_pass
- MA_BM_CLIP 的材质的Shader使用 shadow_vert.glsl, shadow_geom.glsl, prepass_frag.glsl，这里并且定义了宏 USE_ALPHA_CLIP  和 SHADOW_SHADER
- MA_BM_CLIP 也会传入alpha_threshold参数进行Alpha Clip



## 渲染
*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Refresh shadows */
	EEVEE_draw_shadows(sldata, psl);
	...
}
```
>
- EEVEE_draw_shadows 主要是调用 shadow_cube_pass 进行渲染



## Shader
*shadow_vert.glsl*
```
uniform mat4 ShadowModelMatrix;
#ifdef MESH_SHADER
uniform mat3 WorldNormalMatrix;
#endif

in vec3 pos;
#ifdef MESH_SHADER
in vec3 nor;
#endif

out vec4 vPos;
#ifdef MESH_SHADER
out vec3 vNor;
#endif

flat out int face;

void main() {
	vPos = ShadowModelMatrix * vec4(pos, 1.0);
	face = gl_InstanceID;

#ifdef MESH_SHADER
	vNor = WorldNormalMatrix * nor;
#ifdef ATTRIB
	pass_attrib(pos);
#endif
#endif
}

```

<br><br>

*shadow_geom.glsl*
```
layout(std140) uniform shadow_render_block {
	mat4 ShadowMatrix[6];
	mat4 FaceViewMatrix[6];
	vec4 lampPosition;
	int layer;
	float exponent;
};

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

in vec4 vPos[];
flat in int face[];

#ifdef MESH_SHADER
in vec3 vNor[];
in vec3 vNor[];
#endif

out vec3 worldPosition;
#ifdef MESH_SHADER
out vec3 viewPosition; /* Required. otherwise generate linking error. */
out vec3 worldNormal; /* Required. otherwise generate linking error. */
out vec3 viewNormal; /* Required. otherwise generate linking error. */
flat out int shFace;
#else
int shFace;
#endif

void main() {
	shFace = face[0];
	gl_Layer = shFace;

	for (int v = 0; v < 3; ++v) {
		gl_Position = ShadowMatrix[shFace] * vPos[v];
		worldPosition = vPos[v].xyz;
#ifdef MESH_SHADER
		worldNormal = vNor[v];
		viewPosition = (FaceViewMatrix[shFace] * vec4(worldPosition, 1.0)).xyz;
		viewNormal = (FaceViewMatrix[shFace] * vec4(worldNormal, 0.0)).xyz;
#ifdef ATTRIB
		pass_attrib(v);
#endif
#endif
		EmitVertex();
	}

	EndPrimitive();
}
```

<br><br>

*shadow_frag.glsl*
```
layout(std140) uniform shadow_render_block {
	mat4 ShadowMatrix[6];
	mat4 FaceViewMatrix[6];
	vec4 lampPosition;
	int layer;
	float exponent;
};

in vec3 worldPosition;

out vec4 FragColor;

void main() {
	float dist = distance(lampPosition.xyz, worldPosition.xyz);
	FragColor = vec4(dist, 0.0, 0.0, 1.0);
}
```

<br><br>
*prepass_frag.glsl*
```

#ifdef USE_ALPHA_HASH

/* From the paper "Hashed Alpha Testing" by Chris Wyman and Morgan McGuire */
float hash(vec2 a) {
	return fract(1e4 * sin(17.0 * a.x + 0.1 * a.y) * (0.1 + abs(sin(13.0 * a.y + a.x))));
}

float hash3d(vec3 a) {
	return hash(vec2(hash(a.xy), a.z));
}

//uniform float hashScale;
float hashScale = 0.001;

float hashed_alpha_threshold(vec3 co)
{
	/* Find the discretized derivatives of our coordinates. */
	float max_deriv = max(length(dFdx(co)), length(dFdy(co)));
	float pix_scale = 1.0 / (hashScale * max_deriv);

	/* Find two nearest log-discretized noise scales. */
	float pix_scale_log = log2(pix_scale);
	vec2 pix_scales;
	pix_scales.x = exp2(floor(pix_scale_log));
	pix_scales.y = exp2(ceil(pix_scale_log));

	/* Compute alpha thresholds at our two noise scales. */
	vec2 alpha;
	alpha.x = hash3d(floor(pix_scales.x * co));
	alpha.y = hash3d(floor(pix_scales.y * co));

	/* Factor to interpolate lerp with. */
	float fac = fract(log2(pix_scale));

	/* Interpolate alpha threshold from noise at two scales. */
	float x = mix(alpha.x, alpha.y, fac);

	/* Pass into CDF to compute uniformly distrib threshold. */
	float a = min(fac, 1.0 - fac);
	float one_a = 1.0 - a;
	float denom = 1.0 / (2 * a * one_a);
	float one_x = (1 - x);
	vec3 cases = vec3(
		(x * x) * denom,
		(x - 0.5 * a) / one_a,
		1.0 - (one_x * one_x * denom)
	);

	/* Find our final, uniformly distributed alpha threshold. */
	float threshold = (x < one_a) ?	((x < a) ? cases.x : cases.y) :	cases.z;

	/* Avoids threshold == 0. */
	threshold = clamp(threshold, 1.0e-6, 1.0);

	return threshold;
}

#endif

#ifdef USE_ALPHA_CLIP
uniform float alphaThreshold;
#endif

#ifdef SHADOW_SHADER
out vec4 FragColor;
#endif

void main()
{
	/* For now do nothing.
	 * In the future, output object motion blur. */

#if defined(USE_ALPHA_HASH) || defined(USE_ALPHA_CLIP)
#define NODETREE_EXEC

	Closure cl = nodetree_exec();

#if defined(USE_ALPHA_HASH)
	/* Hashed Alpha Testing */
	if (cl.opacity < hashed_alpha_threshold(worldPosition))
		discard;
#elif defined(USE_ALPHA_CLIP)
	/* Alpha clip */
	if (cl.opacity <= alphaThreshold)
		discard;
#endif
#endif

#ifdef SHADOW_SHADER
	float dist = distance(lampPosition.xyz, worldPosition.xyz);
	FragColor = vec4(dist, 0.0, 0.0, 1.0);
#endif
}

```
>
- 主要的是Alpha Clip 来进行discard，达到Prepass。


<br><br><br>

# Transparency Add transparent Shadow method UI.

## 来源

- 主要看这个commit

> GIT : 2017/7/12  *   Eevee: Transparency: Fix crash when using transparent shadows .<br> 
 
> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果

![](/img/Eevee/AlphaClipShadow/4.png)


## 作用
透明物体有影子效果

## 编译
- 重新生成SLN

- git 定位到 2017/7/12   Eevee: Transparency: Fix crash when using transparent shadows.<br>
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## 注意
主要的代码在 2017/7/11  *  Eevee: Transparency: Add transparent Shadow method UI .<br> 

## Code
*eevee_materials.c*
```
void EEVEE_materials_cache_populate(EEVEE_Data *vedata, EEVEE_SceneLayerData *sldata, Object *ob)
{
	...
	/* Get per-material split surface */
		struct Gwn_Batch **mat_geom = DRW_cache_object_surface_material_get(ob, gpumat_array, materials_len);
		if (mat_geom) {
			for (int i = 0; i < materials_len; ++i) {
				Material *ma = give_current_material(ob, i + 1);

				if (ma == NULL)
					ma = &defmaterial;

				/* Shading pass */
				ADD_SHGROUP_CALL(shgrp_array[i], ob, mat_geom[i]);

				/* Depth Prepass */
				ADD_SHGROUP_CALL_SAFE(shgrp_depth_array[i], ob, mat_geom[i]);
				ADD_SHGROUP_CALL_SAFE(shgrp_depth_clip_array[i], ob, mat_geom[i]);

				/* Shadow Pass */
				if (ma->blend_method != MA_BM_SOLID) {
					struct GPUMaterial *gpumat;
					switch (ma->blend_shadow) {
						case MA_BS_SOLID:
							EEVEE_lights_cache_shcaster_add(sldata, psl, mat_geom[i], ob->obmat);
							break;
						case MA_BS_CLIP:
							gpumat = EEVEE_material_mesh_depth_get(scene, ma, false, true);
							EEVEE_lights_cache_shcaster_material_add(sldata, psl, gpumat, mat_geom[i], ob->obmat, &ma->alpha_threshold);
							break;
						case MA_BS_HASHED:
							gpumat = EEVEE_material_mesh_depth_get(scene, ma, true, true);
							EEVEE_lights_cache_shcaster_material_add(sldata, psl, gpumat, mat_geom[i], ob->obmat, NULL);
							break;
						case MA_BS_NONE:
						default:
							break;
					}
				}
				else {
					EEVEE_lights_cache_shcaster_add(sldata, psl, mat_geom[i], ob->obmat);
				}
			}
		}
	...
}
```
>
- /* Shadow Pass */  部分代码，结合上面的应该不难理解，就是 ma->blend_method != MA_BM_SOLID 的时候，都进行prepass