---
layout:     post
title:      "blender eevee World default shader"
subtitle:   ""
date:       2020-07-21 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> 2017/4/26   * Eevee: World default shader.
- Use uniform color world for the world probe.
- Refactored the Fresnel expression to be better with Area Lights.
- Squared the roughness for default materials.

## 作用 
*渲染环境贴图 probe_rt*


## 编译

- 直接编译


## 效果展示
![](/img/Eevee/WorldDefaultShader/world-shader.png)


## CPU

### 渲染
*eevee_probes* 
```c

void EEVEE_refresh_probe(EEVEE_Data *vedata)
{
	...
	DRW_framebuffer_bind(fbl->probe_fb);
	DRW_draw_pass(psl->probe_background);
	...
}
```
> 
-  先绑定 probe_fb frameBuffer, 再用 probe_background 进行渲染


### 绑定frameBuffer probe_fb，probe_fb 关联 tex_probe

>
- 在 eevee_probes.c 文件中的 EEVEE_probes_init 函数中，可以找 DRW_framebuffer_init(&fbl->probe_fb, PROBE_SIZE, PROBE_SIZE, tex_probe, 2);  probe_fb 关联了 tex_probe



### 初始化 tex_probe，tex_probe 关联 probe_rt

>
- 在 eevee_probes.c 文件中的 EEVEE_probes_init 函数中，可以找 tex_probe 关联 txl->probe_rt。
- 那就是渲染 probe_fb，其实就是把东西渲染到 *probe_rt* 上


### probe_background, 关联 default_world

*eevee.c*
```c
const DRWContextState *draw_ctx = DRW_context_state_get();
Scene *scene = draw_ctx->scene;
World *wo = scene->world;

if (false) { /* TODO check for world nodetree */
	// GPUMaterial *gpumat = GPU_material_from_nodetree(struct bNodeTree *ntree, ListBase *gpumaterials, void *engine_type, int options)
}
else {
	float *col = ts.colorBackground;
	static int zero = 0;

	if (wo) {
		col = &wo->horr;
	}

	grp = eevee_cube_shgroup(e_data.default_world, psl->probe_background, geom);
	DRW_shgroup_uniform_int(grp, "Layer", &zero, 1);
	DRW_shgroup_uniform_vec3(grp, "color", col, 1);
}
```
>
- eevee_cube_shgroup(e_data.default_world, psl->probe_background, geom); 表示 probe_background 关联 default_world
- DRW_shgroup_uniform_vec3(grp, "color", col, 1); 直接传入参数到shader中，这个color是World中进行设置


### default_world
```glsl
if (!e_data.default_world) {
		e_data.default_world = DRW_shader_create(
		        datatoc_probe_vert_glsl, datatoc_probe_geom_glsl, datatoc_default_world_frag_glsl, NULL);
	}
```
>
-  default_world 由  probe_vert.glsl , probe_geom.glsl, default_world_frag.glsl 组成




## Shader

### default_world_frag.glsl
```glsl

uniform vec3 color;

out vec4 FragColor;

void main() {
	FragColor = vec4(color, 1.0);
}

```
>
- 比较简单，直接输出颜色


## 使用 probe_rt
- 上面的步骤就是用 default_world ( probe_vert.glsl , probe_geom.glsl, default_world_frag.glsl ) 进行渲染东西到 纹理 probe_rt 上

- 然后把 probe_rt 传入到  probe_prefilter 和 probe_sh_compute 中，用于 Probe Filtering 和 Probe SH Compute, 如下:

```c
{
	psl->probe_prefilter = DRW_pass_create("Probe Filtering", DRW_STATE_WRITE_COLOR);

	struct Batch *geom = DRW_cache_fullscreen_quad_get();
	DRWShadingGroup *grp = eevee_cube_shgroup(e_data.probe_filter_sh, psl->probe_prefilter, geom);
	DRW_shgroup_uniform_float(grp, "sampleCount", &stl->probes->samples_ct, 1);
	DRW_shgroup_uniform_float(grp, "invSampleCount", &stl->probes->invsamples_ct, 1);
	DRW_shgroup_uniform_float(grp, "roughnessSquared", &stl->probes->roughness, 1);
	DRW_shgroup_uniform_float(grp, "lodFactor", &stl->probes->lodfactor, 1);
	DRW_shgroup_uniform_float(grp, "lodMax", &stl->probes->lodmax, 1);
	DRW_shgroup_uniform_int(grp, "Layer", &stl->probes->layer, 1);
	DRW_shgroup_uniform_texture(grp, "texHammersley", e_data.hammersley, 0);
	// DRW_shgroup_uniform_texture(grp, "texJitter", e_data.jitter, 1);
	DRW_shgroup_uniform_texture(grp, "probeHdr", txl->probe_rt, 3);
}

{
	psl->probe_sh_compute = DRW_pass_create("Probe SH Compute", DRW_STATE_WRITE_COLOR);

	DRWShadingGroup *grp = DRW_shgroup_create(e_data.probe_spherical_harmonic_sh, psl->probe_sh_compute);
	DRW_shgroup_uniform_int(grp, "probeSize", &stl->probes->shres, 1);
	DRW_shgroup_uniform_float(grp, "lodBias", &stl->probes->lodfactor, 1);
	DRW_shgroup_uniform_texture(grp, "probeHdr", txl->probe_rt, 0);

	struct Batch *geom = DRW_cache_fullscreen_quad_get();
	DRW_shgroup_call_add(grp, geom, NULL);
}

```