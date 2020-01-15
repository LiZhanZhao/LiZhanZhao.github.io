---
layout:     post
title:      "blender eevee 初始化"
subtitle:   ""
date:       2019-12-20 11:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## eevee 渲染过程

```c
static void EEVEE_draw_scene(void)
{
	EEVEE_Data *ved = DRW_viewport_engine_data_get(EEVEE_ENGINE);
	EEVEE_PassList *psl = ved->psl;
	EEVEE_FramebufferList *fbl = ved->fbl;

	/* Default framebuffer and texture */
	DefaultFramebufferList *dfbl = DRW_viewport_framebuffer_list_get();
	DefaultTextureList *dtxl = DRW_viewport_texture_list_get();

	/* Attach depth to the hdr buffer and bind it */	
	DRW_framebuffer_texture_detach(dtxl->depth);
	DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0);
	DRW_framebuffer_bind(fbl->main);

	/* Clear Depth */
	/* TODO do background */
	float clearcol[4] = {0.0f, 0.0f, 0.0f, 1.0f};
	DRW_framebuffer_clear(true, true, clearcol, 1.0f);

	DRW_draw_pass(psl->pass);

	/* Restore default framebuffer */
	DRW_framebuffer_texture_detach(dtxl->depth);
	DRW_framebuffer_texture_attach(dfbl->default_fb, dtxl->depth, 0);
	DRW_framebuffer_bind(dfbl->default_fb);

	DRW_draw_pass(psl->tonemap);
}
```
以上的渲染过程可以简单理解为 :  ***clear -> draw -> tonemap***

DRW_framebuffer_clear(true, true, clearcol, 1.0f);  

DRW_draw_pass(psl->pass);  

DRW_draw_pass(psl->tonemap);  

## DRW_draw_pass(psl->pass)
通过阅读代码，会发现 psl->pass 的使用的shader就是 lit_surface_vert.glsl 和 lit_surface_frag.glsl 分别是  

*vs*
```glsl

uniform mat4 ModelViewProjectionMatrix;
uniform mat4 ModelMatrix;
uniform mat3 WorldNormalMatrix;

in vec3 pos;
in vec3 nor;

out vec3 worldPosition;
out vec3 worldNormal;

void main() {
	gl_Position = ModelViewProjectionMatrix * vec4(pos, 1.0);
	worldPosition = (ModelMatrix * vec4(pos, 1.0)).xyz;
	worldNormal = WorldNormalMatrix * nor;
}


```


*ps*
```glsl

uniform int light_count;

struct LightData {
	vec4 position;
	vec4 colorAndSpec; /* w : Spec Intensity */
	vec4 spotAndAreaData;
};

layout(std140) uniform light_block {
	LightData   lights_data[MAX_LIGHT];
};

in vec3 worldPosition;
in vec3 worldNormal;

out vec4 fragColor;

void main() {
	vec3 n = normalize(worldNormal);
	vec3 diffuse = vec3(0.0);

	for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
		LightData ld = lights_data[i];
		vec3 l = normalize(ld.position.xyz - worldPosition);
		diffuse += max(0.0, dot(l, n)) * ld.colorAndSpec.rgb / 3.14159;
	}

	fragColor = vec4(diffuse,1.0);
}
```


## DRW_draw_pass(psl->tonemap)

通过阅读代码，会发现 psl->tonemap 的使用的shader就是 lit_surface_vert.glsl 

```glsl

uniform sampler2D hdrColorBuf;

in vec4 uvcoordsvar;

out vec4 fragColor;

void main() {
	fragColor = texture(hdrColorBuf, uvcoordsvar.st);
}
```

## 总结
上面的代码非常简单，传入灯光数据，进行灯光计算，后期使用的tonemap也只是直接绘制原来的图，没有什么特别的，不过这个是了解eevee的渲染流程的好开始。