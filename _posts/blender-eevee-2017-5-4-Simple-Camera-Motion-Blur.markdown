---
layout:     post
title:      "blender eevee Simple Camera Motion Blur."
subtitle:   ""
date:       2020-08-10 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> 2017/5/4   * Eevee: Simple Camera Motion Blur.<br>
Disabled by default. Set ENABLE_EFFECT_MOTION_BLUR to 1 to enable.<br>
No fancy motion blur. Use depth and camera matrix to get the motion vectors. <br>
Then blur in this direction.Only available in camera view.<br>
Only Camera animation is supported, does not take into account the parents motion


*在eevee_effects.c中设置一个宏才会有效果* 
```c
define ENABLE_EFFECT_MOTION_BLUR 1
```

## 作用 
简单的运动模糊后处理效果



## 编译

- 直接编译


## 效果展示


![](/img/Eevee/SimpleCameraMotionBlur/1.gif)

## 后处理

目前后期的效果都放在 *eevee_effects.c* 中，有两个后期效果，一个是 motion blur，一个是tonemap。

### motion blur 和 tonemap 渲染关系

最主要的后期渲染在 *eevee_effects.c* 的 EEVEE_draw_effects 函数中
```
void EEVEE_effects_init(EEVEE_Data *vedata)
{
	...

	/* Ping Pong buffer */
	DRWFboTexture tex = {&txl->color_post, DRW_BUF_RGBA_16, DRW_TEX_FILTER};

	const float *viewport_size = DRW_viewport_size_get();
	DRW_framebuffer_init(&fbl->effect_fb,
	                    (int)viewport_size[0], (int)viewport_size[1],
	                    &tex, 1);
	
	...

	effects->final_color = txl->color_post;

	...
}

void EEVEE_effects_cache_init(EEVEE_Data *vedata)
{
	...

	{
		/* Final pass : Map HDR color to LDR color.
		 * Write result to the default color buffer */
		psl->tonemap = DRW_pass_create("Tone Mapping", DRW_STATE_WRITE_COLOR | DRW_STATE_BLEND);

		DRWShadingGroup *grp = DRW_shgroup_create(e_data.tonemap_sh, psl->tonemap);
		DRW_shgroup_uniform_buffer(grp, "hdrColorBuf", &effects->final_color, 0);
		DRW_shgroup_call_add(grp, quad, NULL);
	}
}

void EEVEE_draw_effects(EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	/* Default framebuffer and texture */
	DefaultFramebufferList *dfbl = DRW_viewport_framebuffer_list_get();
	DefaultTextureList *dtxl = DRW_viewport_texture_list_get();

	if ((effects->enabled_effects & EFFECT_MOTION_BLUR) != 0) {
		/* Motion Blur */
		DRW_framebuffer_bind(fbl->effect_fb);
		DRW_draw_pass(psl->motion_blur);
	}

	/* Restore default framebuffer */
	DRW_framebuffer_texture_detach(dtxl->depth);
	DRW_framebuffer_texture_attach(dfbl->default_fb, dtxl->depth, 0, 0);
	DRW_framebuffer_bind(dfbl->default_fb);

	/* Tonemapping */
	DRW_draw_pass(psl->tonemap);
}

```
>
- motion blur 渲染，先绑定 framebuffer fbl->effect_fb, fbl->effect_fb 绑定纹理txl->color_post，渲染fbl->effect_fb 也就是渲染到纹理txl->color_post 上。
- effects->final_color = txl->color_post; 这里就是 effects->final_color保存motion blur渲染好的texture
- DRW_shgroup_uniform_buffer(grp, "hdrColorBuf", &effects->final_color, 0); 最后把 effects->final_color 传入给 tonemap 进行处理

### 理解 motion blur 
```
void EEVEE_effects_init(EEVEE_Data *vedata)
{
	...

	if (!e_data.motion_blur_sh) {
		e_data.motion_blur_sh = DRW_shader_create_fullscreen(datatoc_effect_motion_blur_frag_glsl, NULL);
	}

	...
}
```
>
- motion blur 主要用到的vs是 gpu_shader_fullscreen_vert.glsl  (blender\source\blender\gpu\shaders)
- motion blur 主要用到的ps是 effect_motion_blur_frag.glsl

*gpu_shader_fullscreen_vert.glsl*
```

#if __VERSION__ == 120
	attribute vec2 pos;
	attribute vec2 uvs;
	varying vec4 uvcoordsvar;
#else
	in vec2 pos;
	in vec2 uvs;
	out vec4 uvcoordsvar;
#endif

void main()
{
	uvcoordsvar = vec4(uvs, 0.0, 0.0);
	gl_Position = vec4(pos, 0.0, 1.0);
}

```


*effect_motion_blur_frag.glsl*
```

uniform sampler2D colorBuffer;
uniform sampler2D depthBuffer;

uniform float blurAmount;

/* current frame */
uniform mat4 currInvViewProjMatrix;

/* past frame frame */
uniform mat4 pastViewProjMatrix;

in vec4 uvcoordsvar;

out vec4 FragColor;

#define MAX_SAMPLE 16

float wang_hash_noise(uint s)
{
	uint seed = (uint(gl_FragCoord.x) * 1664525u + uint(gl_FragCoord.y)) + s;

	seed = (seed ^ 61u) ^ (seed >> 16u);
	seed *= 9u;
	seed = seed ^ (seed >> 4u);
	seed *= 0x27d4eb2du;
	seed = seed ^ (seed >> 15u);

	float value = float(seed);
	value *= 1.0 / 4294967296.0;
	return fract(value);
}

void main()
{
	vec3 ndc_pos;
	ndc_pos.xy = uvcoordsvar.xy;
	ndc_pos.z = texture(depthBuffer, uvcoordsvar.xy).x;

	float noise = 2.0 * wang_hash_noise(0u) / MAX_SAMPLE;

	/* Normalize Device Coordinates are [-1, +1]. */
	ndc_pos = ndc_pos * 2.0 - 1.0;

	vec4 p = currInvViewProjMatrix * vec4(ndc_pos, 1.0);
	vec3 world_pos = p.xyz / p.w; /* Perspective divide */

	/* Now find where was this pixel position
	 * inside the past camera viewport */
	vec4 old_ndc = pastViewProjMatrix * vec4(world_pos, 1.0);
	old_ndc.xyz /= old_ndc.w; /* Perspective divide */

	vec2 motion = (ndc_pos.xy - old_ndc.xy) * blurAmount;

	const float inc = 2.0 / MAX_SAMPLE;
	for (float i = -1.0 + noise; i < 1.0; i += inc) {
		FragColor += texture(colorBuffer, uvcoordsvar.xy + motion * i) / MAX_SAMPLE;
	}
}

```
上面比较关键的是如何计算 old_ndc
>
- 先从后期处理UV坐标[0,1] -> [-1,1], 得到ndc坐标
- ndc -> world pos , 可以参考 [NDC To View 记录](http://shaderstore.cn/2020/08/07/NDC-To-View/) 
-  pastViewProjMatrix  保存的是上一帧 camera 的 ViewProject 矩阵，利用 pastViewProjMatrix 变化world pos，透视除法，得到上一帧的 ndc坐标，也就是old_ndc
- vec2 motion = (ndc_pos.xy - old_ndc.xy) * blurAmount; 计算当前帧和上一帧ndc坐标的总偏移
- 利用motion作为uv偏移，计算画面的模糊效果。


### past_world_to_ndc 参数
```
float ctime = BKE_scene_frame_get(scene);
float past_obmat[4][4], future_obmat[4][4], winmat[4][4];

DRW_viewport_matrix_get(winmat, DRW_MAT_WIN);

/* HACK */
Object cam_cpy;
memcpy(&cam_cpy, v3d->camera, sizeof(cam_cpy));

/* Past matrix */
/* FIXME : This is a temporal solution that does not take care of parent animations */
/* Recalc Anim manualy */
BKE_animsys_evaluate_animdata(scene, &cam_cpy.id, cam_cpy.adt, ctime - 1.0, ADT_RECALC_ANIM);
BKE_object_where_is_calc_time(scene, &cam_cpy, ctime - 1.0);

normalize_m4_m4(past_obmat, cam_cpy.obmat);
invert_m4(past_obmat);
mul_m4_m4m4(effects->past_world_to_ndc, winmat, past_obmat);
```
>
- invert_m4(past_obmat); 这个需要逆,由于  
- Pw * Mw = Pv * Mv       
- (P(世界空间) * 矩阵(世界空间) = P(相机空间) * 矩阵(相机空间))
- Pv = Pw * Mw * Mv^(-1)    
- (P(世界空间) * 矩阵(世界空间) * 矩阵(相机空间) 逆  = P(相机空间) )
- 矩阵(相机空间) 其实就是 cam_cpy.obmat
- 矩阵(相机空间)的逆 其实就是 invert_m4(past_obmat);