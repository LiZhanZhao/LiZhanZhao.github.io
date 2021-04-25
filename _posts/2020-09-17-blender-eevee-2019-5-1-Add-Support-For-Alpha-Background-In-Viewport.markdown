---
layout:     post
title:      "blender eevee Add support for alpha background in viewport"
subtitle:   ""
date:       2021-4-25 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2019/5/1  *   Eevee : Add support for alpha background in viewport. <br> 

> 
Viewport now displays alpha checkerboard pattern like Cycles does when
film alpha is set to "Transparent".

>
Some small workarounds were necessary for Depth of Field and correct TAA
support.


> SVN : 2019/3/8  *  Windows: llvm debug headers.


<br><br>

## 作用
背景 displays alpha checkerboard


<br><br>

## 效果

![](/img/Eevee/AlphaBackground/01/1.png)

<br><br>


### 渲染

*eevee_engine.c*
```
static void eevee_draw_background(void *vedata)
{
	 /* Tonemapping and transfer result to default framebuffer. */
	bool use_render_settings = stl->g_data->use_color_render_settings;

	GPU_framebuffer_bind(dfbl->default_fb);
	DRW_transform_to_display(stl->effects->final_tx, true, use_render_settings);

	/* Draw checkerboard with alpha under. */
	EEVEE_draw_alpha_checker(vedata);

	...

	/* Draw checkerboard with alpha under. */
 	EEVEE_draw_alpha_checker(vedata);
	...
}
```
>
- 这里最主要的是 EEVEE_draw_alpha_checker 函数

<br><br>

### EEVEE_draw_alpha_checker

#### 初始化
```
void EEVEE_effects_init(EEVEE_ViewLayerData *sldata,
                        EEVEE_Data *vedata,
                        Object *camera,
                        const bool minimal)
{
	...
	/* Alpha checker if background is not drawn in viewport. */
	if (!DRW_state_is_image_render() && !DRW_state_draw_background()) {
		effects->enabled_effects |= EFFECT_ALPHA_CHECKER;
	}
	...
}
```
>
- 只要Film 的 Alpha 选择了 Transparent 才会进行if语句下面的

<br><br>

```

static const GPUShaderStages builtin_shader_stages[GPU_SHADER_BUILTIN_LEN] = {
	[GPU_SHADER_2D_CHECKER] =
	{
		.vert = datatoc_gpu_shader_2D_vert_glsl,
		.frag = datatoc_gpu_shader_checker_frag_glsl,
	},
};

void EEVEE_effects_cache_init(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	...
	if ((effects->enabled_effects & EFFECT_ALPHA_CHECKER) != 0) {
		psl->alpha_checker = DRW_pass_create("Alpha Checker",
											DRW_STATE_WRITE_COLOR | DRW_STATE_BLEND_PREMUL_UNDER);

		GPUShader *checker_sh = GPU_shader_get_builtin_shader(GPU_SHADER_2D_CHECKER);

		DRWShadingGroup *grp = DRW_shgroup_create(checker_sh, psl->alpha_checker);

		copy_v4_fl4(effects->color_checker_dark, 0.15f, 0.15f, 0.15f, 1.0f);
		copy_v4_fl4(effects->color_checker_light, 0.2f, 0.2f, 0.2f, 1.0f);

		DRW_shgroup_uniform_vec4(grp, "color1", effects->color_checker_dark, 1);
		DRW_shgroup_uniform_vec4(grp, "color2", effects->color_checker_light, 1);
		DRW_shgroup_uniform_int_copy(grp, "size", 8);
		DRW_shgroup_call_add(grp, quad, NULL);
	}
	...
}
```
>
- 这里的 checker_sh 的Shader 使用 vs : gpu_shader_2D_vert.glsl , ps : gpu_shader_checker_frag.glsl
<br>
- 注意这里的混合模式 DRW_STATE_BLEND_PREMUL_UNDER, 这个混合模式是 glBlendFunc(GL_ONE_MINUS_DST_ALPHA, GL_ONE);


<br><br>


#### Shader

*gpu_shader_2D_vert.glsl*
```

uniform mat4 ModelViewProjectionMatrix;

#ifdef UV_POS
in vec2 u;
#  define pos u
#else
in vec2 pos;
#endif

void main()
{
  gl_Position = ModelViewProjectionMatrix * vec4(pos, 0.0, 1.0);
}

```

<br><br>

*gpu_shader_checker_frag.glsl*
```

uniform vec4 color1;
uniform vec4 color2;
uniform int size;

out vec4 fragColor;

void main()
{
  vec2 phase = mod(gl_FragCoord.xy, (size * 2));

  if ((phase.x > size && phase.y < size) || (phase.x < size && phase.y > size)) {
    fragColor = color1;
  }
  else {
    fragColor = color2;
  }
}

```


<br><br>


#### Draw

```
void EEVEE_draw_alpha_checker(EEVEE_Data *vedata)
{
  EEVEE_PassList *psl = vedata->psl;
  EEVEE_StorageList *stl = vedata->stl;
  EEVEE_EffectsInfo *effects = stl->effects;

  if ((effects->enabled_effects & EFFECT_ALPHA_CHECKER) != 0) {
    float mat[4][4];
    unit_m4(mat);

    /* Fragile, rely on the fact that GPU_SHADER_2D_CHECKER
     * only use the persmat. */
    DRW_viewport_matrix_override_set(mat, DRW_MAT_PERS);

    DRW_draw_pass(psl->alpha_checker);

    DRW_viewport_matrix_override_unset(DRW_MAT_PERS);
  }
}
```


