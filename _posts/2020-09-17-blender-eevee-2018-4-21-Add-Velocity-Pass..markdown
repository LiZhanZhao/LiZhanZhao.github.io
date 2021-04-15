---
layout:     post
title:      "blender eevee Add Velocity pass"
subtitle:   ""
date:       2021-4-15 21:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2018/4/21  *   Eevee : Add Velocity pass. <br> 

> 
This pass create a velocity buffer which is basically a 2D motion vector
texture. This is not yet used for rendering but will be usefull for motion
blur and temporal reprojection.


> SVN : 2018/3/18  vc14 libs: add missing package folder. 


<br><br>

## 作用
添加 Velocity pass

<br><br>

### 渲染
*eevee_effects.c*
```
void EEVEE_draw_effects(EEVEE_ViewLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	...
	/* First resolve the velocity. */
	if ((effects->enabled_effects & EFFECT_VELOCITY_BUFFER) != 0) {
		DRW_viewport_matrix_get(effects->velocity_curr_persinv, DRW_MAT_PERSINV);

		GPU_framebuffer_bind(fbl->velocity_resolve_fb);
		DRW_draw_pass(psl->velocity_resolve);
	}
	DRW_viewport_matrix_get(effects->velocity_past_persmat, DRW_MAT_PERS);
	...
}
```
>
- 绑定 framebuffer velocity_resolve_fb，把DRW_draw_pass(psl->velocity_resolve) 东西渲染上去

<br><br>

### 渲染前
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata, Object *camera)
{
	...
	/**
	 * Motion vector buffer for correct TAA / motion blur.
	 */
	if ((effects->enabled_effects & EFFECT_VELOCITY_BUFFER) != 0) {
		/* TODO use RG16_UNORM */
		effects->velocity_tx = DRW_texture_pool_query_2D(size_fs[0], size_fs[1], DRW_TEX_RG_32,
		                                                 &draw_engine_eevee_type);

		/* TODO output objects velocity during the mainpass. */
		// GPU_framebuffer_texture_attach(fbl->main_fb, effects->velocity_tx, 1, 0);

		GPU_framebuffer_ensure_config(&fbl->velocity_resolve_fb, {
			GPU_ATTACHMENT_NONE,
			GPU_ATTACHMENT_TEXTURE(effects->velocity_tx)
		});
	}
	else {
		effects->velocity_tx = NULL;
	}
	...
}
```
>
- velocity_resolve_fb 绑定了 RT effects->velocity_tx, 那就是 velocity_resolve Pass 渲染东西到 RT effects->velocity_tx 上

<br><br>



```
static void eevee_create_shader_downsample(void)
{
	...
	char *frag_str = BLI_string_joinN(
	    datatoc_common_uniforms_lib_glsl,
	    datatoc_common_view_lib_glsl,
	    datatoc_bsdf_common_lib_glsl,
	    datatoc_effect_velocity_resolve_frag_glsl);

	e_data.velocity_resolve_sh = DRW_shader_create_fullscreen(frag_str, NULL);
	...
}

void EEVEE_effects_cache_init(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	...
	if ((effects->enabled_effects & EFFECT_VELOCITY_BUFFER) != 0) {
		/* This pass compute camera motions to the non moving objects. */
		psl->velocity_resolve = DRW_pass_create("Velocity Resolve", DRW_STATE_WRITE_COLOR);
		DRWShadingGroup *grp = DRW_shgroup_create(e_data.velocity_resolve_sh, psl->velocity_resolve);
		DRW_shgroup_uniform_texture_ref(grp, "depthBuffer", &e_data.depth_src);
		DRW_shgroup_uniform_block(grp, "common_block", sldata->common_ubo);
		DRW_shgroup_uniform_mat4(grp, "currPersinv", effects->velocity_curr_persinv);
		DRW_shgroup_uniform_mat4(grp, "pastPersmat", effects->velocity_past_persmat);
		DRW_shgroup_call_add(grp, quad, NULL);
	}
	...
}
```
>
- velocity_resolve Pass 由 velocity_resolve_frag.glsl 组成
<br><br>
- 传入shader的数据有 当前帧的VP矩阵的逆，上一帧的VP矩阵， 还有深度图




<br><br>


### Shader
*bsdf_common_lib.glsl*
```
vec3 project_point(mat4 m, vec3 v) {
	vec4 tmp = m * vec4(v, 1.0);
	return tmp.xyz / tmp.w;
}
```

*effect_velocity_resolve_frag.glsl*
```
uniform mat4 currPersinv;
uniform mat4 pastPersmat;

out vec2 outData;

void main()
{
	/* Extract pixel motion vector from camera movement. */
	ivec2 texel = ivec2(gl_FragCoord.xy);
	vec2 uv = gl_FragCoord.xy / vec2(textureSize(depthBuffer, 0).xy);

	float depth = texelFetch(depthBuffer, texel, 0).r;

	vec3 world_position = project_point(currPersinv, vec3(uv, depth) * 2.0 - 1.0);
	vec2 uv_history = project_point(pastPersmat, world_position).xy * 0.5 + 0.5;

	outData = uv - uv_history;
}

```
>
- currPersinv 是当前帧的 VP 矩阵的逆
<br><br>
- pastPersmat 是上一帧的 VP 矩阵
<br><br>
- 思路就是同一世界坐标点，比较这个点在 当前帧 和 上一帧 的屏幕uv坐标，保存差值