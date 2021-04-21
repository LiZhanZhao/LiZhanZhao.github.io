---
layout:     post
title:      "blender eevee LookDev"
subtitle:   ""
date:       2021-4-21 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

主要看这个commit

> GIT : 2018/5/25  *   Eevee : LookDev. <br> 

> SVN : 2018/5/28  *  Win64_vc14/Windows_vc14 add patches for boost and osl to support building with clang. 


<br><br>

## 效果

![](/img/Eevee/LookDev/01/1.png)
![](/img/Eevee/LookDev/01/2.png)

## 作用
分析 LookDev 模式下是如何渲染, 其实主要是多了两个球的渲染，加上背景的渲染，还有环境图的背景的渲染

<br><br>

### EEVEE_materials_cache_xxx 调用
>
- EEVEE_materials_cache_init 无论多少个物体，每一帧都只会执行一次
<br><br>
- EEVEE_materials_cache_populate 多少个物体，每一帧就执行多少次，类似于遍历物体
<br><br>
- EEVEE_materials_cache_finish 无论多少个物体，每一帧都只会执行一次
<br><br>
- 先执行 一次 EEVEE_materials_cache_init，再执行多次 EEVEE_materials_cache_populate, 再执行一次 EEVEE_materials_cache_finish

<br><br>

### LookDev 渲染

#### 渲染入口
*eevee_engine.c*
```
static void eevee_draw_background(void *vedata)
{
	...
	/* Post Process */
	...
	...
	/* LookDev */
	EEVEE_lookdev_draw_background(vedata);
	/* END */
	...
	...
	/* Tonemapping and transfer result to default framebuffer. */
	GPU_framebuffer_bind(dfbl->default_fb);
	DRW_transform_to_display(stl->effects->final_tx);
}
```
>
- 主要留意 EEVEE_lookdev_draw_background 函数
<br><br>
- 这个函数 在Post Process 之后， 在Tonemapping 之前 执行

<br><br>

#### EEVEE_lookdev_draw_background
*eevee_lookdev.c*
```
void EEVEE_lookdev_draw_background(EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_StorageList *stl = ((EEVEE_Data *)vedata)->stl;
	EEVEE_EffectsInfo *effects = stl->effects;
	EEVEE_ViewLayerData *sldata = EEVEE_view_layer_data_ensure();
	
	const DRWContextState *draw_ctx = DRW_context_state_get();

	if (psl->lookdev_pass && draw_ctx->v3d) {
		DRW_stats_group_start("Look Dev");
		CameraParams params;
		BKE_camera_params_init(&params);
		View3D *v3d = draw_ctx->v3d;
		RegionView3D *rv3d = draw_ctx->rv3d;
		ARegion *ar = draw_ctx->ar;

		BKE_camera_params_from_view3d(&params, draw_ctx->depsgraph, v3d, rv3d);
		params.is_ortho = true;
		params.ortho_scale = 4.0;
		params.zoom = CAMERA_PARAM_ZOOM_INIT_PERSP;
		BKE_camera_params_compute_viewplane(&params, ar->winx, ar->winy, 1.0f, 1.0f);

		// 这里是为了计算正交投影矩阵(winmat)
		BKE_camera_params_compute_matrix(&params);

		const float *viewport_size = DRW_viewport_size_get();
		int viewport_inset_x = viewport_size[0]/4;
		int viewport_inset_y = viewport_size[1]/4;

		EEVEE_CommonUniformBuffer *common = &sldata->common_data;
		common->la_num_light = 0;
		common->prb_num_planar = 0;
		common->prb_num_render_cube = 1;
		common->prb_num_render_grid = 1;
		common->ao_dist = 0.0f;
		common->ao_factor = 0.0f;
		common->ao_settings = 0.0f;
		DRW_uniformbuffer_update(sldata->common_ubo, common);

		/* override matrices */
		float winmat[4][4];
		float winmat_inv[4][4];

		// 因为上面已经计算了 params.winmat 是正交投影矩阵，这里就利用这个正交矩阵设置为 matrix_override DRW_MAT_WIN + DRW_MAT_WININV
		copy_m4_m4(winmat, params.winmat);
		invert_m4_m4(winmat_inv, winmat);
		DRW_viewport_matrix_override_set(winmat, DRW_MAT_WIN);
		DRW_viewport_matrix_override_set(winmat_inv, DRW_MAT_WININV);

		// 因为 params.winmat 是正交投影矩阵，这里就修改 VP 矩阵，设置 matrix_override DRW_MAT_PERS + DRW_MAT_PERSINV
		float viewmat[4][4];
		DRW_viewport_matrix_get(viewmat, DRW_MAT_VIEW);
		float persmat[4][4];
		float persmat_inv[4][4];
		mul_m4_m4m4(persmat, winmat, viewmat);
		invert_m4_m4(persmat_inv, persmat);
		DRW_viewport_matrix_override_set(persmat, DRW_MAT_PERS);
		DRW_viewport_matrix_override_set(persmat_inv, DRW_MAT_PERSINV);

		GPUFrameBuffer *fb = effects->final_fb;
		GPU_framebuffer_bind(fb);
		GPU_framebuffer_viewport_set(fb, viewport_size[0]-viewport_inset_x, 0, viewport_inset_x, viewport_inset_y);
		DRW_draw_pass(psl->lookdev_pass);

		DRW_viewport_matrix_override_unset_all();
		DRW_stats_group_end();
	}
}
```
>
- 这里主要是 往 effects->final_fb framebuffer 上，进行渲染，渲染使用 lookdev_pass Pass
<br><br>
- 注意的是，渲染前会进行 override matrices
<br><br>
- 矩阵怎么计算? 上面的代码注释可以看到，思路就是计算 正交投影矩阵，然后再 DRW_viewport_matrix_override_set,修改 DRW_MAT_WIN + DRW_MAT_WININV + DRW_MAT_PERS + DRW_MAT_PERSINV，也就是 P 矩阵，VP 矩阵
<br><br>

<br><br>

### lookdev_pass 渲染两个球

#### 初始化
*eevee_materials.c*
```

static void create_default_shader(int options)
{
	char *frag_str = BLI_string_joinN(
	        e_data.frag_shader_lib,
	        datatoc_default_frag_glsl);

	char *defines = eevee_get_defines(options);

	e_data.default_lit[options] = DRW_shader_create(datatoc_lit_surface_vert_glsl, NULL, frag_str, defines);

	MEM_freeN(defines);
	MEM_freeN(frag_str);
}


/**
 * Create a default shading group inside the lookdev pass without standard uniforms.
 **/
static struct DRWShadingGroup *EEVEE_lookdev_shading_group_get(
        EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata,
        bool use_ssr, int shadow_method)
{
	static int ssr_id;
	ssr_id = (use_ssr) ? 1 : -1;
	int options = VAR_MAT_MESH | VAR_MAT_LOOKDEV;

	options |= eevee_material_shadow_option(shadow_method);

	if (e_data.default_lit[options] == NULL) {
		create_default_shader(options);
	}

	if (vedata->psl->lookdev_pass == NULL) {
		DRWState state = DRW_STATE_WRITE_COLOR | DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_ALWAYS | DRW_STATE_CULL_BACK;
		vedata->psl->lookdev_pass = DRW_pass_create("LookDev Pass", state);

		DRWShadingGroup *shgrp = DRW_shgroup_create(e_data.default_lit[options], vedata->psl->lookdev_pass);
		/* XXX / WATCH: This creates non persistent binds for the ubos and textures.
		 * But it's currently OK because the following shgroups does not add any bind. */
		add_standard_uniforms(shgrp, sldata, vedata, &ssr_id, NULL, false, false);
	}

	return DRW_shgroup_create(e_data.default_lit[options], vedata->psl->lookdev_pass);
}

void EEVEE_materials_cache_finish(EEVEE_Data *vedata)
{
	EEVEE_StorageList *stl = ((EEVEE_Data *)vedata)->stl;

	/* Look-Dev */
	const DRWContextState *draw_ctx = DRW_context_state_get();
	const View3D *v3d = draw_ctx->v3d;
	if (v3d && v3d->drawtype == OB_MATERIAL) {
		EEVEE_ViewLayerData *sldata = EEVEE_view_layer_data_ensure();
		EEVEE_LampsInfo *linfo = sldata->lamps;
		struct Gwn_Batch *sphere = DRW_cache_sphere_get();
		static float mat1[4][4];
		static float color[3] = {1.0, 1.0, 1.0};
		static float metallic_on = 1.0f;
		static float metallic_off = 0.00f;
		static float specular = 1.0f;
		static float roughness = 0.05f;

		// 获得View的逆矩阵
		float view_mat[4][4];
		DRW_viewport_matrix_get(view_mat, DRW_MAT_VIEWINV);

		DRWShadingGroup *shgrp = EEVEE_lookdev_shading_group_get(sldata, vedata, false, linfo->shadow_method);
		DRW_shgroup_uniform_vec3(shgrp, "basecol", color, 1);
		DRW_shgroup_uniform_float(shgrp, "metallic", &metallic_on, 1);
		DRW_shgroup_uniform_float(shgrp, "specular", &specular, 1);
		DRW_shgroup_uniform_float(shgrp, "roughness", &roughness, 1);

		// 单位化
		unit_m4(mat1);
		// 乘上View的逆矩阵，其实这个就是相机的世界空间的矩阵
		mul_m4_m4m4(mat1, mat1, view_mat);
		// 在相机的世界空间上再进行偏移
		translate_m4(mat1, -1.5f, 0.0f, -5.0f);

		// mat1 当作 球的 世界矩阵
		DRW_shgroup_call_add(shgrp, sphere, mat1);

		shgrp = EEVEE_lookdev_shading_group_get(sldata, vedata, false, linfo->shadow_method);
		DRW_shgroup_uniform_vec3(shgrp, "basecol", color, 1);
		DRW_shgroup_uniform_float(shgrp, "metallic", &metallic_off, 1);
		DRW_shgroup_uniform_float(shgrp, "specular", &specular, 1);
		DRW_shgroup_uniform_float(shgrp, "roughness", &roughness, 1);
		translate_m4(mat1, 3.0f, 0.0f, 0.0f);
		DRW_shgroup_call_add(shgrp, sphere, mat1);
	}
	/* END */

	BLI_ghash_free(stl->g_data->material_hash, NULL, MEM_freeN);
	BLI_ghash_free(stl->g_data->hair_material_hash, NULL, NULL);
}
```
>
- 通过阅读上面的代码可以了解到，lookdev_pass 使用了Shader lit_surface_vert.glsl + default_frag.glsl, 而且定义了 宏 VAR_MAT_LOOKDEV
<br><br>
- 上面代码的意思就是，把两个 sphere 的渲染 添加到 lookdev_pass 中
<br><br>
- 注释可以看到，sphere 会进行设置世界矩阵，基于camear的世界矩阵来进行偏移

<br><br>

#### Shader
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

*default_frag.glsl*
```
uniform vec3 basecol;
uniform float metallic;
uniform float specular;
uniform float roughness;

Closure nodetree_exec(void)
{
	vec3 dielectric = vec3(0.034) * specular * 2.0;
	vec3 albedo = mix(basecol, vec3(0.0), metallic);
	vec3 f0 = mix(dielectric, basecol, metallic);
	vec3 N = (gl_FrontFacing) ? worldNormal : -worldNormal;
	vec3 out_diff, out_spec, ssr_spec;
	eevee_closure_default(N, albedo, f0, 1, roughness, 1.0, out_diff, out_spec, ssr_spec);

	Closure result = Closure(out_spec + out_diff * albedo, 1.0, vec4(ssr_spec, roughness), normal_encode(normalize(viewNormal), viewCameraVec), 0);

#ifdef LOOKDEV
	gl_FragDepth = 0.0;
#endif

	return result;
}
```
>
- eevee_closure_default 定义在 *lit_surface_frag.glsl* 文件里面

<br><br>


### 渲染背景

#### background_pass

*eevee_materials.c*
```
void EEVEE_materials_init(EEVEE_ViewLayerData *sldata, EEVEE_StorageList *stl, EEVEE_FramebufferList *fbl)
{
	...
	e_data.default_studiolight_background = DRW_shader_create(
			datatoc_background_vert_glsl, NULL, datatoc_default_world_frag_glsl,
			"#define LOOKDEV\n");
	...
}

void EEVEE_materials_cache_init(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	...
	/* LookDev */
	EEVEE_lookdev_cache_init(vedata, &grp, e_data.default_studiolight_background, psl->background_pass, NULL);
	/* END */
	...
}
```

<br><br>

*eevee_lookdev.c*
```
void EEVEE_lookdev_cache_init(EEVEE_Data *vedata, DRWShadingGroup **grp, GPUShader *shader, DRWPass *pass, EEVEE_LightProbesInfo *pinfo)
{
	EEVEE_StorageList *stl = vedata->stl;
	const DRWContextState *draw_ctx = DRW_context_state_get();
	View3D *v3d = draw_ctx->v3d;
	if (v3d && v3d->drawtype == OB_MATERIAL)
	{
		StudioLight *sl = BKE_studiolight_find(v3d->shading.studio_light, STUDIOLIGHT_ORIENTATION_WORLD);
		if ((sl->flag & STUDIOLIGHT_ORIENTATION_WORLD)) {
			struct Gwn_Batch *geom = DRW_cache_fullscreen_quad_get();

			BKE_studiolight_ensure_flag(sl, STUDIOLIGHT_EQUIRECTANGULAR_GPUTEXTURE);
			*grp = DRW_shgroup_create(shader, pass);
			GPUTexture *tex = sl->equirectangular_gputexture;
			DRW_shgroup_uniform_texture(*grp, "image", tex);

			axis_angle_to_mat3_single(stl->studiolight_matrix, 'Z', v3d->shading.studiolight_rot_z);
			DRW_shgroup_uniform_mat3(*grp, "StudioLightMatrix", stl->studiolight_matrix);

			DRW_shgroup_uniform_float(*grp, "backgroundAlpha", &stl->g_data->background_alpha, 1);
			DRW_shgroup_call_add(*grp, geom, NULL);

			/* Do we need to recalc the lightprobes? */
			if (pinfo && (pinfo->studiolight_index != sl->index || pinfo->studiolight_rot_z != v3d->shading.studiolight_rot_z)) {
				pinfo->update_world |= PROBE_UPDATE_ALL;
				pinfo->studiolight_index = sl->index;
				pinfo->studiolight_rot_z = v3d->shading.studiolight_rot_z;
				pinfo->prev_wo_sh_compiled = false;
				pinfo->prev_world = NULL;
			}
		}
	}
}
```
>
- 看代码可以发现，v3d->drawtype == OB_MATERIAL 这个只会在 lookdev的模式下成立，然后就可以看到 background_pass 用Shader background_vert.glsl + default_world_frag.glsl 组成，并且定义了宏 LOOKDEV
<br><br>
- 也就是说，在渲染前，会看看模式是否是 lookdev 来定义 background_pass

<br><br>

#### Shader

*background_vert.glsl*
```
in vec2 pos;

out vec3 varposition;
out vec3 varnormal;
out vec3 viewPosition;

#ifndef VOLUMETRICS
/* necessary for compilation*/
out vec3 worldPosition;
out vec3 worldNormal;
out vec3 viewNormal;
#endif

void main()
{
	gl_Position = vec4(pos, 1.0, 1.0);
	varposition = viewPosition = vec3(pos, -1.0);
	varnormal = normalize(-varposition);
}

```

<br><br>

*default_world_frag.glsl*
```
#define M_PI 3.14159265358979323846

uniform float backgroundAlpha;
uniform mat4 ProjectionMatrix;
uniform mat4 ProjectionMatrixInverse;
uniform mat4 ViewMatrixInverse;
#ifdef LOOKDEV
uniform mat3 StudioLightMatrix;
uniform sampler2D image;
in vec3 viewPosition;
#else
uniform vec3 color;
#endif

out vec4 FragColor;

void background_transform_to_world(vec3 viewvec, out vec3 worldvec)
{
	vec4 v = (ProjectionMatrix[3][3] == 0.0) ? vec4(viewvec, 1.0) : vec4(0.0, 0.0, 1.0, 1.0);
	vec4 co_homogenous = (ProjectionMatrixInverse * v);

	vec4 co = vec4(co_homogenous.xyz / co_homogenous.w, 0.0);
	worldvec = (ViewMatrixInverse * co).xyz;
}

float hypot(float x, float y)
{
	return sqrt(x * x + y * y);
}

void node_tex_environment_equirectangular(vec3 co, sampler2D ima, out vec4 color)
{
	vec3 nco = normalize(co);
	float u = -atan(nco.y, nco.x) / (2.0 * M_PI) + 0.5;
	float v = atan(nco.z, hypot(nco.x, nco.y)) / M_PI + 0.5;

	/* Fix pole bleeding */
	float half_width = 0.5 / float(textureSize(ima, 0).x);
	v = clamp(v, half_width, 1.0 - half_width);

	/* Fix u = 0 seam */
	/* This is caused by texture filtering, since uv don't have smooth derivatives
	 * at u = 0 or 2PI, hardware filtering is using the smallest mipmap for certain
	 * texels. So we force the highest mipmap and don't do anisotropic filtering. */
	color = textureLod(ima, vec2(u, v), 0.0);
}

void main() {
#ifdef LOOKDEV
	vec3 worldvec;
	vec4 color;
	background_transform_to_world(viewPosition, worldvec);
	node_tex_environment_equirectangular(StudioLightMatrix * worldvec, image, color);
#endif

	FragColor = vec4(clamp(color.rgb, vec3(0.0), vec3(1e10)), backgroundAlpha);
}

```

<br><br>

### 渲染环境图的背景

*eevee_lightprobes.c*

```
static void lightprobe_shaders_init(void)
{
	...
	e_data.probe_default_studiolight_sh = DRW_shader_create(
	        datatoc_background_vert_glsl, NULL, datatoc_default_world_frag_glsl, "#define LOOKDEV\n");
	...
}


void EEVEE_lightprobes_cache_init(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	...
	/* LookDev */
	EEVEE_lookdev_cache_init(vedata, &grp, e_data.probe_default_studiolight_sh, psl->probe_background, pinfo);
	/* END */
	...
}
```
>
- 渲染环境球的背景的 probe_background 会也被修改，在 LookDev 的模式下，使用background_vert.glsl + default_world_frag.glsl,也定义了宏 LOOKDEV 
<br><br>
- 这样的话，其实就相当于修改了环境图
