---
layout:     post
title:      "blender eevee Probes: Add data display for cubemaps"
subtitle:   ""
date:       2021-2-1 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/15  * Eevee: Probes: Add data display for cubemaps.<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/ProbesCubemaps/1.png)
![](/img/Eevee/ProbesCubemaps/3.png)
![](/img/Eevee/ProbesCubemaps/2.png)

## 作用 
显示Iradiance Cubemap ，方便debug

## 编译
- 重新生成SLN
- git 定位到  2017/6/15  * Eevee: Probes: Add data display for cubemaps.<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 渲染
跟[blender eevee Add Grid debug display And Multiple Bounces](http://shaderstore.cn/2021/01/29/blender-eevee-2017-6-14-Add-Grid-Debug-Display-And-Multiple-Bounces/) 差不多道理

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
- 用 probe_display 来 Debug.


*eevee_lightprobes.c*
```
void EEVEE_lightprobes_init(EEVEE_SceneLayerData *sldata)
{
	...
	ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, datatoc_octahedron_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_lightprobe_cube_display_frag_glsl);
	shader_str = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	e_data.probe_cube_display_sh = DRW_shader_create(
			datatoc_lightprobe_cube_display_vert_glsl, NULL, shader_str, NULL);
	...
}

void EEVEE_lightprobes_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, EEVEE_StorageList *stl)
{
	...
	psl->probe_display = DRW_pass_create("LightProbe Display", DRW_STATE_WRITE_COLOR | DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS);

	struct Batch *geom = DRW_cache_sphere_get();
	DRWShadingGroup *grp = stl->g_data->cube_display_shgrp = DRW_shgroup_instance_create(e_data.probe_cube_display_sh, psl->probe_display, geom);
	DRW_shgroup_attrib_float(grp, "probe_id", 1); /* XXX this works because we are still uploading 4bytes and using the right stride */
	DRW_shgroup_attrib_float(grp, "probe_location", 3);
	DRW_shgroup_attrib_float(grp, "sphere_size", 1);
	DRW_shgroup_uniform_float(grp, "lodMax", &sldata->probes->lodmax, 1);
	DRW_shgroup_uniform_buffer(grp, "probeCubes", &sldata->probe_pool);
	...
}



static void EEVEE_lightprobes_updates(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, EEVEE_StorageList *stl)
{
	...
	/* Debug Display */
	if ((probe->flag & LIGHTPROBE_FLAG_SHOW_DATA) != 0) {
		DRW_shgroup_call_dynamic_add(stl->g_data->cube_display_shgrp, &ped->probe_id, ob->obmat[3], &probe->data_draw_size);
	}
	...
}


```
>
- cube_display_shgrp 由 lightprobe_cube_display_vert.glsl 和 lightprobe_cube_display_frag.glsl 组成, 在Probe切换到cubemap的时候，就probe_display使用cube_display_shgrp来进行渲染。

## Shader
*lightprobe_cube_display_vert.glsl*
```

in vec3 pos;
in vec3 nor;

/* Instance attrib */
in int probe_id;
in vec3 probe_location;
in float sphere_size;

uniform mat4 ViewProjectionMatrix;

flat out int pid;
out vec3 worldNormal;
out vec3 worldPosition;

void main()
{
	pid = probe_id;
	worldPosition = pos * 0.1 * sphere_size + probe_location;
	gl_Position = ViewProjectionMatrix * vec4(worldPosition, 1.0);
	worldNormal = normalize(nor);
}
```


*lightprobe_cube_display_frag.glsl*
```

uniform mat4 ProjectionMatrix;
uniform mat4 ViewMatrixInverse;

uniform sampler2DArray probeCubes;
uniform float lodMax;

flat in int pid;
in vec3 worldNormal;
in vec3 worldPosition;

out vec4 FragColor;

#define cameraForward   normalize(ViewMatrixInverse[2].xyz)
#define cameraPos       ViewMatrixInverse[3].xyz

void main()
{
	vec3 V = (ProjectionMatrix[3][3] == 0.0) /* if perspective */
	            ? normalize(cameraPos - worldPosition)
	            : cameraForward;
	vec3 N = normalize(worldNormal);
	FragColor = vec4(textureLod_octahedron(probeCubes, vec4(reflect(-V, N), pid), 0.0, lodMax).rgb, 1.0);
}

```

>
- 这里其实就是用sphere来显示Cubemap的octahedron贴图