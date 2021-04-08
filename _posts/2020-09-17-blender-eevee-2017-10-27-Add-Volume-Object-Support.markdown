---
layout:     post
title:      "blender eevee Volumetrics Add Volume object support"
subtitle:   ""
date:       2021-4-8 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/10/27  *   Eevee : Volumetrics: Add Volume object support. <br> 

> This is quite basic as it only support boundbing boxes.
But the material can refine the volume shape in anyway the user like.

>
To overcome this limitation, a voxelisation should be done on the mesh (generating a SDF maybe?) and tested against every volumetric cell.

> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
物体可以进行 Volume


## 效果
*想要看到效果, Viewport Samples(TAA Sample) 要先设置为 1*
![](/img/Eevee/Volumetrics+/02/1.png)

<br><br>


### 渲染入口

*eevee_effects.c*

```
void EEVEE_effects_do_volumetrics(EEVEE_SceneLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_VOLUMETRIC) != 0) {
		DefaultTextureList *dtxl = DRW_viewport_texture_list_get();

		e_data.color_src = txl->color;
		e_data.depth_src = dtxl->depth;

		/* Step 1: Participating Media Properties */
		DRW_framebuffer_bind(fbl->volumetric_fb);
		DRW_draw_pass(psl->volumetric_world_ps);
		DRW_draw_pass(psl->volumetric_objects_ps);

		/* Step 2: Scatter Light */
		DRW_framebuffer_bind(fbl->volumetric_scat_fb);
		DRW_draw_pass(psl->volumetric_scatter_ps);

		/* Step 3: Integration */
		DRW_framebuffer_bind(fbl->volumetric_integ_fb);
		DRW_draw_pass(psl->volumetric_integration_ps);

		/* Step 4: Apply for opaque */
		DRW_framebuffer_bind(fbl->effect_fb);
		DRW_draw_pass(psl->volumetric_resolve_ps);

		/* Swap volume history buffers */
		SWAP(struct GPUFrameBuffer *, fbl->volumetric_scat_fb, fbl->volumetric_integ_fb);
		SWAP(GPUTexture *, txl->volume_scatter, txl->volume_scatter_history);
		SWAP(GPUTexture *, txl->volume_transmittance, txl->volume_transmittance_history);

		/* Swap the buffers and rebind depth to the current buffer */
		DRW_framebuffer_texture_detach(dtxl->depth);
		SWAP(struct GPUFrameBuffer *, fbl->main, fbl->effect_fb);
		SWAP(GPUTexture *, txl->color, txl->color_post);
		DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0, 0);
	}
}
```
>
- 这里最主要是 Step 1: Participating Media Properties 中，多了 DRW_draw_pass(psl->volumetric_objects_ps);


### volumetric_objects_ps

#### 1.初始化