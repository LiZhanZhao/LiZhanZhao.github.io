---
layout:     post
title:      "blender eevee Add Data Display For Planar Reflection."
subtitle:   ""
date:       2021-2-3 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/17  * Eevee : Add data display for planar reflection. .<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/PlanarReflections/Display/1.png)
![](/img/Eevee/PlanarReflections/Display/2.png)


## 作用 
Display 镜面反射,可以用来debug

## 编译
- 重新生成SLN
- git 定位到  2017/6/17  * Eevee : Add data display for planar reflection.  .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 渲染
*eevee_lightprobes.c*
```
void EEVEE_lightprobes_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *UNUSED(vedata))
{
	...
	ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, datatoc_bsdf_common_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_lightprobe_planar_display_frag_glsl);
	shader_str = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	e_data.probe_planar_display_sh = DRW_shader_create(
			datatoc_lightprobe_planar_display_vert_glsl, NULL, shader_str,
			"#define MAX_PLANAR " STRINGIFY(MAX_PLANAR) "\n");
}
```
>
- Planar Display 使用的 lightprobe_planar_display_vert.glsl 和 lightprobe_planar_display_frag.glsl  

<br>
<br>


```

static void EEVEE_planar_reflections_updates(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, EEVEE_TextureList *txl)
{
	...
	/* Debug Display */
	if ((probe->flag & LIGHTPROBE_FLAG_SHOW_DATA) != 0) {
		DRWShadingGroup *grp = DRW_shgroup_create(e_data.probe_planar_display_sh, psl->probe_display);

		DRW_shgroup_uniform_int(grp, "probeIdx", &ped->probe_id, 1);
		DRW_shgroup_uniform_buffer(grp, "probePlanars", &txl->planar_pool);
		DRW_shgroup_uniform_block(grp, "planar_block", sldata->planar_ubo);

		struct Batch *geom = DRW_cache_fullscreen_quad_get();
		DRW_shgroup_call_add(grp, geom, ob->obmat);
	}
}
```
>
- psl->probe_display 用来Debug probe, 这里Debug Planar 使用的是一个 fullscreen quad，需要注意的是，这个所谓的fullscreen quad 其实是一个三角形，可以参考[这里](https://www.slideshare.net/DevCentralAMD/vertex-shader-tricks-bill-bilodeau), 这里Display思路就是多渲染一个 三角形模型用来显示渲染好的RT，这个三角形和probe的obmat矩阵是绑定在一起,就相当于 probe多了一个plane mesh的感觉<br>
这个三角形 Pos : {-1.0f, -1.0f}, { 3.0f, -1.0f}, {-1.0f,  3.0f}, uvs : { 0.0f,  0.0f}, { 2.0f,  0.0f}, { 0.0f,  2.0f}
- ![](/img/Eevee/PlanarReflections/Display/3.png)
- ![](/img/Eevee/PlanarReflections/Display/4.png)


## Shader

*lightprobe_planar_display_vert.glsl*
```
in vec3 pos;

uniform mat4 ModelViewProjectionMatrix;
uniform mat4 ModelMatrix;

out vec3 worldPosition;

void main()
{
	gl_Position = ModelViewProjectionMatrix * vec4(pos, 1.0);
	worldPosition = (ModelMatrix * vec4(pos, 1.0)).xyz;
}
```
>
- ModelViewProjectionMatrix 是跟Probe有关的，没有经过Z Plane 镜面的，只是普通的MVP矩阵



*lightprobe_planar_display_frag.glsl*
```

uniform int probeIdx;
uniform sampler2DArray probePlanars;

layout(std140) uniform planar_block {
	PlanarData planars_data[MAX_PLANAR];
};

in vec3 worldPosition;

out vec4 FragColor;

void main()
{
	PlanarData pd = planars_data[probeIdx];

	/* Fancy fast clipping calculation */
	vec2 dist_to_clip;
	dist_to_clip.x = dot(pd.pl_clip_pos_x, worldPosition);
	dist_to_clip.y = dot(pd.pl_clip_pos_y, worldPosition);
	float fac = dot(step(pd.pl_clip_edges, dist_to_clip.xxyy), vec2(-1.0, 1.0).xyxy); /* compare and add all tests */

	if (fac != 2.0) {
		discard;
	}

	vec4 refco = pd.reflectionmat * vec4(worldPosition, 1.0);
	refco.xy /= refco.w;
	FragColor = vec4(textureLod(probePlanars, vec3(refco.xy, float(probeIdx)), 0.0).rgb, 1.0);
}

```
>
- 不在 Reflection Planar  内就clip，这里就是简单显示 probePlanars RT，pd.reflectionmat 是没有经过镜面的矩阵，[上一篇](http://shaderstore.cn/2021/02/02/blender-eevee-2017-6-17-Initial-Implementation-Of-Planar-Reflections/)有解析