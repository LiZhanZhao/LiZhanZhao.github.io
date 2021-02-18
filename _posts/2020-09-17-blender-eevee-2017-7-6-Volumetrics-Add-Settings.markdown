---
layout:     post
title:      "blender eevee Volumetrics Add settings."
subtitle:   ""
date:       2021-2-18 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/6  * Eevee : Volumetrics: Add settings .<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/VolumetricsSetting/1.png)
![](/img/Eevee/VolumetricsSetting/2.png)

## 作用
Volumetrics 添加 UI, 比较完整的Volumetrics效果

## 编译
- 重新生成SLN
- git 定位到  2017/7/6  * Eevee : Volumetrics: Add settings .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## 添加UI
*eevee_engine.c*
```
static void EEVEE_scene_layer_settings_create(RenderEngine *UNUSED(engine), IDProperty *props)
{
	...
	BKE_collection_engine_property_add_bool(props, "volumetric_enable", false);
	BKE_collection_engine_property_add_float(props, "volumetric_start", 0.1);
	BKE_collection_engine_property_add_float(props, "volumetric_end", 100.0);
	BKE_collection_engine_property_add_int(props, "volumetric_samples", 64);
	BKE_collection_engine_property_add_float(props, "volumetric_sample_distribution", 0.8);
	BKE_collection_engine_property_add_bool(props, "volumetric_lights", true);
	BKE_collection_engine_property_add_bool(props, "volumetric_shadows", false);
	BKE_collection_engine_property_add_int(props, "volumetric_shadow_samples", 16);
	BKE_collection_engine_property_add_bool(props, "volumetric_colored_transmittance", true);
	...
}
```
>
- 这里为 volumetric 添加UI


## 渲染前

*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	if (BKE_collection_engine_property_value_get_bool(props, "volumetric_enable")) {
		World *wo = scene->world;

		/* TODO: this will not be the case if we support object volumetrics */
		if ((wo != NULL) && (wo->use_nodes) && (wo->nodetree != NULL)) {
			effects->enabled_effects |= EFFECT_VOLUMETRIC;

			if (sldata->volumetrics == NULL) {
				sldata->volumetrics = MEM_callocN(sizeof(EEVEE_VolumetricsInfo), "EEVEE_VolumetricsInfo");
			}

			EEVEE_VolumetricsInfo *volumetrics = sldata->volumetrics;
			bool last_use_colored_transmit = volumetrics->use_colored_transmit; /* Save to compare */

			volumetrics->integration_start = BKE_collection_engine_property_value_get_float(props, "volumetric_start");
			volumetrics->integration_end = BKE_collection_engine_property_value_get_float(props, "volumetric_end");

			if (DRW_viewport_is_persp_get()) {
				/* Negate */
				volumetrics->integration_start = -volumetrics->integration_start;
				volumetrics->integration_end = -volumetrics->integration_end;
			}
			else {
				const float clip_start = stl->g_data->viewvecs[0][2];
				const float clip_end = stl->g_data->viewvecs[1][2];
				volumetrics->integration_start = min_ff(volumetrics->integration_end, clip_start);
				volumetrics->integration_end = max_ff(-volumetrics->integration_end, clip_end);
			}

			volumetrics->sample_distribution = BKE_collection_engine_property_value_get_float(props, "volumetric_sample_distribution");
			volumetrics->integration_step_count = (float)BKE_collection_engine_property_value_get_int(props, "volumetric_samples");
			volumetrics->shadow_step_count = (float)BKE_collection_engine_property_value_get_int(props, "volumetric_shadow_samples");

			volumetrics->use_lights = BKE_collection_engine_property_value_get_bool(props, "volumetric_lights");
			volumetrics->use_volume_shadows = BKE_collection_engine_property_value_get_bool(props, "volumetric_shadows");
			volumetrics->use_colored_transmit = BKE_collection_engine_property_value_get_bool(props, "volumetric_colored_transmittance");

			if (last_use_colored_transmit != volumetrics->use_colored_transmit) {
				if (fbl->volumetric_fb != NULL) {
					DRW_framebuffer_free(fbl->volumetric_fb);
					fbl->volumetric_fb = NULL;
				}
			}

			/* Integration result buffer(s) */
			if (volumetrics->use_colored_transmit == false) {
				/* Monocromatic transmittance in alpha */
				DRWFboTexture tex_vol = {&stl->g_data->volumetric, DRW_TEX_RGBA_16, DRW_TEX_MIPMAP | DRW_TEX_FILTER | DRW_TEX_TEMP};

				DRW_framebuffer_init(&fbl->volumetric_fb, &draw_engine_eevee_type,
				                    (int)viewport_size[0] / 2, (int)viewport_size[1] / 2,
				                    &tex_vol, 1);
			}
			else {
				/* Transmittance is separated, No need for alpha and DRW_TEX_RGB_11_11_10 gives the same vram usage */
				/* Hint ! Could reuse this for transparency! */
				DRWFboTexture tex_vol[2] = {&stl->g_data->volumetric, DRW_TEX_RGB_11_11_10, DRW_TEX_MIPMAP | DRW_TEX_FILTER | DRW_TEX_TEMP},
				                            {&stl->g_data->volumetric_transmit, DRW_TEX_RGB_11_11_10, DRW_TEX_MIPMAP | DRW_TEX_FILTER | DRW_TEX_TEMP};

				DRW_framebuffer_init(&fbl->volumetric_fb, &draw_engine_eevee_type,
				                    (int)viewport_size[0] / 2, (int)viewport_size[1] / 2,
				                    tex_vol, 2);
			}
		}
	}
	...
}
```
>
- 这里主要是计算Shader需要用到的参数
- integration_start， integration_end，sample_distribution，integration_step_count，shadow_step_count 根据UI进行输入
- use_lights 是否使用体积光，UI进行勾选
- use_volume_shadows 是否使用体积阴影，UI进行勾选
- use_colored_transmit 是否分开scatter 和 transmit 到 单独的RT上，这个也是UI进行勾选，可以参考[这里](http://shaderstore.cn/2021/02/18/blender-eevee-2017-7-4-Volumetrics-Colored-Transmittance-Support/)

<br><br>



*eevee_materials.c*
```
static char *eevee_get_volume_defines(int options)
{
	char *str = NULL;

	BLI_assert(options < VAR_MAT_MAX);

	DynStr *ds = BLI_dynstr_new();
	BLI_dynstr_appendf(ds, SHADER_DEFINES);
	BLI_dynstr_appendf(ds, "#define VOLUMETRICS\n");

	if ((options & VAR_VOLUME_SHADOW) != 0) {
		BLI_dynstr_appendf(ds, "#define VOLUME_SHADOW\n");
	}
	if ((options & VAR_VOLUME_HOMO) != 0) {
		BLI_dynstr_appendf(ds, "#define VOLUME_HOMOGENEOUS\n");
	}
	if ((options & VAR_VOLUME_LIGHT) != 0) {
		BLI_dynstr_appendf(ds, "#define VOLUME_LIGHTING\n");
	}
	if ((options & VAR_VOLUME_COLOR) != 0) {
		BLI_dynstr_appendf(ds, "#define COLOR_TRANSMITTANCE\n");
	}

	str = BLI_dynstr_get_cstring(ds);
	BLI_dynstr_free(ds);

	return str;
}

struct GPUMaterial *EEVEE_material_world_volume_get(
        struct Scene *scene, World *wo,
        bool use_lights, bool use_volume_shadows, bool is_homogeneous, bool use_color_transmit)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	int options = VAR_WORLD_VOLUME;

	if (use_lights) options |= VAR_VOLUME_LIGHT;
	if (is_homogeneous) options |= VAR_VOLUME_HOMO;
	if (use_volume_shadows) options |= VAR_VOLUME_SHADOW;
	if (use_color_transmit) options |= VAR_VOLUME_COLOR;

	GPUMaterial *mat = GPU_material_from_nodetree_find(&wo->gpumaterial, engine, options);
	if (mat != NULL) {
		return mat;
	}

	char *defines = eevee_get_volume_defines(options);

	mat = GPU_material_from_nodetree(
	        scene, wo->nodetree, &wo->gpumaterial, engine, options,
	        datatoc_background_vert_glsl, NULL, e_data.volume_shader_lib,
	        defines);

	MEM_freeN(defines);

	return mat;
}
```

*eevee_effects.c*

```

void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	e_data.volumetric_upsample_sh = DRW_shader_create_fullscreen(datatoc_volumetric_frag_glsl, "#define STEP_UPSAMPLE\n");
	...
}

void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	if ((effects->enabled_effects & EFFECT_VOLUMETRIC) != 0) {
		const DRWContextState *draw_ctx = DRW_context_state_get();
		Scene *scene = draw_ctx->scene;
		struct World *wo = scene->world; /* Already checked non NULL */
		EEVEE_VolumetricsInfo *volumetrics = sldata->volumetrics;

		struct GPUMaterial *mat = EEVEE_material_world_volume_get(
		        scene, wo, volumetrics->use_lights, volumetrics->use_volume_shadows,
		        false, volumetrics->use_colored_transmit);

		psl->volumetric_integrate_ps = DRW_pass_create("Volumetric Integration", DRW_STATE_WRITE_COLOR);
		DRWShadingGroup *grp = DRW_shgroup_material_create(mat, psl->volumetric_integrate_ps);

		if (grp != NULL) {
			DRW_shgroup_uniform_buffer(grp, "depthFull", &e_data.depth_src);
			DRW_shgroup_uniform_buffer(grp, "shadowCubes", &sldata->shadow_depth_cube_pool);
			DRW_shgroup_uniform_buffer(grp, "irradianceGrid", &sldata->irradiance_pool);
			DRW_shgroup_uniform_block(grp, "light_block", sldata->light_ubo);
			DRW_shgroup_uniform_block(grp, "grid_block", sldata->grid_ubo);
			DRW_shgroup_uniform_block(grp, "shadow_block", sldata->shadow_ubo);
			DRW_shgroup_uniform_int(grp, "light_count", &sldata->lamps->num_light, 1);
			DRW_shgroup_uniform_int(grp, "grid_count", &sldata->probes->num_render_grid, 1);
			DRW_shgroup_uniform_texture(grp, "utilTex", EEVEE_materials_get_util_tex());
			DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)stl->g_data->viewvecs, 2);
			DRW_shgroup_uniform_vec2(grp, "volume_start_end", &sldata->volumetrics->integration_start, 1);
			DRW_shgroup_uniform_vec3(grp, "volume_samples", &sldata->volumetrics->integration_step_count, 1);
			DRW_shgroup_call_add(grp, quad, NULL);

			if (volumetrics->use_colored_transmit == false) { /* Monochromatic transmittance */
				psl->volumetric_resolve_ps = DRW_pass_create("Volumetric Resolve", DRW_STATE_WRITE_COLOR | DRW_STATE_TRANSMISSION);
				grp = DRW_shgroup_create(e_data.volumetric_upsample_sh, psl->volumetric_resolve_ps);
				DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)stl->g_data->viewvecs, 2);
				DRW_shgroup_uniform_buffer(grp, "depthFull", &e_data.depth_src);
				DRW_shgroup_uniform_buffer(grp, "volumetricBuffer", &stl->g_data->volumetric);
				DRW_shgroup_call_add(grp, quad, NULL);
			}
			else {
				psl->volumetric_resolve_transmit_ps = DRW_pass_create("Volumetric Transmittance Resolve", DRW_STATE_WRITE_COLOR | DRW_STATE_MULTIPLY);
				grp = DRW_shgroup_create(e_data.volumetric_upsample_sh, psl->volumetric_resolve_transmit_ps);
				DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)stl->g_data->viewvecs, 2);
				DRW_shgroup_uniform_buffer(grp, "depthFull", &e_data.depth_src);
				DRW_shgroup_uniform_buffer(grp, "volumetricBuffer", &stl->g_data->volumetric_transmit);
				DRW_shgroup_call_add(grp, quad, NULL);

				psl->volumetric_resolve_ps = DRW_pass_create("Volumetric Resolve", DRW_STATE_WRITE_COLOR | DRW_STATE_ADDITIVE);
				grp = DRW_shgroup_create(e_data.volumetric_upsample_sh, psl->volumetric_resolve_ps);
				DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)stl->g_data->viewvecs, 2);
				DRW_shgroup_uniform_buffer(grp, "depthFull", &e_data.depth_src);
				DRW_shgroup_uniform_buffer(grp, "volumetricBuffer", &stl->g_data->volumetric);
				DRW_shgroup_call_add(grp, quad, NULL);
			}
		}
		else {
			/* Compilation failled */
			effects->enabled_effects &= ~EFFECT_VOLUMETRIC;
		}
	}
	...
}
```
<br><br>

use_lights | VOLUME_LIGHTING 宏
is_homogeneous | VOLUME_HOMOGENEOUS 宏
use_volume_shadows | VOLUME_SHADOW 宏
use_color_transmit | COLOR_TRANSMITTANCE 宏

>
- volumetric_integrate_ps 由 background_vert.glsl , volumetric_frag.glsl 组成，需要注意的是，勾选什么就定义什么宏
- volumetric_frag.glsl 中的 participating_media_properties 中的 nodetree_exec 是由连接 World Output 的Volume的节点生成的。
- volumetrics->use_colored_transmit 选项是判断是否把 Scatter 和 Transmittance 单独渲染到一张RT上，参考[这里](http://shaderstore.cn/2021/02/18/blender-eevee-2017-7-4-Volumetrics-Colored-Transmittance-Support/)


> 把 Scatter 和 Transmittance 合拼渲染到一张RT上的情况
- volumetric_integrate_ps 不会定义COLOR_TRANSMITTANCE宏
- volumetric_resolve_ps 全屏后处理，由volumetric_frag.glsl 组成，定义了  STEP_UPSAMPLE，渲染状态是 DRW_STATE_WRITE_COLOR 和 DRW_STATE_TRANSMISSION


> 把 Scatter 和 Transmittance 单独渲染到一张RT上的情况
- volumetric_integrate_ps 定义COLOR_TRANSMITTANCE宏
- volumetric_resolve_transmit_ps 全屏后处理，由volumetric_frag.glsl 组成，定义了  STEP_UPSAMPLE，渲染状态是 DRW_STATE_WRITE_COLOR 和 DRW_STATE_MULTIPLY
- volumetric_resolve_ps 全屏后处理，由volumetric_frag.glsl 组成，定义了  STEP_UPSAMPLE，渲染状态是 DRW_STATE_WRITE_COLOR 和 DRW_STATE_ADDITIVE


## 渲染

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Volumetrics */
	EEVEE_effects_do_volumetrics(sldata, vedata);
	...
}
```


*eevee_effects.c*
```
void EEVEE_effects_do_volumetrics(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_VOLUMETRIC) != 0) {
		DefaultTextureList *dtxl = DRW_viewport_texture_list_get();

		e_data.depth_src = dtxl->depth;

		/* Compute volumetric integration at halfres. */
		DRW_framebuffer_texture_attach(fbl->volumetric_fb, stl->g_data->volumetric, 0, 0);
		if (sldata->volumetrics->use_colored_transmit) {
			DRW_framebuffer_texture_attach(fbl->volumetric_fb, stl->g_data->volumetric_transmit, 1, 0);
		}
		DRW_framebuffer_bind(fbl->volumetric_fb);
		DRW_draw_pass(psl->volumetric_integrate_ps);

		/* Resolve at fullres */
		DRW_framebuffer_texture_detach(dtxl->depth);
		DRW_framebuffer_bind(fbl->main);
		if (sldata->volumetrics->use_colored_transmit) {
			DRW_draw_pass(psl->volumetric_resolve_transmit_ps);
		}
		DRW_draw_pass(psl->volumetric_resolve_ps);

		/* Restore */
		DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0, 0);
		DRW_framebuffer_texture_detach(stl->g_data->volumetric);
		if (sldata->volumetrics->use_colored_transmit) {
			DRW_framebuffer_texture_detach(stl->g_data->volumetric_transmit);
		}
	}
}
```
>
- 渲染的时候也要注意一下，use_colored_transmit 开启的话，就会调用 DRW_draw_pass(psl->volumetric_resolve_transmit_ps); 如果没有开启的话，就不用调用


## Shader
*lamps_lib.glsl*
```
float light_visibility(LightData ld, vec3 W, vec4 l_vector)
{
	float vis = 1.0;

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

#if !defined(VOLUMETRICS) || defined(VOLUME_SHADOW)
	/* shadowing */
	if (ld.l_shadowid >= (MAX_SHADOW_MAP + MAX_SHADOW_CUBE)) {
		vis *= shadow_cascade(ld.l_shadowid, W);
	}
	else if (ld.l_shadowid >= 0.0) {
		vis *= shadow_cubemap(ld.l_shadowid, l_vector);
	}
#endif

	return vis;
}
```


*volumetirc_frag.glsl*
```

#ifdef VOLUMETRICS

#define VOLUMETRIC_INTEGRATION_MAX_STEP 256
#define VOLUMETRIC_SHADOW_MAX_STEP 128

uniform int light_count;
uniform vec2 volume_start_end;
uniform vec3 volume_samples;

#define volume_start                   volume_start_end.x
#define volume_end                     volume_start_end.y

#define volume_integration_steps       volume_samples.x
#define volume_shadows_steps           volume_samples.y
#define volume_sample_distribution     volume_samples.z

#ifdef COLOR_TRANSMITTANCE
layout(location = 0) out vec4 outScattering;
layout(location = 1) out vec4 outTransmittance;
#else
out vec4 outScatteringTransmittance;
#endif

/* Warning: theses are not attributes, theses are global vars. */
vec3 worldPosition = vec3(0.0);
vec3 viewPosition = vec3(0.0);
vec3 viewNormal = vec3(0.0);

uniform sampler2D depthFull;

void participating_media_properties(vec3 wpos, out vec3 extinction, out vec3 scattering, out vec3 emission, out float anisotropy)
{
#ifndef VOLUME_HOMOGENEOUS
	worldPosition = wpos;
	viewPosition = (ViewMatrix * vec4(wpos, 1.0)).xyz; /* warning, Perf. */
#endif

	Closure cl = nodetree_exec();

	scattering = cl.scatter;
	emission = cl.emission;
	anisotropy = cl.anisotropy;
	extinction = max(vec3(1e-4), cl.absorption + cl.scatter);
}

vec3 participating_media_extinction(vec3 wpos)
{
#ifndef VOLUME_HOMOGENEOUS
	worldPosition = wpos;
	viewPosition = (ViewMatrix * vec4(wpos, 1.0)).xyz; /* warning, Perf. */
#endif

	Closure cl = nodetree_exec();

	return max(vec3(1e-4), cl.absorption + cl.scatter);
}

float phase_function_isotropic()
{
	return 1.0 / (4.0 * M_PI);
}

float phase_function(vec3 v, vec3 l, float g)
{
#ifndef VOLUME_ISOTROPIC /* TODO Use this flag when only isotropic closures are used */
	/* Henyey-Greenstein */
	float cos_theta = dot(v, l);
	g = clamp(g, -1.0 + 1e-3, 1.0 - 1e-3);
	float sqr_g = g * g;
	return (1- sqr_g) / (4.0 * M_PI * pow(1 + sqr_g - 2 * g * cos_theta, 3.0 / 2.0));
#else
	return phase_function_isotropic();
#endif
}

vec3 light_volume(LightData ld, vec4 l_vector)
{
	float power;
	float dist = max(1e-4, abs(l_vector.w - ld.l_radius));
	/* TODO : Area lighting ? */
	/* Removing Area Power. */
	/* TODO : put this out of the shader. */
	if (ld.l_type == AREA) {
		power = 0.0962 * (ld.l_sizex * ld.l_sizey * 4.0f * M_PI);
	}
	else {
		power = 0.0248 * (4.0 * ld.l_radius * ld.l_radius * M_PI * M_PI);
	}
	return ld.l_color * power / (l_vector.w * l_vector.w);
}

vec3 irradiance_volumetric(vec3 wpos)
{
	IrradianceData ir_data = load_irradiance_cell(0, vec3(1.0));
	vec3 irradiance = ir_data.cubesides[0] + ir_data.cubesides[1] + ir_data.cubesides[2];
	ir_data = load_irradiance_cell(0, vec3(-1.0));
	irradiance += ir_data.cubesides[0] + ir_data.cubesides[1] + ir_data.cubesides[2];
	irradiance *= 0.16666666; /* 1/6 */
	return irradiance;
}

vec3 light_volume_shadow(LightData ld, vec3 ray_wpos, vec4 l_vector, vec3 s_extinction)
{
#ifdef VOLUME_SHADOW

#ifdef VOLUME_HOMOGENEOUS
	/* Simple extinction */
	return exp(-s_extinction * l_vector.w);
#else
	/* Heterogeneous volume shadows */
	float dd = l_vector.w / volume_shadows_steps;
	vec3 L = l_vector.xyz * l_vector.w;
	vec3 shadow = vec3(1.0);
	for (float s = 0.5; s < VOLUMETRIC_SHADOW_MAX_STEP && s < (volume_shadows_steps - 0.1); s += 1.0) {
		vec3 pos = ray_wpos + L * (s / volume_shadows_steps);
		vec3 s_extinction = participating_media_extinction(pos);
		shadow *= exp(-s_extinction * dd);
	}
	return shadow;
#endif /* VOLUME_HOMOGENEOUS */

#else
	return vec3(1.0);
#endif /* VOLUME_SHADOW */
}

float find_next_step(float iter, float noise)
{
	float progress = (iter + noise) / volume_integration_steps;

	float linear_split = mix(volume_start, volume_end, progress);

	if (ProjectionMatrix[3][3] == 0.0) {
		float exp_split = volume_start * pow(volume_end / volume_start, progress);
		return mix(linear_split, exp_split, volume_sample_distribution);
	}
	else {
		return linear_split;
	}
}

/* Based on Frosbite Unified Volumetric.
 * https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite */
void main()
{
	vec2 uv = (gl_FragCoord.xy * 2.0) / ivec2(textureSize(depthFull, 0));
	float scene_depth = texelFetch(depthFull, ivec2(gl_FragCoord.xy) * 2, 0).r; /* use the same depth as in the upsample step */
	vec3 vpos = get_view_space_from_depth(uv, scene_depth);
	vec3 wpos = (ViewMatrixInverse * vec4(vpos, 1.0)).xyz;
	vec3 wdir = (ProjectionMatrix[3][3] == 0.0) ? normalize(cameraPos - wpos) : cameraForward;

	/* Note: this is NOT the distance to the camera. */
	float max_z = vpos.z;

	/* project ray to clip plane so we can integrate in even steps in clip space. */
	vec3 wdir_proj = wdir / abs(dot(cameraForward, wdir));
	float wlen = length(wdir_proj);

	/* Transmittance: How much light can get through. */
	vec3 transmittance = vec3(1.0);

	/* Scattering: Light that has been accumulated from scattered light sources. */
	vec3 scattering = vec3(0.0);

	vec3 ray_origin = (ProjectionMatrix[3][3] == 0.0)
		? cameraPos
		: (ViewMatrixInverse * vec4(get_view_space_from_depth(uv, 0.5), 1.0)).xyz;

#ifdef VOLUME_HOMOGENEOUS
	/* Put it out of the loop for homogeneous media. */
	vec3 s_extinction, s_scattering, s_emission;
	float s_anisotropy;
	participating_media_properties(vec3(0.0), s_extinction, s_scattering, s_emission, s_anisotropy);
#endif

	/* Start from near clip. TODO make start distance an option. */
	float rand = texture(utilTex, vec3(gl_FragCoord.xy / LUT_SIZE, 2.0)).r;
	/* Less noisy but noticeable patterns, could work better with temporal AA. */
	// float rand = (1.0 / 16.0) * float(((int(gl_FragCoord.x + gl_FragCoord.y) & 0x3) << 2) + (int(gl_FragCoord.x) & 0x3));
	float dist = volume_start;
	for (float i = 0.5; i < VOLUMETRIC_INTEGRATION_MAX_STEP && i < (volume_integration_steps - 0.1); ++i) {
		float new_dist = find_next_step(rand, i);
		float step = dist - new_dist; /* Marching step */
		dist = new_dist;

		vec3 ray_wpos = ray_origin + wdir_proj * dist;

#ifndef VOLUME_HOMOGENEOUS
		vec3 s_extinction, s_scattering, s_emission;
		float s_anisotropy;
		participating_media_properties(ray_wpos, s_extinction, s_scattering, s_emission, s_anisotropy);
#endif

		/* Evaluate each light */
		vec3 Lscat = s_emission;

#ifdef VOLUME_LIGHTING /* Lights */
		for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
			LightData ld = lights_data[i];

			vec4 l_vector;
			l_vector.xyz = ld.l_position - ray_wpos;
			l_vector.w = length(l_vector.xyz);

			float Vis = light_visibility(ld, ray_wpos, l_vector);

			vec3 Li = light_volume(ld, l_vector) * light_volume_shadow(ld, ray_wpos, l_vector, s_extinction);

			Lscat += Li * Vis * s_scattering * phase_function(-wdir, l_vector.xyz / l_vector.w, s_anisotropy);
		}
#endif

		/* Environment : Average color. */
		Lscat += irradiance_volumetric(wpos) * s_scattering * phase_function_isotropic();

		/* Evaluate Scattering */
		float s_len = wlen * step;
		vec3 Tr = exp(-s_extinction * s_len);

		/* integrate along the current step segment */
		Lscat = (Lscat - Lscat * Tr) / s_extinction;
		/* accumulate and also take into account the transmittance from previous steps */
		scattering += transmittance * Lscat;

		/* Evaluate transmittance to view independantely */
		transmittance *= Tr;

		if (dist < max_z)
			break;
	}

#ifdef COLOR_TRANSMITTANCE
	outScattering = vec4(scattering, 1.0);
	outTransmittance = vec4(transmittance, 1.0);
#else
	float mono_transmittance = dot(transmittance, vec3(1.0)) / 3.0;

	outScatteringTransmittance = vec4(scattering, mono_transmittance);
#endif
}

#else /* STEP_UPSAMPLE */

out vec4 FragColor;

uniform sampler2D depthFull;
uniform sampler2D volumetricBuffer;

uniform mat4 ProjectionMatrix;

vec4 get_view_z_from_depth(vec4 depth)
{
	vec4 d = 2.0 * depth - 1.0;
	return -ProjectionMatrix[3][2] / (d + ProjectionMatrix[2][2]);
}

void main()
{
#if 0 /* 2 x 2 with bilinear */

	const vec4 bilinear_weights[4] = vec4[4](
		vec4(9.0 / 16.0,  3.0 / 16.0, 3.0 / 16.0, 1.0 / 16.0 ),
		vec4(3.0 / 16.0,  9.0 / 16.0, 1.0 / 16.0, 3.0 / 16.0 ),
		vec4(3.0 / 16.0,  1.0 / 16.0, 9.0 / 16.0, 3.0 / 16.0 ),
		vec4(1.0 / 16.0,  3.0 / 16.0, 3.0 / 16.0, 9.0 / 16.0 )
	);

	/* Depth aware upsampling */
	vec4 depths;
	ivec2 texel_co = ivec2(gl_FragCoord.xy * 0.5) * 2;

	/* TODO use textureGather on glsl 4.0 */
	depths.x = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 0)).r;
	depths.y = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 0)).r;
	depths.z = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 2)).r;
	depths.w = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 2)).r;

	vec4 target_depth = texelFetch(depthFull, ivec2(gl_FragCoord.xy), 0).rrrr;

	depths = get_view_z_from_depth(depths);
	target_depth = get_view_z_from_depth(target_depth);

	vec4 weights = 1.0 - step(0.05, abs(depths - target_depth));

	/* Index in range [0-3] */
	int pix_id = int(dot(mod(ivec2(gl_FragCoord.xy), 2), ivec2(1, 2)));
	weights *= bilinear_weights[pix_id];

	float weight_sum = dot(weights, vec4(1.0));

	if (weight_sum == 0.0) {
		weights.x = 1.0;
		weight_sum = 1.0;
	}

	texel_co = ivec2(gl_FragCoord.xy * 0.5);

	vec4 integration_result;
	integration_result  = texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 0)) * weights.x;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 0)) * weights.y;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 1)) * weights.z;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 1)) * weights.w;

#else /* 4 x 4 */

	/* Depth aware upsampling */
	vec4 depths[4];
	ivec2 texel_co = ivec2(gl_FragCoord.xy * 0.5) * 2;

	/* TODO use textureGather on glsl 4.0 */
	texel_co += ivec2(-2, -2);
	depths[0].x = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 0)).r;
	depths[0].y = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 0)).r;
	depths[0].z = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 2)).r;
	depths[0].w = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 2)).r;

	texel_co += ivec2(4, 0);
	depths[1].x = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 0)).r;
	depths[1].y = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 0)).r;
	depths[1].z = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 2)).r;
	depths[1].w = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 2)).r;

	texel_co += ivec2(-4, 4);
	depths[2].x = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 0)).r;
	depths[2].y = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 0)).r;
	depths[2].z = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 2)).r;
	depths[2].w = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 2)).r;

	texel_co += ivec2(4, 0);
	depths[3].x = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 0)).r;
	depths[3].y = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 0)).r;
	depths[3].z = texelFetchOffset(depthFull, texel_co, 0, ivec2(0, 2)).r;
	depths[3].w = texelFetchOffset(depthFull, texel_co, 0, ivec2(2, 2)).r;

	vec4 target_depth = texelFetch(depthFull, ivec2(gl_FragCoord.xy), 0).rrrr;

	depths[0] = get_view_z_from_depth(depths[0]);
	depths[1] = get_view_z_from_depth(depths[1]);
	depths[2] = get_view_z_from_depth(depths[2]);
	depths[3] = get_view_z_from_depth(depths[3]);

	target_depth = get_view_z_from_depth(target_depth);

	vec4 weights[4];
	weights[0] = 1.0 - step(0.05, abs(depths[0] - target_depth));
	weights[1] = 1.0 - step(0.05, abs(depths[1] - target_depth));
	weights[2] = 1.0 - step(0.05, abs(depths[2] - target_depth));
	weights[3] = 1.0 - step(0.05, abs(depths[3] - target_depth));

	float weight_sum;
	weight_sum  = dot(weights[0], vec4(1.0));
	weight_sum += dot(weights[1], vec4(1.0));
	weight_sum += dot(weights[2], vec4(1.0));
	weight_sum += dot(weights[3], vec4(1.0));

	if (weight_sum == 0.0) {
		weights[0].x = 1.0;
		weight_sum = 1.0;
	}

	texel_co = ivec2(gl_FragCoord.xy * 0.5);

	vec4 integration_result;

	texel_co += ivec2(-1, -1);
	integration_result  = texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 0)) * weights[0].x;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 0)) * weights[0].y;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 1)) * weights[0].z;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 1)) * weights[0].w;

	texel_co += ivec2(2, 0);
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 0)) * weights[1].x;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 0)) * weights[1].y;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 1)) * weights[1].z;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 1)) * weights[1].w;

	texel_co += ivec2(-2, 2);
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 0)) * weights[2].x;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 0)) * weights[2].y;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 1)) * weights[2].z;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 1)) * weights[2].w;

	texel_co += ivec2(2, 0);
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 0)) * weights[3].x;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 0)) * weights[3].y;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(0, 1)) * weights[3].z;
	integration_result += texelFetchOffset(volumetricBuffer, texel_co, 0, ivec2(1, 1)) * weights[3].w;
#endif

	FragColor = integration_result / weight_sum;
}
#endif

```
>
- Shader 的原理和核心基本上都跟 [这里](http://shaderstore.cn/2021/02/10/blender-eevee-2017-7-3-Initial-Implementation-Of-Volumetrics/) 差不多