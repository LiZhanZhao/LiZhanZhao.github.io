---
layout:     post
title:      "blender eevee LightCache: Initial Implementation"
subtitle:   ""
date:       2021-4-23 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2018/7/10  *   Eevee : LightCache: Initial Implementation. <br> 

> 
This separate probe rendering from viewport rendering, making possible to
run the baking in another thread (non blocking and faster).

>
The baked lighting is saved in the blend file. Nothing needs to be
recomputed on load.

>
There is a few missing bits / bugs:
- Cache cannot be saved to disk as a separate file, it is saved in the DNA
  for now making file larger and memory usage higher.
- Auto update only cubemaps does update the grids (bug).
- Probes cannot be updated individually (considered as dynamic).
- Light Cache cannot be (re)generated during render.


> SVN : 2018/5/28  *  Win64_vc14/Windows_vc14 add patches for boost and osl to support building with clang. 


<br><br>

## 作用
实现烘焙间接光功能，以前是实时的，现在变成了点击按钮才进行烘焙，这里其实就是改变了 间接光数据 的计算方式，Shader呈现出来的效果差不多

<br><br>

## 效果
![](/img/Eevee/LightCache/01/1.png)

<br><br>


### Bake Indirect Lighting 按钮

- 经过调式，可以发现 点击 Bake Indirect Lighting 按钮 的时候，进触发 *render_shading.c* 中的 *light_cache_bake_invoke* 函数  
<br>
- 进入 *render_shading.c* 中的 *light_cache_bake_invoke* 函数 里面，会先调用 *eevee_lightcache.c* 的 *EEVEE_lightbake_job_create* 函数  
<br>
- 进入 *eevee_lightcache.c* 的 *EEVEE_lightbake_job_create* 函数里面，首先会调用自身文件的 *EEVEE_lightbake_job* 函数，其次就是调用自身文件的 *EEVEE_lightbake_update* 函数  

<br><br>

#### EEVEE_lightbake_job

*eevee_lightcache.c*
```
typedef struct EEVEE_LightBake {
	Depsgraph *depsgraph;
	ViewLayer *view_layer;
	ViewLayer *view_layer_input;
	LightCache *lcache;
	Scene *scene;
	struct Main *bmain;

	LightProbe **probe;              /* Current probe being rendered. */
	GPUTexture *rt_color;            /* Target cube color texture. */
	GPUTexture *rt_depth;            /* Target cube depth texture. */
	GPUFrameBuffer *rt_fb[6];        /* Target cube framebuffers. */
	GPUFrameBuffer *store_fb;        /* Storage framebuffer. */
	int rt_res;                      /* Cube render target resolution. */

	/* Shared */
	int layer;                       /* Target layer to store the data to. */
	float samples_ct, invsamples_ct; /* Sample count for the convolution. */
	float lod_factor;                /* Sampling bias during convolution step. */
	float lod_max;                   /* Max cubemap LOD to sample when convolving. */
	int cube_len, grid_len;          /* Number of probes to render + world probe. */

	/* Irradiance grid */
	EEVEE_LightGrid *grid;           /* Current probe being rendered (UBO data). */
	int irr_cube_res;                /* Target cubemap at MIP 0. */
	int irr_size[3];                 /* Size of the irradiance texture. */
	int total_irr_samples;           /* Total for all grids */
	int grid_sample;                 /* Nth sample of the current grid being rendered. */
	int grid_sample_len;             /* Total number of samples for the current grid. */
	int grid_curr;                   /* Nth grid in the cache being rendered. */
	int bounce_curr, bounce_len;     /* The current light bounce being evaluated. */
	float vis_range, vis_blur;       /* Sample Visibility compression and bluring. */
	float vis_res;                   /* Resolution of the Visibility shadowmap. */
	GPUTexture *grid_prev;           /* Result of previous light bounce. */
	LightProbe **grid_prb;           /* Pointer to the id.data of the probe object. */

	/* Reflection probe */
	EEVEE_LightProbe *cube;          /* Current probe being rendered (UBO data). */
	int ref_cube_res;                /* Target cubemap at MIP 0. */
	int cube_offset;                 /* Index of the current cube. */
	float probemat[6][4][4];         /* ViewProjection matrix for each cube face. */
	float texel_size, padding_size;  /* Texel and padding size for the final octahedral map. */
	float roughness;                 /* Roughness level of the current mipmap. */
	LightProbe **cube_prb;           /* Pointer to the id.data of the probe object. */

	/* Dummy Textures */
	struct GPUTexture *dummy_color, *dummy_depth;
	struct GPUTexture *dummy_layer_color;

	int total, done; /* to compute progress */
	short *stop, *do_update;
	float *progress;

	bool resource_only;              /* For only handling the resources. */
	bool own_resources;
	bool own_light_cache;            /* If the lightcache was created for baking, it's first owned by the baker. */
	int delay;                       /* ms. delay the start of the baking to not slowdown interactions (TODO remove) */

	void *gl_context, *gwn_context;  /* If running in parallel (in a separate thread), use this context. */
} EEVEE_LightBake;
```
>
- EEVEE_LightBake 保存了很多lightBake的有用信息的结构体

<br><br>

```
void EEVEE_lightbake_job(void *custom_data, short *stop, short *do_update, float *progress)
{
	EEVEE_LightBake *lbake = (EEVEE_LightBake *)custom_data;
	Depsgraph *depsgraph = lbake->depsgraph;
	int frame = 0; /* TODO make it user param. */

	DEG_graph_relations_update(depsgraph, lbake->bmain, lbake->scene, lbake->view_layer_input);
	DEG_evaluate_on_framechange(lbake->bmain, depsgraph, frame);

	lbake->view_layer = DEG_get_evaluated_view_layer(depsgraph);
	lbake->stop = stop;
	lbake->do_update = do_update;
	lbake->progress = progress;

	/* Count lightprobes */
	// 填充 lbake 数据，例如 total_irr_samples 和 grid_len 和  lbake->cube_len
	eevee_lightbake_count_probes(lbake);   


	/* We need to create the FBOs in the right context.
	 * We cannot do it in the main thread. */
	eevee_lightbake_context_enable(lbake);

	// 填充 lbake 数据, bounce_len 和 vis_res 和 rt_res 等
	eevee_lightbake_create_resources(lbake);
	
	// 填充 lbake 数据, 创建 rt_depth 和 rt_color
	eevee_lightbake_create_render_target(lbake, lbake->rt_res);

	eevee_lightbake_context_disable(lbake);

	/* Gather all probes data */
	// 填充 lbake 数据, grid_data
	eevee_lightbake_gather_probes(lbake);

	LightCache *lcache = lbake->lcache;

	/* HACK: Sleep to delay the first rendering operation
	 * that causes a small freeze (caused by VBO generation)
	 * because this step is locking at this moment. */
	/* TODO remove this. */
	if (lbake->delay) {
		PIL_sleep_ms(lbake->delay);
	}

	/* Render world irradiance and reflection first */
	if (lcache->flag & LIGHTCACHE_UPDATE_WORLD) {
		lbake->probe = NULL;
		lightbake_do_sample(lbake, eevee_lightbake_render_world_sample);
	}

	/* Render irradiance grids */
	if (lcache->flag & LIGHTCACHE_UPDATE_GRID) {
		for (lbake->bounce_curr = 0; lbake->bounce_curr < lbake->bounce_len; ++lbake->bounce_curr) {
			/* Bypass world, start at 1. */
			lbake->probe = lbake->grid_prb + 1;
			lbake->grid = lcache->grid_data + 1;
			for (lbake->grid_curr = 1;
			     lbake->grid_curr < lbake->grid_len;
			     ++lbake->grid_curr, ++lbake->probe, ++lbake->grid)
			{
				LightProbe *prb = *lbake->probe;
				lbake->grid_sample_len = prb->grid_resolution_x *
				                         prb->grid_resolution_y *
				                         prb->grid_resolution_z;
				for (lbake->grid_sample = 0;
				     lbake->grid_sample < lbake->grid_sample_len;
				     ++lbake->grid_sample)
				{
					lightbake_do_sample(lbake, eevee_lightbake_render_grid_sample);
				}
			}
		}
	}

	/* Render reflections */
	if (lcache->flag & LIGHTCACHE_UPDATE_CUBE) {
		/* Bypass world, start at 1. */
		lbake->probe = lbake->cube_prb + 1;
		lbake->cube = lcache->cube_data + 1;
		for (lbake->cube_offset = 1;
		     lbake->cube_offset < lbake->cube_len;
		     ++lbake->cube_offset, ++lbake->probe, ++lbake->cube)
		{
			lightbake_do_sample(lbake, eevee_lightbake_render_probe_sample);
		}
	}

	/* Read the resulting lighting data to save it to file/disk. */
	eevee_lightbake_context_enable(lbake);
	eevee_lightbake_readback_irradiance(lcache);
	eevee_lightbake_readback_reflections(lcache);
	eevee_lightbake_context_disable(lbake);

	lcache->flag |=  LIGHTCACHE_BAKED;
	lcache->flag &= ~LIGHTCACHE_BAKING;

	/* Assume that if lbake->gl_context is NULL
	 * we are not running in this in a job, so update
	 * the scene lightcache pointer before deleting it. */
	if (lbake->gl_context == NULL) {
		BLI_assert(BLI_thread_is_main());
		EEVEE_lightbake_update(lbake);
	}

	eevee_lightbake_delete_resources(lbake);
}
```
>
- 一开始是各种填充 EEVEE_LightBake 结构体数据
<br><br>  

- Render world irradiance and reflection first 使用 eevee_lightbake_render_world_sample 函数  
<br><br>

- Render irradiance grids 使用 eevee_lightbake_render_probe_sample 函数
<br><br>

- Render reflections 使用 eevee_lightbake_render_probe_sample 函数
<br><br>

- 以上的 渲染函数，grids 把结果渲染到 lcache->grid_tx.tex 上，cube 把结果渲染到 lcache->cube_tx.tex 上，最用 变成 lcache->grid_tx.data + lcache->cube_tx.data


<br><br>

### EEVEE_lightprobes_refresh 渲染

在*eevee_engine.c* 的 *eevee_draw_background*中, 会进行调用 EEVEE_lightprobes_refresh 渲染, 但是这个 EEVEE_lightprobes_refresh 现在只会进行渲染World, 不会进行 其他的了。


EEVEE_lightprobes_refresh 就会调用 *eevee_lightprobes.c* 中的 *EEVEE_lightbake_update_world_quick*

<br>

#### EEVEE_lightbake_update_world_quick

*eevee_lightcache.c*
```
/* This is to update the world irradiance and reflection contribution from
 * within the viewport drawing (does not have the overhead of a full light cache rebuild.) */
void EEVEE_lightbake_update_world_quick(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata, const Scene *scene)
{
	LightCache *lcache = vedata->stl->g_data->light_cache;

	EEVEE_LightBake lbake = {
		.resource_only = true
	};

	/* Create resources. */
	eevee_lightbake_create_render_target(&lbake, scene->eevee.gi_cubemap_resolution);

	EEVEE_lightbake_cache_init(sldata, vedata, lbake.rt_color, lbake.rt_depth);

	EEVEE_lightbake_render_world(sldata, vedata, lbake.rt_fb);
	EEVEE_lightbake_filter_glossy(sldata, vedata, lbake.rt_color, lbake.store_fb, 0, 1.0f, lcache->mips_len);
	EEVEE_lightbake_filter_diffuse(sldata, vedata, lbake.rt_color, lbake.store_fb, 0, 1.0f);

	/* Don't hide grids if they are already rendered. */
	lcache->grid_len = max_ii(1, lcache->grid_len);
	lcache->cube_len = 1;

	lcache->flag |= LIGHTCACHE_CUBE_READY | LIGHTCACHE_GRID_READY;
	lcache->flag &= ~LIGHTCACHE_UPDATE_WORLD;

	eevee_lightbake_delete_resources(&lbake);
}
```
>
- 这里最主要的就是要结合 lcache->grid_tx.tex 进行保存数据