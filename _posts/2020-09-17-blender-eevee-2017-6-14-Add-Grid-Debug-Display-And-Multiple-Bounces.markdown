---
layout:     post
title:      "blender eevee Add Grid debug display And Multiple Bounces"
subtitle:   ""
date:       2021-1-29 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/14  * Eevee: Add Grid debug display.<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/IradianceGrid/Display/1.png)
![](/img/Eevee/IradianceGrid/Display/2.png)

## 作用 
显示IradianceGrid，方便debug

## 编译
- 重新生成SLN
- git 定位到  2017/6/14  * Eevee: Add Grid debug display.<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## 渲染
*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	DRW_draw_pass(psl->probe_display);
	...

}
```
>
- 在主要的渲染管线上多了 probe_display

## probe_display
*eevee_lightprobes.c*

```

void EEVEE_lightprobes_init(EEVEE_SceneLayerData *sldata)
{
	...
	ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, datatoc_irradiance_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_lightprobe_grid_display_frag_glsl);
	shader_str = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	e_data.probe_grid_display_sh = DRW_shader_create(
			datatoc_lightprobe_grid_display_vert_glsl, NULL, shader_str,
#if defined(IRRADIANCE_SH_L2)
			"#define IRRADIANCE_SH_L2\n"
#elif defined(IRRADIANCE_CUBEMAP)
			"#define IRRADIANCE_CUBEMAP\n"
#elif defined(IRRADIANCE_HL2)
			"#define IRRADIANCE_HL2\n"
#endif
			);
}
```


```
static void EEVEE_lightprobes_updates(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	...
	/* Debug Display */
	if ((probe->flag & LIGHTPROBE_FLAG_SHOW_INFLUENCE) != 0) {
		struct Batch *geom = DRW_cache_sphere_get();

		DRWShadingGroup *grp = DRW_shgroup_instance_create(e_data.probe_grid_display_sh, psl->probe_display, geom);

		DRW_shgroup_set_instance_count(grp, num_cell);

		DRW_shgroup_uniform_int(grp, "offset", &egrid->offset, 1);
		DRW_shgroup_uniform_ivec3(grp, "grid_resolution", egrid->resolution, 1);
		DRW_shgroup_uniform_vec3(grp, "corner", egrid->corner, 1);
		DRW_shgroup_uniform_vec3(grp, "increment_x", egrid->increment_x, 1);
		DRW_shgroup_uniform_vec3(grp, "increment_y", egrid->increment_y, 1);
		DRW_shgroup_uniform_vec3(grp, "increment_z", egrid->increment_z, 1);
		DRW_shgroup_uniform_buffer(grp, "irradianceGrid", &sldata->irradiance_pool);
	}
}
```
>
- probe_display 由 lightprobe_grid_display_vert.glsl, irradiance_lib.glsl,  lightprobe_grid_display_frag.glsl 构成
- <br>
```
struct Batch *geom = DRW_cache_sphere_get();
DRWShadingGroup *grp = DRW_shgroup_instance_create(e_data.probe_grid_display_sh, psl->probe_display, geom);
DRW_shgroup_set_instance_count(grp, num_cell);
```
instance numcell个球 

## Shader
*lightprobe_grid_display_vert.glsl*
```

in vec3 pos;
in vec3 nor;

uniform mat4 ViewProjectionMatrix;

uniform int offset;
uniform ivec3 grid_resolution;
uniform vec3 corner;
uniform vec3 increment_x;
uniform vec3 increment_y;
uniform vec3 increment_z;

flat out int cellOffset;
out vec3 worldNormal;

void main()
{
	vec3 ls_cell_location;
	/* Keep in sync with update_irradiance_probe */
	ls_cell_location.z = float(gl_InstanceID % grid_resolution.z);
	ls_cell_location.y = float((gl_InstanceID / grid_resolution.z) % grid_resolution.y);
	ls_cell_location.x = float(gl_InstanceID / (grid_resolution.z * grid_resolution.y));

	cellOffset = offset + gl_InstanceID;

	vec3 ws_cell_location = corner +
	    (increment_x * ls_cell_location.x +
	     increment_y * ls_cell_location.y +
	     increment_z * ls_cell_location.z);

	gl_Position = ViewProjectionMatrix * vec4(pos * 0.02 + ws_cell_location, 1.0);
	worldNormal = normalize(nor);
}
```

*lightprobe_grid_display_frag.glsl*
```

flat in int cellOffset;
in vec3 worldNormal;

out vec4 FragColor;

void main()
{
	IrradianceData ir_data = load_irradiance_cell(cellOffset, worldNormal);
	FragColor = vec4(compute_irradiance(worldNormal, ir_data), 1.0);
}

```
>
- 这个可以参考 [Add-Irradiance-Grid-Support](http://shaderstore.cn/2021/01/28/blender-eevee-2017-6-13-Add-Irradiance-Grid-Support/)
- 这个是基本是Iradiance的精髓了


# Multiple Bounces

## 来源

- 主要看这个commit

> GIT : 2017/6/15  * Eevee: Irradiance grid: support for non-blocking update and multiple bounces.<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51

## 渲染
*eevee_lightprobes.c*
```
void EEVEE_lightprobes_refresh(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	...
	/* Reflection probes depend on diffuse lighting thus on irradiance grid */
	const int max_bounce = 3;
	while (pinfo->updated_bounce < max_bounce) {
		pinfo->num_render_grid = pinfo->num_grid;

		for (int i = 1; (ob = pinfo->probes_grid_ref[i]) && (i < MAX_GRID); i++) {
			EEVEE_LightProbeEngineData *ped = EEVEE_lightprobe_data_get(ob);

			if (ped->need_update) {
				EEVEE_LightGrid *egrid = &pinfo->grid_data[i];
				LightProbe *prb = (LightProbe *)ob->data;
				int cell_id = ped->updated_cells;

				SWAP(GPUTexture *, sldata->irradiance_pool, sldata->irradiance_rt);

				/* Temporary Remove all probes. */
				int tmp_num_render_grid = pinfo->num_render_grid;
				int tmp_num_render_cube = pinfo->num_render_cube;
				pinfo->num_render_cube = 0;

				/* Use light from previous bounce when capturing radiance. */
				if (pinfo->updated_bounce == 0) {
					pinfo->num_render_grid = 0;
				}

				float pos[3];
				lightprobe_cell_location_get(egrid, cell_id, pos);

				render_scene_to_probe(sldata, psl, pos, prb->clipsta, prb->clipend);
				diffuse_filter_probe(sldata, psl, egrid->offset + cell_id);

				/* Restore */
				pinfo->num_render_grid = tmp_num_render_grid;
				pinfo->num_render_cube = tmp_num_render_cube;

				/* To see what is going on. */
				SWAP(GPUTexture *, sldata->irradiance_pool, sldata->irradiance_rt);

				ped->updated_cells++;
				if (ped->updated_cells >= ped->num_cell) {
					ped->need_update = false;
				}

				/* Only do one probe per frame */
				DRW_viewport_request_redraw();
				return;
			}
		}

		pinfo->updated_bounce++;
		pinfo->num_render_grid = pinfo->num_grid;

		if (pinfo->updated_bounce < max_bounce) {
			/* Retag all grids to update for next bounce */
			for (int i = 1; (ob = pinfo->probes_grid_ref[i]) && (i < MAX_GRID); i++) {
				EEVEE_LightProbeEngineData *ped = EEVEE_lightprobe_data_get(ob);
				ped->need_update = true;
				ped->updated_cells = 0;
			}
			SWAP(GPUTexture *, sldata->irradiance_pool, sldata->irradiance_rt);
		}
	}
	...
}
```
>
- 个人理解 Multiple Bounces 的实现方式就是渲染多次, 例如Iradiance Grid的一个cell进行拍摄cubemap的时候,第一次bounes的时候,物体是没有受到Iradiance影响的,得到的cubemap是没有带Iradiance信息的,而第二次bounes的时候,那就是第二次拍摄cubemap的时候,物体是带有Iradiance的,那么个人理解这样就可以实现 Multiple Bounces