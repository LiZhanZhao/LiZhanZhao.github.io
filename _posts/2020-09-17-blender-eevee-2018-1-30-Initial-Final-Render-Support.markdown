---
layout:     post
title:      "blender eevee Initial Final Render support"
subtitle:   ""
date:       2021-4-15 19:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2018/1/30  *   Eevee : Initial Final Render support. <br> 

> 
TAA / multiple samples is not working at the moment.

> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
render to image 的功能可以正确运行

<br><br>

### eevee_render_to_image
*eevee_engine.c*
```
static void eevee_render_to_image(void *vedata, struct RenderEngine *engine, struct Depsgraph *depsgraph)
{
	EEVEE_render_init(vedata, engine, depsgraph);

	DRW_render_object_iter(vedata, engine, depsgraph, EEVEE_render_cache);
	/* Actually do the rendering. */
	EEVEE_render_draw(vedata, engine, depsgraph);
	/* Write outputs to RenderResult. */
	EEVEE_render_output(vedata, engine, depsgraph);
}
```
>
- 这个函数主要是用于 render_to_image，在进行 render_to_image 的时候，会被调用, 这个函数里面涉及到的函数在文件 *eevee_render.c* 中定义
<br><br>
- render_to_image 的快捷键是 F12

<br><br>

*eevee_render.c*
```
/** \file eevee_render.c
 *  \ingroup draw_engine
 */

/**
 * Render functions for final render outputs.
 */

#include "DRW_engine.h"
#include "DRW_render.h"

#include "BLI_rand.h"

#include "DEG_depsgraph_query.h"

#include "GPU_framebuffer.h"
#include "GPU_glew.h"

#include "RE_pipeline.h"

#include "eevee_private.h"

void EEVEE_render_init(EEVEE_Data *ved, RenderEngine *engine, struct Depsgraph *depsgraph)
{
	EEVEE_Data *vedata = (EEVEE_Data *)ved;
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_ViewLayerData *sldata = EEVEE_view_layer_data_ensure();
	Scene *scene = DEG_get_evaluated_scene(depsgraph);
	const float *viewport_size = DRW_viewport_size_get();

	/* Init default FB and render targets:
	 * In render mode the default framebuffer is not generated
	 * because there is no viewport. So we need to manually create it or
	 * not use it. For code clarity we just allocate it make use of it. */
	DefaultFramebufferList *dfbl = DRW_viewport_framebuffer_list_get();
	DefaultTextureList *dtxl = DRW_viewport_texture_list_get();

	/* NOTE : use 32 bit format for precision in render mode. */
	DRWFboTexture dtex = {&dtxl->depth, DRW_TEX_DEPTH_24_STENCIL_8, 0};
	DRW_framebuffer_init(&dfbl->default_fb, &draw_engine_eevee_type,
	                     (int)viewport_size[0], (int)viewport_size[1],
	                     &dtex, 1);

	DRWFboTexture tex = {&txl->color, DRW_TEX_RGBA_32, DRW_TEX_FILTER | DRW_TEX_MIPMAP};
	DRW_framebuffer_init(&fbl->main, &draw_engine_eevee_type,
	                     (int)viewport_size[0], (int)viewport_size[1],
	                     &tex, 1);

	/* Alloc transient data. */
	if (!stl->g_data) {
		stl->g_data = MEM_callocN(sizeof(*stl->g_data), __func__);
	}
	EEVEE_PrivateData *g_data = stl->g_data;
	g_data->background_alpha = 1.0f; /* TODO option */
	g_data->valid_double_buffer = 0;

	/* Alloc common ubo data. */
	if (sldata->common_ubo == NULL) {
		sldata->common_ubo = DRW_uniformbuffer_create(sizeof(sldata->common_data), &sldata->common_data);
	}

	/* Set the pers & view matrix. */
	struct Object *camera = RE_GetCamera(engine->re);
	float frame = BKE_scene_frame_get(scene);
	RE_GetCameraWindow(engine->re, camera, frame, g_data->winmat);
	RE_GetCameraModelMatrix(engine->re, camera, g_data->viewinv);

	invert_m4_m4(g_data->viewmat, g_data->viewinv);
	mul_m4_m4m4(g_data->persmat, g_data->winmat, g_data->viewmat);
	invert_m4_m4(g_data->persinv, g_data->persmat);
	invert_m4_m4(g_data->wininv, g_data->winmat);

	DRW_viewport_matrix_override_set(g_data->persmat, DRW_MAT_PERS);
	DRW_viewport_matrix_override_set(g_data->persinv, DRW_MAT_PERSINV);
	DRW_viewport_matrix_override_set(g_data->winmat, DRW_MAT_WIN);
	DRW_viewport_matrix_override_set(g_data->wininv, DRW_MAT_WININV);
	DRW_viewport_matrix_override_set(g_data->viewmat, DRW_MAT_VIEW);
	DRW_viewport_matrix_override_set(g_data->viewinv, DRW_MAT_VIEWINV);

	/* EEVEE_effects_init needs to go first for TAA */
	EEVEE_effects_init(sldata, vedata, camera);
	EEVEE_materials_init(sldata, stl, fbl);
	EEVEE_lights_init(sldata);
	EEVEE_lightprobes_init(sldata, vedata);

	/* INIT CACHE */
	EEVEE_bloom_cache_init(sldata, vedata);
	EEVEE_depth_of_field_cache_init(sldata, vedata);
	EEVEE_effects_cache_init(sldata, vedata);
	EEVEE_lightprobes_cache_init(sldata, vedata);
	EEVEE_lights_cache_init(sldata, psl);
	EEVEE_materials_cache_init(vedata);
	EEVEE_motion_blur_cache_init(sldata, vedata);
	EEVEE_occlusion_cache_init(sldata, vedata);
	EEVEE_screen_raytrace_cache_init(sldata, vedata);
	EEVEE_subsurface_cache_init(sldata, vedata);
	EEVEE_temporal_sampling_cache_init(sldata, vedata);
	EEVEE_volumes_cache_init(sldata, vedata);
}

void EEVEE_render_cache(
        void *vedata, struct Object *ob,
        struct RenderEngine *UNUSED(engine), struct Depsgraph *UNUSED(depsgraph))
{
	EEVEE_ViewLayerData *sldata = EEVEE_view_layer_data_ensure();

	if (DRW_check_object_visible_within_active_context(ob) == false) {
		return;
	}

	if (ELEM(ob->type, OB_MESH, OB_CURVE, OB_SURF, OB_FONT)) {
		EEVEE_materials_cache_populate(vedata, sldata, ob);

		const bool cast_shadow = true;

		if (cast_shadow) {
			EEVEE_lights_cache_shcaster_object_add(sldata, ob);
		}
	}
	else if (ob->type == OB_LIGHTPROBE) {
		EEVEE_lightprobes_cache_add(sldata, ob);
	}
	else if (ob->type == OB_LAMP) {
		EEVEE_lights_cache_add(sldata, ob);
	}
}

void EEVEE_render_draw(EEVEE_Data *vedata, struct RenderEngine *UNUSED(engine), struct Depsgraph *UNUSED(depsgraph))
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	DefaultTextureList *dtxl = DRW_viewport_texture_list_get();
	EEVEE_ViewLayerData *sldata = EEVEE_view_layer_data_ensure();
	EEVEE_PrivateData *g_data = stl->g_data;

	/* FINISH CACHE */
	EEVEE_materials_cache_finish(vedata);
	EEVEE_lights_cache_finish(sldata);
	EEVEE_lightprobes_cache_finish(sldata, vedata);

	{
		float clear_col[4] = {0.0f, 0.0f, 0.0f, 0.6f};
		unsigned int primes[3] = {2, 3, 7};
		double offset[3] = {0.0, 0.0, 0.0};
		double r[3];

		BLI_halton_3D(primes, offset, stl->effects->taa_current_sample, r);
		EEVEE_update_noise(psl, fbl, r);

		/* Refresh Probes & shadows */
		EEVEE_lightprobes_refresh(sldata, vedata);
		DRW_uniformbuffer_update(sldata->common_ubo, &sldata->common_data);
		EEVEE_draw_shadows(sldata, psl);

		DRW_viewport_matrix_override_set(g_data->persmat, DRW_MAT_PERS);
		DRW_viewport_matrix_override_set(g_data->persinv, DRW_MAT_PERSINV);
		DRW_viewport_matrix_override_set(g_data->winmat, DRW_MAT_WIN);
		DRW_viewport_matrix_override_set(g_data->wininv, DRW_MAT_WININV);
		DRW_viewport_matrix_override_set(g_data->viewmat, DRW_MAT_VIEW);
		DRW_viewport_matrix_override_set(g_data->viewinv, DRW_MAT_VIEWINV);

		DRW_framebuffer_texture_detach(dtxl->depth);
		DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0, 0);
		DRW_framebuffer_bind(fbl->main);
		DRW_framebuffer_clear(true, true, true, clear_col, 1.0f);
		/* Depth prepass */
		DRW_draw_pass(psl->depth_pass);
		DRW_draw_pass(psl->depth_pass_cull);
		/* Create minmax texture */
		EEVEE_create_minmax_buffer(vedata, dtxl->depth, -1);
		EEVEE_occlusion_compute(sldata, vedata, dtxl->depth, -1);
		EEVEE_volumes_compute(sldata, vedata);
		/* Shading pass */
		DRW_draw_pass(psl->background_pass);
		EEVEE_draw_default_passes(psl);
		DRW_draw_pass(psl->material_pass);
		EEVEE_subsurface_data_render(sldata, vedata);
		/* Effects pre-transparency */
		EEVEE_subsurface_compute(sldata, vedata);
		EEVEE_reflection_compute(sldata, vedata);
		EEVEE_occlusion_draw_debug(sldata, vedata);
		DRW_draw_pass(psl->probe_display);
		EEVEE_refraction_compute(sldata, vedata);
		/* Opaque refraction */
		DRW_draw_pass(psl->refract_depth_pass);
		DRW_draw_pass(psl->refract_depth_pass_cull);
		DRW_draw_pass(psl->refract_pass);
		/* Volumetrics Resolve Opaque */
		EEVEE_volumes_resolve(sldata, vedata);
		/* Transparent */
		DRW_pass_sort_shgroup_z(psl->transparent_pass);
		DRW_draw_pass(psl->transparent_pass);
		/* Post Process */
		EEVEE_draw_effects(sldata, vedata);
	}
}

void EEVEE_render_output(EEVEE_Data *vedata, RenderEngine *engine, struct Depsgraph *UNUSED(depsgraph))
{
	EEVEE_StorageList *stl = vedata->stl;

	const char *viewname = NULL;
	const float *render_size = DRW_viewport_size_get();

	RenderResult *rr = RE_engine_begin_result(engine, 0, 0, (int)render_size[0], (int)render_size[1], NULL, viewname);
	RenderLayer *rl = rr->layers.first;
	RenderPass *rp = RE_pass_find_by_name(rl, RE_PASSNAME_COMBINED, viewname);

	DRW_framebuffer_bind(stl->effects->final_fb);
	DRW_framebuffer_read_data(0, 0, (int)render_size[0], (int)render_size[1], 4, 0, rp->rect);

	RE_engine_end_result(engine, rr, false, false, false);
}
```

