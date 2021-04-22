---
layout:     post
title:      "blender eevee Add support for new Hair geometry system"
subtitle:   ""
date:       2021-4-22 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

主要看这个commit

> GIT : 2018/5/29  *   Eevee : Add support for new Hair geometry system. <br> 

> This now can shade actual poly strips that mimics cylinders.
This makes hair coverage exact compared to the line method and result in
smoother fading hair.

> This does make the sampling a bit more exact but needs more samples to
converge properly.


> SVN : 2018/5/28  *  Win64_vc14/Windows_vc14 add patches for boost and osl to support building with clang. 


<br><br>

## 效果
![](/img/Eevee/Hair/02/1.png)


<br><br>

## 作用
优化 头发 效果

<br><br>


### 渲染入口
*eevee_engine.c*
```
static void eevee_draw_background(void *vedata)
{
	...
	/* Refresh Hair */
	DRW_draw_pass(psl->hair_tf_pass);
	...

	/* Depth prepass */
	DRW_stats_group_start("Prepass");
	DRW_draw_pass(psl->depth_pass);
	DRW_draw_pass(psl->depth_pass_cull);
	DRW_stats_group_end();
	...

	DRW_draw_pass(psl->material_pass);
}
```
>
-  Refresh Hair 是优先执行，No.1 执行

<br><br>

### 渲染前

#### 初始化
*eevee_materials.c*
```
void EEVEE_materials_init(EEVEE_ViewLayerData *sldata, EEVEE_StorageList *stl, EEVEE_FramebufferList *fbl)
{
	e_data.vert_shader_str = BLI_string_joinN(
			datatoc_common_view_lib_glsl,
			datatoc_common_hair_lib_glsl,
			datatoc_lit_surface_vert_glsl);

	...

	char *vert_str = BLI_string_joinN(
			datatoc_common_view_lib_glsl,
			datatoc_common_hair_lib_glsl,
			datatoc_prepass_vert_glsl);

	e_data.default_hair_prepass_sh = DRW_shader_create(
			vert_str, NULL, datatoc_prepass_frag_glsl,
			"#define HAIR_SHADER\n");

	e_data.default_hair_prepass_clip_sh = DRW_shader_create(
			vert_str, NULL, datatoc_prepass_frag_glsl,
			"#define HAIR_SHADER\n"
			"#define CLIP_PLANES\n");

	...
}

void EEVEE_materials_cache_populate(EEVEE_Data *vedata, EEVEE_ViewLayerData *sldata, Object *ob, bool *cast_shadow)
{
	...
	// 这里是检查模型有没有添加粒子系统，然后再检查下粒子系统是否使用hair
	if (ob->type == OB_MESH) {
		if (ob != draw_ctx->object_edit) {
			material_hash = stl->g_data->hair_material_hash;
			for (ModifierData *md = ob->modifiers.first; md; md = md->next) {
				if (md->type != eModifierType_ParticleSystem) {
					continue;
				}
				ParticleSystem *psys = ((ParticleSystemModifierData *)md)->psys;
				if (!psys_check_enabled(ob, psys, false)) {
					continue;
				}
				if (!DRW_check_psys_visible_within_active_context(ob, psys)) {
					continue;
				}
				ParticleSettings *part = psys->part;
				const int draw_as = (part->draw_as == PART_DRAW_REND) ? part->ren_as : part->draw_as;
				if (draw_as != PART_DRAW_PATH) {
					continue;
				}

				//<------------------------------ 这里开始

				DRWShadingGroup *shgrp = NULL;
				Material *ma = give_current_material(ob, part->omat);

				if (ma == NULL) {
					ma = &defmaterial;
				}

				float *color_p = &ma->r;
				float *metal_p = &ma->ray_mirror;
				float *spec_p = &ma->spec;
				float *rough_p = &ma->gloss_mir;

				//<------------------------------ 第一步

				shgrp = DRW_shgroup_hair_create(
				        ob, psys, md,
				        psl->depth_pass, psl->hair_tf_pass,
				        e_data.default_hair_prepass_sh);

				shgrp = DRW_shgroup_hair_create(
				        ob, psys, md,
				        psl->depth_pass_clip, psl->hair_tf_pass,
				        e_data.default_hair_prepass_clip_sh);
				DRW_shgroup_uniform_block(shgrp, "clip_block", sldata->clip_ubo);

				shgrp = NULL;
				if (ma->use_nodes && ma->nodetree) {
					static float half = 0.5f;
					static float error_col[3] = {1.0f, 0.0f, 1.0f};
					static float compile_col[3] = {0.5f, 0.5f, 0.5f};

					//<------------------------------ 第二步
					// 这里nodetree出来的材质是 定义了 VAR_MAT_HAIR 宏
					struct GPUMaterial *gpumat = EEVEE_material_hair_get(scene, ma, sldata->lamps->shadow_method);

					switch (GPU_material_status(gpumat)) {
						case GPU_MAT_SUCCESS:
						{
							//<------------------------------ 第三步
							shgrp = DRW_shgroup_material_hair_create(
							        ob, psys, md,
							        psl->material_pass, psl->hair_tf_pass,
							        gpumat);
							add_standard_uniforms(shgrp, sldata, vedata, NULL, NULL, false, false);
							break;
						}
						case GPU_MAT_QUEUED:
						{
							sldata->probes->all_materials_updated = false;
							color_p = compile_col;
							metal_p = spec_p = rough_p = &half;
							break;
						}
						case GPU_MAT_FAILED:
						default:
							color_p = error_col;
							metal_p = spec_p = rough_p = &half;
							break;
					}
				}

				/* Fallback to default shader */
				if (shgrp == NULL) {
					bool use_ssr = ((stl->effects->enabled_effects & EFFECT_SSR) != 0);
					shgrp = EEVEE_default_shading_group_get(sldata, vedata,
					                                        ob, psys, md,
					                                        true, false, use_ssr,
					                                        sldata->lamps->shadow_method);
					DRW_shgroup_uniform_vec3(shgrp, "basecol", color_p, 1);
					DRW_shgroup_uniform_float(shgrp, "metallic", metal_p, 1);
					DRW_shgroup_uniform_float(shgrp, "specular", spec_p, 1);
					DRW_shgroup_uniform_float(shgrp, "roughness", rough_p, 1);
				}
			}
		}
	}
	...
}
```
>
- 这一大坨代码都是为了给 depth_pass, depth_pass_clip, hair_tf_pass 和 material_pass 列表添加 drawcall

<br><br>

#### DRW_shgroup_hair_create 函数

```
DRWShadingGroup *DRW_shgroup_hair_create(
        Object *object, ParticleSystem *psys, ModifierData *md,
        DRWPass *hair_pass, DRWPass *tf_pass,
        GPUShader *shader)
{
	return drw_shgroup_create_hair_procedural_ex(object, psys, md, hair_pass, tf_pass, NULL, shader);
}

```
>
- DRW_shgroup_hair_create 调用 drw_shgroup_create_hair_procedural_ex

<br><br>

##### drw_shgroup_create_hair_procedural_ex  函数

*draw_hair.c*
```
static GPUShader *hair_refine_shader_get(ParticleRefineShader sh)
{
	if (g_refine_shaders[sh]) {
		return g_refine_shaders[sh];
	}

	char *vert_with_lib = BLI_string_joinN(datatoc_common_hair_lib_glsl, datatoc_common_hair_refine_vert_glsl);

	const char *var_names[1] = {"outData"};

	g_refine_shaders[sh] = DRW_shader_create_with_transform_feedback(vert_with_lib, NULL, "#define HAIR_PHASE_SUBDIV\n",
	                                                                 GPU_SHADER_TFB_POINTS, var_names, 1);

	MEM_freeN(vert_with_lib);

	return g_refine_shaders[sh];
}
```
>
- 这里使用到了 Opengl Transform Feedback，可以参考 [feedback](https://open.gl/feedback)
<br><br>
- 直接把vs里面的 outData 输出到 framebuffer中

<br>

```
static DRWShadingGroup *drw_shgroup_create_hair_procedural_ex(
        Object *object, ParticleSystem *psys, ModifierData *md,
        DRWPass *hair_pass, DRWPass *tf_pass,
        struct GPUMaterial *gpu_mat, GPUShader *gpu_shader)
{
	/* TODO(fclem): Pass the scene as parameter */
	const DRWContextState *draw_ctx = DRW_context_state_get();
	Scene *scene = draw_ctx->scene;

	int subdiv = scene->r.hair_subdiv;
	int thickness_res = (scene->r.hair_type == SCE_HAIR_SHAPE_STRAND) ? 1 : 2;

	ParticleHairCache *hair_cache;
	ParticleSettings *part = psys->part;
	bool need_ft_update = particles_ensure_procedural_data(object, psys, md, &hair_cache, subdiv, thickness_res);

	DRWShadingGroup *shgrp;
	if (gpu_mat) {
		shgrp = DRW_shgroup_material_create(gpu_mat, hair_pass);
	}
	else if (gpu_shader) {
		shgrp = DRW_shgroup_create(gpu_shader, hair_pass);
	}
	else {
		BLI_assert(0);
	}

	/* TODO optimize this. Only bind the ones GPUMaterial needs. */
	for (int i = 0; i < hair_cache->num_uv_layers; ++i) {
		for (int n = 0; hair_cache->uv_layer_names[i][n][0] != '\0'; ++n) {
			DRW_shgroup_uniform_texture(shgrp, hair_cache->uv_layer_names[i][n], hair_cache->uv_tex[i]);
		}
	}
	for (int i = 0; i < hair_cache->num_col_layers; ++i) {
		for (int n = 0; hair_cache->col_layer_names[i][n][0] != '\0'; ++n) {
			DRW_shgroup_uniform_texture(shgrp, hair_cache->col_layer_names[i][n], hair_cache->col_tex[i]);
		}
	}

	DRW_shgroup_uniform_texture(shgrp, "hairPointBuffer", hair_cache->final[subdiv].proc_tex);
	DRW_shgroup_uniform_int(shgrp, "hairStrandsRes", &hair_cache->final[subdiv].strands_res, 1);
	DRW_shgroup_uniform_int_copy(shgrp, "hairThicknessRes", thickness_res);
	DRW_shgroup_uniform_float(shgrp, "hairRadShape", &part->shape, 1);
	DRW_shgroup_uniform_float_copy(shgrp, "hairRadRoot", part->rad_root * part->rad_scale * 0.5f);
	DRW_shgroup_uniform_float_copy(shgrp, "hairRadTip", part->rad_tip * part->rad_scale * 0.5f);
	DRW_shgroup_uniform_bool_copy(shgrp, "hairCloseTip", (part->shape_flag & PART_SHAPE_CLOSE_TIP) != 0);
	/* TODO(fclem): Until we have a better way to cull the hair and render with orco, bypass culling test. */
	DRW_shgroup_call_object_add_no_cull(shgrp, hair_cache->final[subdiv].proc_hairs[thickness_res-1], object);

	/* Transform Feedback subdiv. */
	if (need_ft_update) {
		int final_points_ct = hair_cache->final[subdiv].strands_res * hair_cache->strands_count;
		GPUShader *tf_shader = hair_refine_shader_get(PART_REFINE_CATMULL_ROM);
		DRWShadingGroup *tf_shgrp = DRW_shgroup_transform_feedback_create(tf_shader, tf_pass,
		                                                                  hair_cache->final[subdiv].proc_buf);
		DRW_shgroup_uniform_texture(tf_shgrp, "hairPointBuffer", hair_cache->point_tex);
		DRW_shgroup_uniform_texture(tf_shgrp, "hairStrandBuffer", hair_cache->strand_tex);
		DRW_shgroup_uniform_int(tf_shgrp, "hairStrandsRes", &hair_cache->final[subdiv].strands_res, 1);
		DRW_shgroup_call_procedural_points_add(tf_shgrp, final_points_ct, NULL);
	}

	return shgrp;
}
```
>
- 通过阅读上面的代码可以知道
<br><br>
- hair_pass 使用 shader, 然后再设置一些Shader里面的uniform变量，这里的这个shader 是外部传进来的。
<br><br>
- tf_pass 使用 tf_shader, 这个 tf_shader 是由 common_hair_refine_vert.glsl + common_hair_lib.glsl,并且定义了 #define HAIR_PHASE_SUBDIV，主要这里没有 frag
<br><br>
- tf 的意思就是 Opengl Transform Feedback，可以参考 [feedback](https://open.gl/feedback)，这里会把 common_hair_refine_vert.glsl 中的 outData 输出到 hair_cache->final[subdiv].proc_buf 上，hair_cache->final[subdiv].proc_tex 和这个  hair_cache->final[subdiv].proc_buf 是绑定在一起的
<br><br>
- need_ft_update 什么时候为true，经过验证，只要修改了hair的面板的参数，need_ft_update 都会为true，但是 need_ft_update 为true之后，为 tf_pass 添加了一次 tf_shader 之后，就会变回 false，也就是说，修改参数一次，tf_pass 就会添加 tf_shader 执行一次
<br><br>
- 之后会出现 DRW_shgroup_material_hair_create ，其实道理是一样的，只不过是用nodetree，而不是shader

<br><br>

#### depth_pass + depth_pass_clip + hair_tf_pass 
```
shgrp = DRW_shgroup_hair_create(
		ob, psys, md,
		psl->depth_pass, psl->hair_tf_pass,
		e_data.default_hair_prepass_sh);

shgrp = DRW_shgroup_hair_create(
		ob, psys, md,
		psl->depth_pass_clip, psl->hair_tf_pass,
		e_data.default_hair_prepass_clip_sh);
```
>
- 经过阅读上面可以，可以知道
<br><br>
- psl->depth_pass 添加使用 drawcall，drawcall 使用shader 是 vs : common_hair_lib.glsl + prepass_vert.glsl,  ps : prepass_frag.glsl, 定义宏 HAIR_SHADER
<br><br>
- psl->depth_pass_clip 也一样，只不过 Shader 定义宏多了一个 CLIP_PLANES
<br><br>
- hair_tf_pass 也会添加 drawcall，只要hair的参数变化的，就回执行，往 hair_cache->final[subdiv].proc_buf 输出数据，hair_cache->final[subdiv].proc_tex 变化，这个 hair_tf_pass 使用 common_hair_refine_vert.glsl + common_hair_lib.glsl

<br><br>

#### material_pass

```
//<------------------------------ 第二步
// 这里nodetree出来的材质是 定义了 VAR_MAT_HAIR 宏
struct GPUMaterial *gpumat = EEVEE_material_hair_get(scene, ma, sldata->lamps->shadow_method);
...

//<------------------------------ 第三步
shgrp = DRW_shgroup_material_hair_create(
		ob, psys, md,
		psl->material_pass, psl->hair_tf_pass,
		gpumat);
add_standard_uniforms(shgrp, sldata, vedata, NULL, NULL, false, false);
break;
```
>
- material_pass 使用 nodetree 生成Shader代码，注意的是，这些代码会直接定义宏 VAR_MAT_HAIR

<br><br>

### Shader

#### hair_tf_pass

*common_hair_lib.glsl*
```
/**
 * Library to create hairs dynamically from control points.
 * This is less bandwidth intensive than fetching the vertex attributes
 * but does more ALU work per vertex. This also reduce the number
 * of data the CPU has to precompute and transfert for each update.
 **/

/**
 * hairStrandsRes: Number of points per hair strand.
 * 2 - no subdivision
 * 3+ - 1 or more interpolated points per hair.
 **/
uniform int hairStrandsRes = 8;

/**
 * hairThicknessRes : Subdiv around the hair.
 * 1 - Wire Hair: Only one pixel thick, independant of view distance.
 * 2 - Polystrip Hair: Correct width, flat if camera is parallel.
 * 3+ - Cylinder Hair: Massive calculation but potentially perfect. Still need proper support.
 **/
uniform int hairThicknessRes = 1;

/* Hair thickness shape. */
uniform float hairRadRoot = 0.01;
uniform float hairRadTip = 0.0;
uniform float hairRadShape = 0.5;
uniform bool hairCloseTip = true;

/* -- Per control points -- */
uniform samplerBuffer hairPointBuffer; /* RGBA32F */
#define point_position     xyz
#define point_time         w           /* Position along the hair length */

/* -- Per strands data -- */
uniform usamplerBuffer hairStrandBuffer; /* R32UI */

/* Not used, use one buffer per uv layer */
//uniform samplerBuffer hairUVBuffer; /* RG32F */
//uniform samplerBuffer hairColBuffer; /* RGBA16 linear color */

void unpack_strand_data(uint data, out int strand_offset, out int strand_segments)
{
#if 0 /* Pack point count */
	// strand_offset = (data & 0x1FFFFFFFu);
	// strand_segments = 1u << (data >> 29u); /* We only need 3 bits to store subdivision level. */
#else
	strand_offset = int(data & 0x00FFFFFFu);
	strand_segments = int(data >> 24u);
#endif
}

/* -- Subdivision stage -- */
/**
 * We use a transform feedback to preprocess the strands and add more subdivision to it.
 * For the moment theses are simple smooth interpolation but one could hope to see the full
 * children particle modifiers being evaluated at this stage.
 *
 * If no more subdivision is needed, we can skip this step.
 **/

#ifdef HAIR_PHASE_SUBDIV
int hair_get_base_id(float local_time, int strand_segments, out float interp_time)
{
	float time_per_strand_seg = 1.0 / float(strand_segments);

	float ratio = local_time / time_per_strand_seg;
	interp_time = fract(ratio);

	return int(ratio);
}

void hair_get_interp_attribs(out vec4 data0, out vec4 data1, out vec4 data2, out vec4 data3, out float interp_time)
{
	float local_time = float(gl_VertexID % hairStrandsRes) / float(hairStrandsRes - 1);

	int hair_id = gl_VertexID / hairStrandsRes;
	uint strand_data = texelFetch(hairStrandBuffer, hair_id).x;

	int strand_offset, strand_segments;
	unpack_strand_data(strand_data, strand_offset, strand_segments);

	int id = hair_get_base_id(local_time, strand_segments, interp_time);

	int ofs_id = id + strand_offset;

	data0 = texelFetch(hairPointBuffer, ofs_id - 1);
	data1 = texelFetch(hairPointBuffer, ofs_id);
	data2 = texelFetch(hairPointBuffer, ofs_id + 1);
	data3 = texelFetch(hairPointBuffer, ofs_id + 2);

	if (id <= 0) {
		/* root points. Need to reconstruct previous data. */
		data0 = data1 * 2.0 - data2;
	}
	if (id + 1 >= strand_segments) {
		/* tip points. Need to reconstruct next data. */
		data3 = data2 * 2.0 - data1;
	}
}
#endif

/* -- Drawing stage -- */
/**
 * For final drawing, the vertex index and the number of vertex per segment
 **/

#ifndef HAIR_PHASE_SUBDIV
int hair_get_strand_id(void)
{
	return gl_VertexID / (hairStrandsRes * hairThicknessRes);
}

int hair_get_base_id(void)
{
	return gl_VertexID / hairThicknessRes;
}

/* Copied from cycles. */
float hair_shaperadius(float shape, float root, float tip, float time)
{
	float radius = 1.0 - time;

	if (shape < 0.0) {
		radius = pow(radius, 1.0 + shape);
	}
	else {
		radius = pow(radius, 1.0 / (1.0 - shape));
	}

	if (hairCloseTip && (time > 0.99)) {
		return 0.0;
	}

	return (radius * (root - tip)) + tip;
}

void hair_get_pos_tan_nor_time(
        bool is_persp, vec3 camera_pos, vec3 camera_z,
        out vec3 wpos, out vec3 wtan, out vec3 wnor, out float time, out float thickness, out float thick_time)
{
	int id = hair_get_base_id();
	vec4 data = texelFetch(hairPointBuffer, id);
	wpos = data.point_position;
	time = data.point_time;
	if (time == 0.0) {
		/* Hair root */
		wtan = texelFetch(hairPointBuffer, id + 1).point_position - wpos;
	}
	else {
		wtan = wpos - texelFetch(hairPointBuffer, id - 1).point_position;
	}

	vec3 camera_vec = (is_persp) ? wpos - camera_pos : -camera_z;
	wnor = normalize(cross(camera_vec, wtan));

	thickness = hair_shaperadius(hairRadShape, hairRadRoot, hairRadTip, time);

	if (hairThicknessRes > 1) {
		thick_time = float(gl_VertexID % hairThicknessRes) / float(hairThicknessRes - 1);
		thick_time = thickness * (thick_time * 2.0 - 1.0);

		wpos += wnor * thick_time;
	}
}

vec2 hair_get_customdata_vec2(const samplerBuffer cd_buf)
{
	int id = hair_get_strand_id();
	return texelFetch(cd_buf, id).rg;
}

vec3 hair_get_customdata_vec3(const samplerBuffer cd_buf)
{
	int id = hair_get_strand_id();
	return texelFetch(cd_buf, id).rgb;
}

vec4 hair_get_customdata_vec4(const samplerBuffer cd_buf)
{
	int id = hair_get_strand_id();
	return texelFetch(cd_buf, id).rgba;
}

vec3 hair_get_strand_pos(void)
{
	int id = hair_get_strand_id() * hairStrandsRes;
	return texelFetch(hairPointBuffer, id).point_position;
}

#endif

```

<br>


*common_hair_refine_vert.glsl*
```
/* To be compiled with common_hair_lib.glsl */

out vec4 outData;

void main(void)
{
	float interp_time;
	vec4 data0, data1, data2, data3;
	hair_get_interp_attribs(data0, data1, data2, data3, interp_time);

	/* TODO some interpolation. */

	outData = mix(data1, data2, interp_time);
}
```
>
- 这里是根据参数直接输出 outData 到 framebuffer 上

<br><br>


#### depth_pass
*prepass_vert.glsl*
```
uniform mat4 ModelViewProjectionMatrix;
uniform mat4 ModelMatrix;

/* keep in sync with DRWManager.view_data */
layout(std140) uniform clip_block {
	vec4 ClipPlanes[1];
};

#ifndef HAIR_SHADER
in vec3 pos;
#endif

void main()
{
#ifdef HAIR_SHADER
	float time, thick_time, thickness;
	vec3 pos, nor, binor;
	hair_get_pos_tan_nor_time(
	        (ProjectionMatrix[3][3] == 0.0),
	        ViewMatrixInverse[3].xyz, ViewMatrixInverse[2].xyz,
	        pos, nor, binor, time, thickness, thick_time);

	gl_Position = ViewProjectionMatrix * vec4(pos, 1.0);
	vec4 worldPosition = vec4(pos, 1.0);
#else
	gl_Position = ModelViewProjectionMatrix * vec4(pos, 1.0);
	vec4 worldPosition = (ModelMatrix * vec4(pos, 1.0));
#endif

#ifdef CLIP_PLANES
	gl_ClipDistance[0] = dot(vec4(worldPosition.xyz, 1.0), ClipPlanes[0]);
#endif
	/* TODO motion vectors */
}
```
>
- hair_get_pos_tan_nor_time 会采样 hairPointBuffer


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

uniform float hashAlphaOffset;

float hashed_alpha_threshold(vec3 co)
{
	const float hash_scale = 1.0; /* Roughly in pixel */

	/* Find the discretized derivatives of our coordinates. */
	float max_deriv = max(length(dFdx(co)), length(dFdy(co)));
	float pix_scale = 1.0 / (hash_scale * max_deriv);

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

	/* Jitter the threshold for TAA accumulation. */
	return fract(threshold + hashAlphaOffset);
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

<br><br>

#### material_pass
*lit_surface_vert.glsl*
```
uniform mat4 ModelViewProjectionMatrix;
uniform mat4 ModelMatrix;
uniform mat4 ModelViewMatrix;
uniform mat3 WorldNormalMatrix;
#ifndef ATTRIB
uniform mat3 NormalMatrix;
uniform mat4 ModelMatrixInverse;
#endif

#ifndef HAIR_SHADER
in vec3 pos;
in vec3 nor;
#endif

out vec3 worldPosition;
out vec3 viewPosition;

/* Used for planar reflections */
/* keep in sync with EEVEE_ClipPlanesUniformBuffer */
layout(std140) uniform clip_block {
	vec4 ClipPlanes[1];
};

#ifdef USE_FLAT_NORMAL
flat out vec3 worldNormal;
flat out vec3 viewNormal;
#else
out vec3 worldNormal;
out vec3 viewNormal;
#endif

#ifdef HAIR_SHADER
out vec3 hairTangent;
out float hairThickTime;
out float hairThickness;
out float hairTime;
#endif

void main()
{
#ifdef HAIR_SHADER
	vec3 pos, nor;
	hair_get_pos_tan_nor_time(
	        (ProjectionMatrix[3][3] == 0.0),
	        ViewMatrixInverse[3].xyz, ViewMatrixInverse[2].xyz,
	        pos, nor, hairTangent, hairTime, hairThickness, hairThickTime);

	gl_Position = ViewProjectionMatrix * vec4(pos, 1.0);
	viewPosition = (ModelMatrixInverse * vec4(pos, 1.0)).xyz;
	worldPosition = pos;
	worldNormal = nor;
	viewNormal = normalize(mat3(ViewMatrixInverse) * nor);
#else
	gl_Position = ModelViewProjectionMatrix * vec4(pos, 1.0);
	viewPosition = (ModelViewMatrix * vec4(pos, 1.0)).xyz;
	worldPosition = (ModelMatrix * vec4(pos, 1.0)).xyz;
	worldNormal = normalize(WorldNormalMatrix * nor);
	viewNormal = normalize(NormalMatrix * nor);
#endif

	/* Used for planar reflections */
	gl_ClipDistance[0] = dot(vec4(worldPosition, 1.0), ClipPlanes[0]);

#ifdef ATTRIB
	pass_attrib(pos);
#endif
}
```

<br>
<br>

*lit_surface_frag.glsl*
```
#ifdef HAIR_SHADER
in vec3 hairTangent; /* world space */
in float hairThickTime;
in float hairThickness;
in float hairTime;

uniform int hairThicknessRes = 1;
#endif

...

#ifdef HAIR_SHADER
	if (hairThicknessRes == 1) {
		/* Random normal distribution on the hair surface. */
		vec3 T = normalize(worldNormal); /* meh, TODO fix worldNormal misnaming. */
		vec3 B = normalize(cross(V, T));
		N = cross(T, B); /* Normal facing view */
		/* We want a cosine distribution. */
		float cos_theta = rand.x * 2.0 - 1.0;
		float sin_theta = sqrt(max(0.0, 1.0f - cos_theta*cos_theta));;
		N = N * sin_theta + B * cos_theta;

#  ifdef CLOSURE_GLOSSY
		/* Hair random normal does not work with SSR :(.
		 * It just create self reflection feedback (which is beautifful btw)
		 * but not correct. */
		ssr_id = NO_SSR; /* Force bypass */
#  endif
	}
	else {
		vec3 T = normalize(cross(hairTangent, worldNormal));
		/* We want a cosine distribution. */
		float cos_theta = hairThickTime / hairThickness;
		float sin_theta = sqrt(max(0.0, 1.0f - cos_theta*cos_theta));;
		N = normalize(hairTangent * cos_theta + T * sin_theta);
	}
#endif

```

### 总结
- hair_tf_pass 先执行，主要是利用hair参数 + Transform Feedback 技术，输出Hair的Vertex Animation 数据到 RT 上
<br>
- depth_pass 后执行，利用第一个输出的RT数据，在 VS 中进行 变化效果
<br>
- material_pass 在VS 利用利用第一个输出的RT数据，进行Vertex Animation，然后在 PS 中进行渲染
