---
layout:     post
title:      "blender eevee Add Grid debug display"
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