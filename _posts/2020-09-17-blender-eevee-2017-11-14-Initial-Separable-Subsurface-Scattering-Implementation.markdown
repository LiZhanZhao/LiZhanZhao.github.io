---
layout:     post
title:      "blender eevee Initial Separable Subsurface Scattering implementation"
subtitle:   ""
date:       2021-4-9 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/11/14  *   Eevee : Initial Separable Subsurface Scattering implementation. <br> 

> How to use:
- Enable subsurface scattering in the render options.
- Add Subsurface BSDF to your shader.
- Check "Screen Space Subsurface Scattering" in the material panel options.

> This initial implementation has a few limitations:
- only supports gaussian SSS.
- Does not support principled shader.
- The radius parameters is baked down to a number of samples and then put into an UBO. This means the radius input socket cannot be used. You need to tweak the default vector directly.
- The "texture blur" is considered as always set to 1


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
实现初级版本的SSS效果
<br><br>

## 效果

![](/img/Eevee/SSS/01/1.png)
![](/img/Eevee/SSS/01/2.png)

<br><br>

### 渲染入口

*eevee_engine.c*
```
static void EEVEE_engine_init(void *ved)
{
    ...
    /* EEVEE_effects_init needs to go first for TAA */
	EEVEE_effects_init(sldata, vedata);
    ...
}
```

<br>

*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
    ...
    effects->enabled_effects |= EEVEE_subsurface_init(sldata, vedata);
    ...
}
```

<br>

*eevee_engine.c*
```
static void EEVEE_cache_init(void *vedata)
{
    ...
    EEVEE_subsurface_cache_init(sldata, vedata);
    ...
}

static void EEVEE_draw_scene(void *vedata)
{
    ...
    /* Shading pass */
    DRW_stats_group_start("Shading");
    DRW_draw_pass(psl->background_pass);
    EEVEE_draw_default_passes(psl);
    DRW_draw_pass(psl->material_pass);
    EEVEE_subsurface_data_render(sldata, vedata); //<-----------------
    DRW_stats_group_end();
    ...

    /* Effects pre-transparency */
	EEVEE_subsurface_compute(sldata, vedata);       //<-----------------

    ...

}
```
>
- 这里需要注意一下执行的顺序
<br><br>
- 执行 EEVEE_subsurface_init
<br><br>
- 执行 EEVEE_subsurface_cache_init
<br><br>
- 执行 EEVEE_subsurface_data_render
<br><br>
- 执行 EEVEE_subsurface_compute

<br><br>

### 渲染流程
*eevee_subsurface.c*
```

#include "DRW_render.h"

#include "BLI_dynstr.h"

#include "eevee_private.h"
#include "GPU_texture.h"

/* SSR shader variations */
enum {
	SSR_SAMPLES      = (1 << 0) | (1 << 1),
	SSR_RESOLVE      = (1 << 2),
	SSR_FULL_TRACE   = (1 << 3),
	SSR_MAX_SHADER   = (1 << 4),
};

static struct {
	/* Screen Space SubSurfaceScattering */
	struct GPUShader *sss_sh[2];
} e_data = {NULL}; /* Engine data */

extern char datatoc_effect_subsurface_frag_glsl[];

static void eevee_create_shader_subsurface(void)
{
	e_data.sss_sh[0] = DRW_shader_create_fullscreen(datatoc_effect_subsurface_frag_glsl, "#define FIRST_PASS\n");
	e_data.sss_sh[1] = DRW_shader_create_fullscreen(datatoc_effect_subsurface_frag_glsl, "#define SECOND_PASS\n");
}

int EEVEE_subsurface_init(EEVEE_SceneLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_TextureList *txl = vedata->txl;
	const float *viewport_size = DRW_viewport_size_get();

	const DRWContextState *draw_ctx = DRW_context_state_get();
	SceneLayer *scene_layer = draw_ctx->scene_layer;
	IDProperty *props = BKE_scene_layer_engine_evaluated_get(scene_layer, COLLECTION_MODE_NONE, RE_engine_id_BLENDER_EEVEE);

	if (BKE_collection_engine_property_value_get_bool(props, "sss_enable")) {

		/* Shaders */
		if (!e_data.sss_sh[0]) {
			eevee_create_shader_subsurface();
		}

		/* NOTE : we need another stencil because the stencil buffer is on the same texture
		 * as the depth buffer we are sampling from. This could be avoided if the stencil is
		 * a separate texture but that needs OpenGL 4.4 or ARB_texture_stencil8.
		 * OR OpenGL 4.3 / ARB_ES3_compatibility if using a renderbuffer instead */
		DRWFboTexture texs[2] = {&txl->sss_stencil, DRW_TEX_DEPTH_24_STENCIL_8, 0},
		                         {&txl->sss_blur, DRW_TEX_RGBA_16, DRW_TEX_FILTER};

		DRW_framebuffer_init(&fbl->sss_blur_fb, &draw_engine_eevee_type, (int)viewport_size[0], (int)viewport_size[1],
		                     texs, 2);

		DRWFboTexture tex_data = {&txl->sss_data, DRW_TEX_RGBA_16, DRW_TEX_FILTER};
		DRW_framebuffer_init(&fbl->sss_clear_fb, &draw_engine_eevee_type, (int)viewport_size[0], (int)viewport_size[1],
		                     &tex_data, 1);

		return EFFECT_SSS;
	}

	/* Cleanup to release memory */
	DRW_TEXTURE_FREE_SAFE(txl->sss_data);
	DRW_TEXTURE_FREE_SAFE(txl->sss_blur);
	DRW_TEXTURE_FREE_SAFE(txl->sss_stencil);
	DRW_FRAMEBUFFER_FREE_SAFE(fbl->sss_blur_fb);
	DRW_FRAMEBUFFER_FREE_SAFE(fbl->sss_clear_fb);

	return 0;
}

void EEVEE_subsurface_cache_init(EEVEE_SceneLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_SSS) != 0) {
		/** Screen Space SubSurface Scattering overview
		 * TODO
		 */
		psl->sss_blur_ps = DRW_pass_create("Blur Horiz", DRW_STATE_WRITE_COLOR | DRW_STATE_STENCIL_EQUAL);

		psl->sss_resolve_ps = DRW_pass_create("Blur Vert", DRW_STATE_WRITE_COLOR | DRW_STATE_ADDITIVE | DRW_STATE_STENCIL_EQUAL);
	}
}

void EEVEE_subsurface_add_pass(EEVEE_Data *vedata, unsigned int sss_id, struct GPUUniformBuffer *sss_profile)
{
	DefaultTextureList *dtxl = DRW_viewport_texture_list_get();
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_PassList *psl = vedata->psl;
	struct Gwn_Batch *quad = DRW_cache_fullscreen_quad_get();

	DRWShadingGroup *grp = DRW_shgroup_create(e_data.sss_sh[0], psl->sss_blur_ps);
	DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)vedata->stl->g_data->viewvecs, 2);
	DRW_shgroup_uniform_texture(grp, "utilTex", EEVEE_materials_get_util_tex());
	DRW_shgroup_uniform_buffer(grp, "depthBuffer", &dtxl->depth);
	DRW_shgroup_uniform_buffer(grp, "sssData", &txl->sss_data);
	DRW_shgroup_uniform_block(grp, "sssProfile", sss_profile);
	DRW_shgroup_stencil_mask(grp, sss_id);
	DRW_shgroup_call_add(grp, quad, NULL);

	grp = DRW_shgroup_create(e_data.sss_sh[1], psl->sss_resolve_ps);
	DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)vedata->stl->g_data->viewvecs, 2);
	DRW_shgroup_uniform_texture(grp, "utilTex", EEVEE_materials_get_util_tex());
	DRW_shgroup_uniform_buffer(grp, "depthBuffer", &dtxl->depth);
	DRW_shgroup_uniform_buffer(grp, "sssData", &txl->sss_blur);
	DRW_shgroup_uniform_block(grp, "sssProfile", sss_profile);
	DRW_shgroup_stencil_mask(grp, sss_id);
	DRW_shgroup_call_add(grp, quad, NULL);
}

void EEVEE_subsurface_data_render(EEVEE_SceneLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_SSS) != 0) {
		float clear[4] = {0.0f, 0.0f, 0.0f, 0.0f};
		/* Clear sss_data texture only... can this be done in a more clever way? */
		DRW_framebuffer_bind(fbl->sss_clear_fb);
		DRW_framebuffer_clear(true, false, false, clear, 0.0f);

		if ((effects->enabled_effects & EFFECT_NORMAL_BUFFER) != 0) {
			DRW_framebuffer_texture_detach(txl->ssr_normal_input);
		}
		if ((effects->enabled_effects & EFFECT_SSR) != 0) {
			DRW_framebuffer_texture_detach(txl->ssr_specrough_input);
		}
		if ((effects->enabled_effects & EFFECT_NORMAL_BUFFER) != 0) {
			DRW_framebuffer_texture_attach(fbl->main, txl->ssr_normal_input, 2, 0);
		}
		if ((effects->enabled_effects & EFFECT_SSR) != 0) {
			DRW_framebuffer_texture_attach(fbl->main, txl->ssr_specrough_input, 3, 0);
		}
		DRW_framebuffer_texture_detach(txl->sss_data);
		DRW_framebuffer_texture_attach(fbl->main, txl->sss_data, 1, 0);
		DRW_framebuffer_bind(fbl->main);

		DRW_draw_pass(psl->sss_pass);

		if ((effects->enabled_effects & EFFECT_NORMAL_BUFFER) != 0) {
			DRW_framebuffer_texture_detach(txl->ssr_normal_input);
		}
		if ((effects->enabled_effects & EFFECT_SSR) != 0) {
			DRW_framebuffer_texture_detach(txl->ssr_specrough_input);
		}
		DRW_framebuffer_texture_detach(txl->sss_data);
		if ((effects->enabled_effects & EFFECT_NORMAL_BUFFER) != 0) {
			DRW_framebuffer_texture_attach(fbl->main, txl->ssr_normal_input, 1, 0);
		}
		if ((effects->enabled_effects & EFFECT_SSR) != 0) {
			DRW_framebuffer_texture_attach(fbl->main, txl->ssr_specrough_input, 2, 0);
		}
		DRW_framebuffer_texture_attach(fbl->sss_clear_fb, txl->sss_data, 0, 0);
	}
}

void EEVEE_subsurface_compute(EEVEE_SceneLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_SSS) != 0) {
		float clear[4] = {0.0f, 0.0f, 0.0f, 0.0f};
		DefaultTextureList *dtxl = DRW_viewport_texture_list_get();

		DRW_stats_group_start("SSS");

		/* Copy stencil channel, could be avoided (see EEVEE_subsurface_init) */
		DRW_framebuffer_blit(fbl->main, fbl->sss_blur_fb, false, true);

		DRW_framebuffer_texture_detach(dtxl->depth);

		/* First horizontal pass */
		DRW_framebuffer_bind(fbl->sss_blur_fb);
		DRW_framebuffer_clear(true, false, false, clear, 0.0f);
		DRW_draw_pass(psl->sss_blur_ps);

		/* First vertical pass + Resolve */
		DRW_framebuffer_texture_detach(txl->sss_stencil);
		DRW_framebuffer_texture_attach(fbl->main, txl->sss_stencil, 0, 0);
		DRW_framebuffer_bind(fbl->main);
		DRW_draw_pass(psl->sss_resolve_ps);

		/* Restore */
		DRW_framebuffer_texture_detach(txl->sss_stencil);
		DRW_framebuffer_texture_attach(fbl->sss_blur_fb, txl->sss_stencil, 0, 0);
		DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0, 0);

		DRW_stats_group_end();
	}
}

void EEVEE_subsurface_free(void)
{
	DRW_SHADER_FREE_SAFE(e_data.sss_sh[0]);
	DRW_SHADER_FREE_SAFE(e_data.sss_sh[1]);
}

```
>
- 第一步，在 EEVEE_subsurface_data_render 中，利用 psl->sss_pass 把数据渲染到 RT txl->sss_data 和 RT txl->ssr_normal_input 和 RT txl->ssr_specrough_input 上 <br><br>
- RT txl->ssr_normal_input 和 RT txl->ssr_specrough_input 是根据 EFFECT_NORMAL_BUFFER 和 EFFECT_SSR 来判断是否进行渲染的<br><br>
- 第二步，/* First horizontal pass */ 在 EEVEE_subsurface_compute 中，利用 psl->sss_blur_ps 把东西渲染到 framebuffer fbl->sss_blur_fb 上，fbl->sss_blur_fb 绑定了 RT txl->sss_stencil 和 RT txl->sss_blur，这一步会传入 RT txl->sss_data 到Shader 中 <br><br>
- 第三步，/* First vertical pass + Resolve */ 在 EEVEE_subsurface_compute 中，利用 psl->sss_resolve_ps 把东西渲染到 RT  txl->sss_stencil 上，这一步会传入到 RT txl->sss_blur 到 Shader 中


<br>

### sss_pass

#### 1. 初始化
*eevee_materials.c*
```
void EEVEE_materials_cache_init(EEVEE_Data *vedata)
{
    ...
    {
		DRWState state = DRW_STATE_WRITE_COLOR | DRW_STATE_DEPTH_EQUAL | DRW_STATE_CLIP_PLANES | DRW_STATE_WIRE | DRW_STATE_WRITE_STENCIL;
		psl->sss_pass = DRW_pass_create("Subsurface Pass", state);
		e_data.sss_count = 0;
	}
    ...
}
```


#### 2. 应用
*eevee_materials.c*
```

static char *eevee_get_defines(int options)
{
	char *str = NULL;

	DynStr *ds = BLI_dynstr_new();
	BLI_dynstr_appendf(ds, SHADER_DEFINES);

	...
	if ((options & VAR_MAT_SSS) != 0) {
		BLI_dynstr_appendf(ds, "#define USE_SSS\n");
	}
	...

	str = BLI_dynstr_get_cstring(ds);
	BLI_dynstr_free(ds);

	return str;
}

struct GPUMaterial *EEVEE_material_mesh_get(
        struct Scene *scene, Material *ma, EEVEE_Data *vedata,
        bool use_blend, bool use_multiply, bool use_refract, bool use_sss, int shadow_method)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	int options = VAR_MAT_MESH;

	if (use_blend) options |= VAR_MAT_BLEND;
	if (use_multiply) options |= VAR_MAT_MULT;
	if (use_refract) options |= VAR_MAT_REFRACT;
	if (use_sss) options |= VAR_MAT_SSS;
	if (vedata->stl->effects->use_volumetrics && use_blend) options |= VAR_MAT_VOLUME;

	options |= eevee_material_shadow_option(shadow_method);

	GPUMaterial *mat = GPU_material_from_nodetree_find(&ma->gpumaterial, engine, options);
	if (mat) {
		return mat;
	}

	char *defines = eevee_get_defines(options);

	mat = GPU_material_from_nodetree(
	        scene, ma->nodetree, &ma->gpumaterial, engine, options,
	        datatoc_lit_surface_vert_glsl, NULL, e_data.frag_shader_lib,
	        defines);

	MEM_freeN(defines);

	return mat;
}


void EEVEE_subsurface_add_pass(EEVEE_Data *vedata, unsigned int sss_id, struct GPUUniformBuffer *sss_profile)
{
	DefaultTextureList *dtxl = DRW_viewport_texture_list_get();
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_PassList *psl = vedata->psl;
	struct Gwn_Batch *quad = DRW_cache_fullscreen_quad_get();

	DRWShadingGroup *grp = DRW_shgroup_create(e_data.sss_sh[0], psl->sss_blur_ps);
	DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)vedata->stl->g_data->viewvecs, 2);
	DRW_shgroup_uniform_texture(grp, "utilTex", EEVEE_materials_get_util_tex());
	DRW_shgroup_uniform_buffer(grp, "depthBuffer", &dtxl->depth);
	DRW_shgroup_uniform_buffer(grp, "sssData", &txl->sss_data);
	DRW_shgroup_uniform_block(grp, "sssProfile", sss_profile);
	DRW_shgroup_stencil_mask(grp, sss_id);
	DRW_shgroup_call_add(grp, quad, NULL);

	grp = DRW_shgroup_create(e_data.sss_sh[1], psl->sss_resolve_ps);
	DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)vedata->stl->g_data->viewvecs, 2);
	DRW_shgroup_uniform_texture(grp, "utilTex", EEVEE_materials_get_util_tex());
	DRW_shgroup_uniform_buffer(grp, "depthBuffer", &dtxl->depth);
	DRW_shgroup_uniform_buffer(grp, "sssData", &txl->sss_blur);
	DRW_shgroup_uniform_block(grp, "sssProfile", sss_profile);
	DRW_shgroup_stencil_mask(grp, sss_id);
	DRW_shgroup_call_add(grp, quad, NULL);
}


static void material_opaque(
        Material *ma, GHash *material_hash, EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata,
        bool do_cull, bool use_flat_nor, struct GPUMaterial **gpumat, struct GPUMaterial **gpumat_depth,
        struct DRWShadingGroup **shgrp, struct DRWShadingGroup **shgrp_depth, struct DRWShadingGroup **shgrp_depth_clip)
{
    ...
    const bool use_gpumat = (ma->use_nodes && ma->nodetree);
	const bool use_refract = ((ma->blend_flag & MA_BL_SS_REFRACTION) != 0) && ((stl->effects->enabled_effects & EFFECT_REFRACT) != 0);
	const bool use_sss = ((ma->blend_flag & MA_BL_SS_SUBSURFACE) != 0) && ((stl->effects->enabled_effects & EFFECT_SSS) != 0);

	EeveeMaterialShadingGroups *emsg = BLI_ghash_lookup(material_hash, (const void *)ma);

	if (emsg) {
		*shgrp = emsg->shading_grp;
		*shgrp_depth = emsg->depth_grp;
		*shgrp_depth_clip = emsg->depth_clip_grp;

		/* This will have been created already, just perform a lookup. */
		*gpumat = (use_gpumat) ? EEVEE_material_mesh_get(
		        scene, ma, vedata, false, false, use_refract, use_sss, linfo->shadow_method) : NULL;
		*gpumat_depth = (use_gpumat) ? EEVEE_material_mesh_depth_get(
		        scene, ma, (ma->blend_method == MA_BM_HASHED), false) : NULL;
		return;
	}

	if (use_gpumat) {
		/* Shading */
		*gpumat = EEVEE_material_mesh_get(scene, ma, vedata, false, false, use_refract, use_sss, linfo->shadow_method);

		*shgrp = DRW_shgroup_material_create(*gpumat,
		                                     (use_refract) ? psl->refract_pass :
		                                     (use_sss) ? psl->sss_pass : psl->material_pass);
		if (*shgrp) {
			static int no_ssr = -1;
			static int first_ssr = 0;
			int *ssr_id = (stl->effects->use_ssr && !use_refract) ? &first_ssr : &no_ssr;
			add_standard_uniforms(*shgrp, sldata, vedata, ssr_id, &ma->refract_depth, use_refract, false);

			if (use_sss) {
                //<------
				struct GPUUniformBuffer *sss_profile = GPU_material_sss_profile_get(*gpumat);
				if (sss_profile) {
					DRW_shgroup_stencil_mask(*shgrp, e_data.sss_count + 1);
                    //<------
					EEVEE_subsurface_add_pass(vedata, e_data.sss_count + 1, sss_profile); 
					e_data.sss_count++;
				}
			}
		}
		else {
			/* Shader failed : pink color */
			static float col[3] = {1.0f, 0.0f, 1.0f};
			static float half = 0.5f;

			color_p = col;
			metal_p = spec_p = rough_p = &half;
		}

		/* Alpha CLipped : Discard pixel from depth pass, then
		 * fail the depth test for shading. */
		...
	}
    ...
}
```
> use_sss 为 true 的时候<br><br>
- 物体加入到 psl->sss_pass 里面进行渲染，Shader 会定义了 #define USE_SSS
<br><br>
- 计算 sss_profile = GPU_material_sss_profile_get(*gpumat);
<br><br>
- 设置模板值，DRW_shgroup_stencil_mask
<br><br>
- 调用 EEVEE_subsurface_add_pass ，在 EEVEE_subsurface_add_pass 中，是增加 sss_blur_ps Pass 的渲染个数，还有 sss_resolve_ps Pass
<br><br>
- 这里个人理解就是 如果一个物体 进行 SSS 渲染，那么 就会 就会把这个物体 添加到 psl->sss_pass 中，然后 计算 这个物体的 sss_profile 数据，然后 sss_blur_ps 和 sss_resolve_ps 都会增加多一个后处理，需要传入 sss_profile 数据作为输入。

<br><br>

#### 3. Shader
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
	result.sss_data.rgb = eevee_surface_diffuse_lit(N, vec3(1.0), 1.0) * color.rgb;
	result.sss_data.a = scale;
#else
	node_bsdf_diffuse(color, 0.0, N, result);
#endif
}
```
>
- 因为使用了NodeTree， subsurface scattering 节点，

<br><br>

### sss_profile 计算

#### 1. GPU_material_sss_profile_get 函数

*gpu_material.c*
```
struct GPUUniformBuffer *GPU_material_sss_profile_get(GPUMaterial *material)
{
	if (material->sss_dirty) {
		GPU_material_sss_profile_update(material);
	}
	return material->sss_profile;
}

static void GPU_material_sss_profile_update(GPUMaterial *material)
{
	GPUSssKernelData kd;

	compute_sss_kernel(&kd, material->sss_radii);

	/* Update / Create UBO */
	GPU_uniformbuffer_update(material->sss_profile, &kd);

	material->sss_dirty = false;
}

static void sss_calculate_offsets(GPUSssKernelData *kd)
{
	float step = 2.0f / (float)(SSS_SAMPLES - 1);
	for (int i = 0; i < SSS_SAMPLES; i++) {
		float o = ((float)i) * step - 1.0f;
		float sign = (o < 0.0f) ? -1.0f : 1.0f;
		float ofs = sign * fabsf(powf(o, SSS_EXPONENT));
		kd->kernel[i][3] = ofs;
	}
}

static float gaussian_primitive(float x) {
	const float sigma = 0.3f; /* Contained mostly between -1..1 */
	return 0.5f * error_function(x / ((float)M_SQRT2 * sigma));
}

static float gaussian_integral(float x0, float x1) {
	return gaussian_primitive(x0) - gaussian_primitive(x1);
}

static void compute_sss_kernel(GPUSssKernelData *kd, float *radii)
{
	/* Normalize size */
	copy_v3_v3(kd->radii_n, radii);
	kd->max_radius = MAX3(kd->radii_n[0], kd->radii_n[1], kd->radii_n[2]);
	mul_v3_fl(kd->radii_n, 1.0f / kd->max_radius);

	/* Compute samples locations on the 1d kernel */
	sss_calculate_offsets(kd);

#if 0 /* Maybe used for other distributions */
	/* Calculate areas (using importance-sampling) */
	float areas[SSS_SAMPLES];
	sss_calculate_areas(&kd, areas);
#endif

	/* Weights sum for normalization */
	float sum[3] = {0.0f, 0.0f, 0.0f};

	/* Compute interpolated weights */
	for (int i = 0; i < SSS_SAMPLES; i++) {
		float x0, x1;

		if (i == 0) {
			x0 = kd->kernel[0][3] - abs(kd->kernel[0][3] - kd->kernel[1][3]) / 2.0f;
		}
		else {
			x0 = (kd->kernel[i - 1][3] + kd->kernel[i][3]) / 2.0f;
		}

		if (i == SSS_SAMPLES - 1) {
			x1 = kd->kernel[SSS_SAMPLES - 1][3] + abs(kd->kernel[SSS_SAMPLES - 2][3] - kd->kernel[SSS_SAMPLES - 1][3]) / 2.0f;
		}
		else {
			x1 = (kd->kernel[i][3] + kd->kernel[i + 1][3]) / 2.0f;
		}

		kd->kernel[i][0] = gaussian_integral(x0 / kd->radii_n[0], x1 / kd->radii_n[0]);
		kd->kernel[i][1] = gaussian_integral(x0 / kd->radii_n[1], x1 / kd->radii_n[1]);
		kd->kernel[i][2] = gaussian_integral(x0 / kd->radii_n[2], x1 / kd->radii_n[2]);

		sum[0] += kd->kernel[i][0];
		sum[1] += kd->kernel[i][1];
		sum[2] += kd->kernel[i][2];
	}

	/* Normalize */
	for (int i = 0; i < SSS_SAMPLES; i++) {
		 kd->kernel[i][0] /= sum[0];
		 kd->kernel[i][1] /= sum[1];
		 kd->kernel[i][2] /= sum[2];
	}

	/* Put center sample at the start of the array (to sample first) */
	float tmpv[4];
	copy_v4_v4(tmpv, kd->kernel[SSS_SAMPLES / 2]);
	for (int i = SSS_SAMPLES / 2; i > 0; i--) {
		copy_v4_v4(kd->kernel[i], kd->kernel[i - 1]);
	}
	copy_v4_v4(kd->kernel[0], tmpv);
}

```
>
- 这里就是计算 sss_profile 的主要过程， 在 gpu_material.c 中可以找到
- 理论先不记录，TODO

<br><br>

#### 2. sss_radii 

*node_shader_subsurface_scattering.c*
```
static int node_shader_gpu_subsurface_scattering(GPUMaterial *mat, bNode *node, bNodeExecData *UNUSED(execdata), GPUNodeStack *in, GPUNodeStack *out)
{
	if (!in[5].link)
		GPU_link(mat, "world_normals_get", &in[5].link);

	if (node->sss_id == 0) {
		bNodeSocket *socket = BLI_findlink(&node->original->inputs, 2);
		bNodeSocketValueRGBA *socket_data = socket->default_value;
		/* For some reason it seems that the socket value is in ARGB format. */
		GPU_material_sss_profile_create(mat, &socket_data->value[1]);
	}

	return GPU_stack_link(mat, node, "node_subsurface_scattering", in, out, GPU_uniform(&node->sss_id));
}
```

*gpu_material.c*
```
void GPU_material_sss_profile_create(GPUMaterial *material, float *radii)
{
	material->sss_radii = radii;
	material->sss_dirty = true;

	/* Update / Create UBO */
	if (material->sss_profile == NULL) {
		material->sss_profile = GPU_uniformbuffer_create(sizeof(GPUSssKernelData), NULL, NULL);
	}
}
```
>
- 这里可以只是 sss_radii 是从node获得的

<br>


### 设置模板值 
*draw_manager.c*
```
void DRW_shgroup_stencil_mask(DRWShadingGroup *shgroup, unsigned int mask)
{
	shgroup->stencil_mask = mask;
}
```
>
- 这里个人理解就是 SSS 会进行 stencil

<br><br>


### sss_blur_ps + sss_resolve_ps

#### 1. sss_blur_ps 初始化
看到上面的 *eevee_subsurface.c* 可以 得到 sss_blur_ps 由  effect_subsurface_frag.glsl 组成，定义了宏 #define FIRST_PASS

<br><br>

#### 2. sss_resolve_ps 初始化
sss_resolve_ps 由  effect_subsurface_frag.glsl 组成，定义了宏 #define SECOND_PASS

<br><br>

#### 3. Shader 

```
/* Based on Separable SSS. by Jorge Jimenez and Diego Gutierrez */

#define SSS_SAMPLES 25
layout(std140) uniform sssProfile {
	vec4 kernel[SSS_SAMPLES];
	vec4 radii_max_radius;
};

uniform sampler2D depthBuffer;
uniform sampler2D sssData;
uniform sampler2DArray utilTex;

out vec4 FragColor;

uniform mat4 ProjectionMatrix;
uniform vec4 viewvecs[2];

float get_view_z_from_depth(float depth)
{
	if (ProjectionMatrix[3][3] == 0.0) {
		float d = 2.0 * depth - 1.0;
		return -ProjectionMatrix[3][2] / (d + ProjectionMatrix[2][2]);
	}
	else {
		return viewvecs[0].z + depth * viewvecs[1].z;
	}
}

vec3 get_view_space_from_depth(vec2 uvcoords, float depth)
{
	if (ProjectionMatrix[3][3] == 0.0) {
		return (viewvecs[0].xyz + vec3(uvcoords, 0.0) * viewvecs[1].xyz) * get_view_z_from_depth(depth);
	}
	else {
		return viewvecs[0].xyz + vec3(uvcoords, depth) * viewvecs[1].xyz;
	}
}

#define LUT_SIZE 64
#define M_PI_2     1.5707963267948966        /* pi/2 */
#define M_2PI      6.2831853071795865        /* 2*pi */

void main(void)
{
	vec2 pixel_size = 1.0 / vec2(textureSize(depthBuffer, 0).xy); /* TODO precompute */
	vec2 uvs = gl_FragCoord.xy * pixel_size;
	vec4 sss_data = texture(sssData, uvs).rgba;
	float depth_view = get_view_z_from_depth(texture(depthBuffer, uvs).r);

	float rand = texelFetch(utilTex, ivec3(ivec2(gl_FragCoord.xy) % LUT_SIZE, 2), 0).r;
#ifdef FIRST_PASS
	float angle = M_2PI * rand + M_PI_2;
	vec2 dir = vec2(1.0, 0.0);
#else /* SECOND_PASS */
	float angle = M_2PI * rand;
	vec2 dir = vec2(0.0, 1.0);
#endif
	vec2 dir_rand = vec2(cos(angle), sin(angle));

	/* Compute kernel bounds in 2D. */
	float homcoord = ProjectionMatrix[2][3] * depth_view + ProjectionMatrix[3][3];
	vec2 scale = vec2(ProjectionMatrix[0][0], ProjectionMatrix[1][1]) * sss_data.aa / homcoord;
	vec2 finalStep = scale * radii_max_radius.w;
	finalStep *= 0.5; /* samples range -1..1 */

	/* Center sample */
	vec3 accum = sss_data.rgb * kernel[0].rgb;

	for (int i = 1; i < SSS_SAMPLES; i++) {
		/* Rotate samples that are near the kernel center. */
		vec2 sample_uv = uvs + kernel[i].a * finalStep * ((abs(kernel[i].a) > 0.3) ? dir : dir_rand);
		vec3 color = texture(sssData, sample_uv).rgb;
		float sample_depth = texture(depthBuffer, sample_uv).r;
		sample_depth = get_view_z_from_depth(sample_depth);

		/* Depth correction factor. */
		float depth_delta = depth_view - sample_depth;
		float s = clamp(1.0 - exp(-(depth_delta * depth_delta) / (2.0 * sss_data.a)), 0.0, 1.0);

		/* Out of view samples. */
		if (any(lessThan(sample_uv, vec2(0.0))) || any(greaterThan(sample_uv, vec2(1.0)))) {
			s = 1.0;
		}

		accum += kernel[i].rgb * mix(color, sss_data.rgb, s);
	}

#ifdef FIRST_PASS
	FragColor = vec4(accum, sss_data.a);
#else /* SECOND_PASS */
	FragColor = vec4(accum, 1.0);
#endif
}

```






