---
layout:     post
title:      "blender eevee Add support for Alpha clip and Hashed Alpha transparency"
subtitle:   ""
date:       2021-2-21 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/9  *  Eevee: Add support for Alpha clip and Hashed Alpha transparency .<br> 

> Hashed Alpha transparency offers a noisy output but has the benefit of being correctly ordered. Noise can be attenuated with Multisampling / AntiAliasing.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/AlphaTest/1.png)
![](/img/Eevee/AlphaTest/2.png)
![](/img/Eevee/AlphaTest/3.png)

## 作用
Alpha Clip 和 Alpha Hashed transparency的实现

## 编译
- 重新生成SLN
- git 定位到  2017/7/9  * Eevee : Add support for Alpha clip and Hashed Alpha transparency .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## 渲染
*eevee_materials.c*
```
void EEVEE_materials_cache_init(EEVEE_Data *vedata)
{
	...
	{
		DRWState state = DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS | DRW_STATE_WIRE;
		psl->depth_pass = DRW_pass_create("Depth Pass", state);
		stl->g_data->depth_shgrp = DRW_shgroup_create(e_data.default_prepass_sh, psl->depth_pass);

		state = DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS | DRW_STATE_CULL_BACK;
		psl->depth_pass_cull = DRW_pass_create("Depth Pass Cull", state);
		stl->g_data->depth_shgrp_cull = DRW_shgroup_create(e_data.default_prepass_sh, psl->depth_pass_cull);

		state = DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS | DRW_STATE_CLIP_PLANES | DRW_STATE_WIRE;
		psl->depth_pass_clip = DRW_pass_create("Depth Pass Clip", state);
		stl->g_data->depth_shgrp_clip = DRW_shgroup_create(e_data.default_prepass_clip_sh, psl->depth_pass_clip);

		state = DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS | DRW_STATE_CLIP_PLANES | DRW_STATE_CULL_BACK;
		psl->depth_pass_clip_cull = DRW_pass_create("Depth Pass Cull Clip", state);
		stl->g_data->depth_shgrp_clip_cull = DRW_shgroup_create(e_data.default_prepass_clip_sh, psl->depth_pass_clip_cull);
	}

	{
		DRWState state = DRW_STATE_WRITE_COLOR | DRW_STATE_DEPTH_EQUAL | DRW_STATE_CLIP_PLANES | DRW_STATE_WIRE;
		psl->material_pass = DRW_pass_create("Material Shader Pass", state);
	}
	...
}
```

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Depth prepass */
	DRW_draw_pass(psl->depth_pass);
	DRW_draw_pass(psl->depth_pass_cull);
	...
	DRW_draw_pass(psl->material_pass);
	...
}
```
>
- 这里可以看到先渲染 psl->depth_pass, 再渲染 psl->material_pass。
- 其实这里就相当于 Depth Prepass 的一个过程，因为 psl->depth_pass 的渲染状态是 DRW_STATE_WRITE_DEPTH 和 DRW_STATE_DEPTH_LESS，而psl->material_pass的渲染状态是 DRW_STATE_DEPTH_EQUAL
- psl->depth_pass 相当于先渲染一次，但是只是先深度的，而 psl->material_pass 是渲染物体，写颜色，而且会进行深度测试，深度测试是 DRW_STATE_DEPTH_EQUAL。

<br><br>

*eevee_materials.c*
```
void EEVEE_materials_cache_populate(EEVEE_Data *vedata, EEVEE_SceneLayerData *sldata, Object *ob)
{
	...
	/* Get per-material split surface */
	struct Gwn_Batch **mat_geom = DRW_cache_object_surface_material_get(ob, gpumat_array, materials_len);
	if (mat_geom) {
		for (int i = 0; i < materials_len; ++i) {
			ADD_SHGROUP_CALL(shgrp_array[i], ob, mat_geom[i]);

			/* Depth Prepass */
			DRWShadingGroup *depth_shgrp = NULL;
			DRWShadingGroup *depth_clip_shgrp;

			Material *ma = give_current_material(ob, i + 1);

			if (ma != NULL && (ma->use_nodes && ma->nodetree)) {
				if (ELEM(ma->blend_method, MA_BM_CLIP, MA_BM_HASHED)) {
					Scene *scene = draw_ctx->scene;
					DRWPass *depth_pass, *depth_clip_pass;
					struct GPUMaterial *gpumat = EEVEE_material_mesh_depth_get(scene, ma, (ma->blend_method == MA_BM_HASHED));

					depth_pass = do_cull ? psl->depth_pass_cull : psl->depth_pass;
					depth_clip_pass = do_cull ? psl->depth_pass_clip_cull : psl->depth_pass_clip;

					/* Use same shader for both. */
					depth_shgrp = DRW_shgroup_material_create(gpumat, depth_pass);
					depth_clip_shgrp = DRW_shgroup_material_create(gpumat, depth_clip_pass);

					if (ma->blend_method == MA_BM_CLIP) {
						DRW_shgroup_uniform_float(depth_shgrp, "alphaThreshold", &ma->alpha_threshold, 1);
						DRW_shgroup_uniform_float(depth_clip_shgrp, "alphaThreshold", &ma->alpha_threshold, 1);
					}
				}

				/* Shadow Pass */
				/* TODO clipped shadow map */
				EEVEE_lights_cache_shcaster_add(sldata, psl, mat_geom[i], ob->obmat);
			}

			if (depth_shgrp == NULL) {
				depth_shgrp = do_cull ? stl->g_data->depth_shgrp_cull : stl->g_data->depth_shgrp;
				depth_clip_shgrp = do_cull ? stl->g_data->depth_shgrp_clip_cull : stl->g_data->depth_shgrp_clip;

				/* Shadow Pass */
				EEVEE_lights_cache_shcaster_add(sldata, psl, mat_geom[i], ob->obmat);
			}

			ADD_SHGROUP_CALL(depth_shgrp, ob, mat_geom[i]);
			ADD_SHGROUP_CALL(depth_clip_shgrp, ob, mat_geom[i]);
		}
	}
	...
}
```
> 
- 核心代码在这里
```
struct GPUMaterial *gpumat = EEVEE_material_mesh_depth_get(scene, ma, (ma->blend_method == MA_BM_HASHED));
depth_pass = do_cull ? psl->depth_pass_cull : psl->depth_pass;
depth_clip_pass = do_cull ? psl->depth_pass_clip_cull : psl->depth_pass_clip;
/* Use same shader for both. */
depth_shgrp = DRW_shgroup_material_create(gpumat, depth_pass);
depth_clip_shgrp = DRW_shgroup_material_create(gpumat, depth_clip_pass);
...
ADD_SHGROUP_CALL(depth_shgrp, ob, mat_geom[i]);
```

<br>

> ADD_SHGROUP_CALL(depth_shgrp, ob, mat_geom[i]);
- mat_geom 物体的几何体和 depth_shgrp 进行关联

<br>

> depth_shgrp = DRW_shgroup_material_create(gpumat, depth_pass);
- 这里是把gpumat材质跟depth_pass关联在一起，也就是在把这个gpumat材质加入到depth_pass里面，depth_pass在执行渲染的时候，就会渲染这个gpumat材质，gpumat的几何模型就是mat_geom[i]

<br>

> depth_pass = do_cull ? psl->depth_pass_cull : psl->depth_pass;
- 如果do_cell开启的话，会用 psl->depth_pass_cull 的渲染状态来渲染物体，如果 do_cell 没有开启的话，会用 psl->depth_pass 的渲染状态，psl->depth_pass_cull 和 psl->depth_pass 的渲染状态的区别就是是否有 DRW_STATE_CULL_BACK

<br>

> struct GPUMaterial *gpumat = EEVEE_material_mesh_depth_get(scene, ma, (ma->blend_method == MA_BM_HASHED));
- gpumat的Shader代码主要是来自于 EEVEE_material_mesh_depth_get 函数


## Shader
*eevee_materials.c*
```

static char *eevee_get_defines(int options)
{
	char *str = NULL;

	DynStr *ds = BLI_dynstr_new();
	BLI_dynstr_appendf(ds, SHADER_DEFINES);

	if ((options & VAR_MAT_MESH) != 0) {
		BLI_dynstr_appendf(ds, "#define MESH_SHADER\n");
	}
	if ((options & VAR_MAT_HAIR) != 0) {
		BLI_dynstr_appendf(ds, "#define HAIR_SHADER\n");
	}
	if ((options & VAR_MAT_PROBE) != 0) {
		BLI_dynstr_appendf(ds, "#define PROBE_CAPTURE\n");
	}
	if ((options & VAR_MAT_AO) != 0) {
		BLI_dynstr_appendf(ds, "#define USE_AO\n");
	}
	if ((options & VAR_MAT_FLAT) != 0) {
		BLI_dynstr_appendf(ds, "#define USE_FLAT_NORMAL\n");
	}
	if ((options & VAR_MAT_BENT) != 0) {
		BLI_dynstr_appendf(ds, "#define USE_BENT_NORMAL\n");
	}
	if ((options & VAR_MAT_CLIP) != 0) {
		BLI_dynstr_appendf(ds, "#define USE_ALPHA_CLIP\n");
	}
	if ((options & VAR_MAT_HASH) != 0) {
		BLI_dynstr_appendf(ds, "#define USE_ALPHA_HASH\n");
	}

	str = BLI_dynstr_get_cstring(ds);
	BLI_dynstr_free(ds);

	return str;
}

struct GPUMaterial *EEVEE_material_mesh_depth_get(struct Scene *scene, Material *ma, bool use_hashed_alpha)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	int options = VAR_MAT_MESH;

	if (use_hashed_alpha) {
		options |= VAR_MAT_HASH;
	}
	else {
		options |= VAR_MAT_CLIP;
	}

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
	        datatoc_lit_surface_vert_glsl, NULL, frag_str,
	        defines);

	MEM_freeN(frag_str);
	MEM_freeN(defines);

	return mat;
}
```
>
- 材质主要使用  lit_surface_vert.glsl 和 prepass_frag.glsl, 但是这里要判断 use_hashed_alpha 参数是否开启，这个 use_hashed_alpha 关联的是 宏 USE_ALPHA_CLIP 和 USE_ALPHA_HASH，如果不开启就使用 USE_ALPHA_CLIP ，否则就是 USE_ALPHA_HASH。

<br><br>

*lit_surface_vert.glsl*
```

uniform mat4 ModelViewProjectionMatrix;
uniform mat4 ModelMatrix;
uniform mat4 ModelViewMatrix;
uniform mat3 WorldNormalMatrix;
#ifndef ATTRIB
uniform mat3 NormalMatrix;
#endif

in vec3 pos;
in vec3 nor;

out vec3 worldPosition;
out vec3 viewPosition;

/* Used for planar reflections */
uniform vec4 ClipPlanes[1];

#ifdef USE_FLAT_NORMAL
flat out vec3 worldNormal;
flat out vec3 viewNormal;
#else
out vec3 worldNormal;
out vec3 viewNormal;
#endif

void main() {
	gl_Position = ModelViewProjectionMatrix * vec4(pos, 1.0);
	viewPosition = (ModelViewMatrix * vec4(pos, 1.0)).xyz;
	worldPosition = (ModelMatrix * vec4(pos, 1.0)).xyz;
	viewNormal = normalize(NormalMatrix * nor);
	worldNormal = normalize(WorldNormalMatrix * nor);

	/* Used for planar reflections */
	gl_ClipDistance[0] = dot(vec4(worldPosition, 1.0), ClipPlanes[0]);

#ifdef ATTRIB
	pass_attrib(pos);
#endif
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
}
```
>
- 这里最主要的还是判断alpha来确定是否进行discard.

<br><br>
## NodeTree生成的Shader代码

```
/* ********************** matcap style render ******************** */

void material_preview_matcap(vec4 color, sampler2D ima, vec4 N, vec4 mask, out vec4 result)
{
	...
}

//------------------------------------- 以上的都是gpu_shader_material.glsl代码
//------------------------------------- 以下才是生成的代码

in vec3 var0;
const mat4 cons2 = mat4(1.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 1.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 1.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 0.000000000000, 1.000000000000);
const vec3 cons3 = vec3(0.000000000000, 0.000000000000, 0.000000000000);
const vec3 cons4 = vec3(1.000000000000, 1.000000000000, 1.000000000000);
const float cons5 = float(0.000000000000);
const float cons6 = float(0.000000000000);
uniform sampler2D samp0;

Closure nodetree_exec(void)
{
	vec3 tmp7;
	vec4 tmp10;
	float tmp11;
	vec4 tmp13;
	Closure tmp15;
	Closure tmp17;

	mapping(var0, cons2, cons3, cons4, cons5, cons6, tmp7);
	node_tex_image(tmp7, samp0, tmp10, tmp11);
	srgb_to_linearrgb(tmp10, tmp13);
	node_bsdf_transparent(tmp13, tmp15);
	node_output_eevee_material(tmp15, tmp17);

	return tmp17;
}
#ifndef NODETREE_EXEC
out vec4 fragColor;
void main()
{
	Closure cl = nodetree_exec();
	fragColor = vec4(cl.radiance, cl.opacity);
}
#endif

```
>
- 这里没有定义 NODETREE_EXEC 的话，就会默认给一个main函数，上面的prepass_frag有main函数，所以就定义了NODETREE_EXEC
```
out vec4 fragColor;
void main()
{
	Closure cl = nodetree_exec();
	fragColor = vec4(cl.radiance, cl.opacity);
}
```