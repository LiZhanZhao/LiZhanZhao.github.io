---
layout:     post
title:      "blender-eevee-2017-3-18-commit"
subtitle:   ""
date:       2019-12-11 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 阅读背景
阅读前请完成 [blender-编译源码](http://shaderstore.cn/2019/12/11/blender-1/) 。这次主要是记录自己 blender eevee 的学习过程。
<br>
这次的研究基于git checkout 到 blender的 ***2017/3/18 Add Stencil test ...*** 的提交上。

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
	DRW_framebuffer_clear(true, true, true, clearcol, 1.0f);

	DRW_draw_pass(psl->depth_pass);
	DRW_draw_pass(psl->depth_pass_cull);
	DRW_draw_pass(psl->pass);

	/* Restore default framebuffer */
	DRW_framebuffer_texture_detach(dtxl->depth);
	DRW_framebuffer_texture_attach(dfbl->default_fb, dtxl->depth, 0);
	DRW_framebuffer_bind(dfbl->default_fb);

	DRW_draw_pass(psl->tonemap);
}

```
以上的渲染过程可以简单理解为 :  ***clear -> draw depth -> draw obj -> tonemap***


DRW_framebuffer_clear(true, true, true, clearcol, 1.0f); 

DRW_draw_pass(psl->depth_pass);  

DRW_draw_pass(psl->depth_pass_cull);  

DRW_draw_pass(psl->pass);  

DRW_draw_pass(psl->tonemap);  

跟 [上一篇](http://shaderstore.cn/2019/12/11/blender-eevee-2017-3-17-commit/) 差不多，要了解的就是 *DRW_draw_pass(psl->depth_pass)* 

# DRW_draw_pass(psl->depth_pass);

通过阅读代码会发现 psl->depth_pass 的使用的shader就是 gpu_shader_3D_vert.glsl, gpu_shader_depth_only_frag.glsl


*vs*
```glsl

uniform mat4 ModelViewProjectionMatrix;
#ifdef USE_NORMALS
uniform mat3 NormalMatrix;
#endif

#if __VERSION__ == 120
  attribute vec3 pos;
#ifdef USE_NORMALS
  attribute vec3 nor;
  varying vec3 normal;
#endif
#else
  in vec3 pos;
#ifdef USE_NORMALS
  in vec3 nor;
  out vec3 normal;
#endif
#endif

void main()
{
#ifdef USE_NORMALS
	normal = normalize(NormalMatrix * nor);
#endif
	gl_Position = ModelViewProjectionMatrix * vec4(pos, 1.0);
}


```


*ps*
```glsl

void main()
{
	// no color output, only depth (line below is implicit)
	// gl_FragDepth = gl_FragCoord.z;
}

```

## 总结
上面的*DRW_draw_pass(psl->depth_pass)*主要是填充深度图，应该是为了在 draw obj 之前先把深度图计算好，draw obj 之后就可以用深度图。