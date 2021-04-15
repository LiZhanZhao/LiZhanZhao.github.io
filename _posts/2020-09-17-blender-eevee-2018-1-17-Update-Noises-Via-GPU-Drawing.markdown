---
layout:     post
title:      "blender eevee Update noises (in utilTex) via GPU drawing"
subtitle:   ""
date:       2021-4-15 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2018/1/17  *   Eevee : Perf: Update noises (in utilTex) via GPU drawing. <br> 

> 
This leads to a ~3ms improvement of CPU time during drawing.
This prevent the rendering from being stalled waiting for the texture data to be transfered.

> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
利用GPU 去修改 util_tex 的第3层

<br><br>

### 渲染前

*eevee_engine.c*
```
static void eevee_engine_init(void *ved)
{
	...
	EEVEE_materials_init(stl, fbl);
	...
}
```
<br>

*eevee_materials.c*
```
static void eevee_init_util_texture(void)
{
	const int layers = 3 + 16;
	float (*texels)[4] = MEM_mallocN(sizeof(float[4]) * 64 * 64 * layers, "utils texels");
	float (*texels_layer)[4] = texels;

	/* Copy ltc_mat_ggx into 1st layer */		//<----------------------------- 1st layer
	memcpy(texels_layer, ltc_mat_ggx, sizeof(float[4]) * 64 * 64);
	texels_layer += 64 * 64;

	/* Copy bsdf_split_sum_ggx into 2nd layer red and green channels.			//<----------------------------- 2nd layer
	   Copy ltc_mag_ggx into 2nd layer blue channel. */
	for (int i = 0; i < 64 * 64; i++) {
		texels_layer[i][0] = bsdf_split_sum_ggx[i * 2 + 0];
		texels_layer[i][1] = bsdf_split_sum_ggx[i * 2 + 1];
		texels_layer[i][2] = ltc_mag_ggx[i];
	}
	texels_layer += 64 * 64;

	/* Copy blue noise in 3rd layer  */							//<----------------------------- 3rd layer
	for (int i = 0; i < 64 * 64; i++) {
		texels_layer[i][0] = blue_noise[i][0];
		texels_layer[i][1] = blue_noise[i][1];
		texels_layer[i][2] = cosf(blue_noise[i][1] * 2.0f * M_PI);
		texels_layer[i][3] = sinf(blue_noise[i][1] * 2.0f * M_PI);
	}
	texels_layer += 64 * 64;

	/* Copy Refraction GGX LUT in layer 4 - 20 */			// 4 - 20 layer
	for (int j = 0; j < 16; ++j) {
		for (int i = 0; i < 64 * 64; i++) {
			texels_layer[i][0] = btdf_split_sum_ggx[j * 2][i];
			texels_layer[i][1] = btdf_split_sum_ggx[j * 2][i];
			texels_layer[i][2] = btdf_split_sum_ggx[j * 2][i];
			texels_layer[i][3] = btdf_split_sum_ggx[j * 2][i];
		}
		texels_layer += 64 * 64;
	}

	e_data.util_tex = DRW_texture_create_2D_array(
	        64, 64, layers, DRW_TEX_RGBA_16, DRW_TEX_FILTER | DRW_TEX_WRAP, (float *)texels);

	MEM_freeN(texels);
}

static void eevee_init_noise_texture(void)
{
	e_data.noise_tex = DRW_texture_create_2D(64, 64, DRW_TEX_RGBA_16, 0, (float *)blue_noise);
}

void EEVEE_materials_init(EEVEE_StorageList *stl, EEVEE_FramebufferList *fbl)
{
	...

	e_data.update_noise_sh = DRW_shader_create_fullscreen(
			datatoc_update_noise_frag_glsl, NULL);

	// 初始化 util_tex
	eevee_init_util_texture();

	// 初始化 noise_tex
	eevee_init_noise_texture();

	...

	{
		/* Update noise Framebuffer. */
		if (fbl->update_noise_fb == NULL) {
			fbl->update_noise_fb = DRW_framebuffer_create();
		}
	}
}
```
>
- 在这里可以看到，在渲染前, 先初始化 util_tex 和 noise_tex
<br>
- util_tex 是一个纹理数组，layers 个 64x64的 RT 数组, 上面注释已经标记了每一层的保存的数据情况
<br>
- noise_tex 保存 blue_noise 数据


<br><br>


### 渲染
*eevee_materials.c*
```
void EEVEE_materials_cache_init(EEVEE_Data *vedata)
{
	...
	{
		psl->update_noise_pass = DRW_pass_create("Update Noise Pass", DRW_STATE_WRITE_COLOR);
		DRWShadingGroup *grp = DRW_shgroup_create(e_data.update_noise_sh, psl->update_noise_pass);
		DRW_shgroup_uniform_texture(grp, "blueNoise", e_data.noise_tex);
		DRW_shgroup_uniform_vec3(grp, "offsets", e_data.noise_offsets, 1);
		DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL);
	}

	...
}


void EEVEE_update_noise(EEVEE_PassList *psl, EEVEE_FramebufferList *fbl, double offsets[3])
{
	e_data.noise_offsets[0] = offsets[0];
	e_data.noise_offsets[1] = offsets[1];
	e_data.noise_offsets[2] = offsets[2];

	/* Attach & detach because we don't currently support multiple FB per texture,
	 * and this would be the case for multiple viewport. */
	DRW_framebuffer_texture_layer_attach(fbl->update_noise_fb, e_data.util_tex, 0, 2, 0);
	DRW_framebuffer_bind(fbl->update_noise_fb);
	DRW_draw_pass(psl->update_noise_pass);
	DRW_framebuffer_texture_detach(e_data.util_tex);
}
```

*eevee_engine.c*
```
static void eevee_draw_background(void *vedata)
{
	...
	if (DRW_state_is_image_render()) {
		BLI_halton_3D(primes, offset, stl->effects->taa_current_sample, r);
		/* Set jitter offset */
		EEVEE_update_noise(psl, fbl, r);
	}
	else if ((stl->effects->enabled_effects & EFFECT_TAA) != 0) {
		BLI_halton_3D(primes, offset, stl->effects->taa_current_sample, r);
		/* Set jitter offset */
		EEVEE_update_noise(psl, fbl, r);
	}
	...
}
```
>
- 主要是 EEVEE_update_noise 函数 进行渲染
<br><br>
- framebuffer update_noise_fb 绑定了 RT util_tex，
<br><br>
- 进行渲染 DRW_draw_pass(psl->update_noise_pass) 的时候，绑定了 framebuffer update_noise_fb 
<br><br>
- 也就是 DRW_draw_pass(psl->update_noise_pass) 把东西渲染到 RT util_tex 的第 3 个 layer
<br><br>
- update_noise_pass 使用了update_noise_frag.glsl


<br><br>

### Shader

*update_noise_frag.glsl*
```
uniform sampler2D blueNoise;
uniform vec3 offsets;

out vec4 FragColor;

#define M_2PI 6.28318530717958647692

void main(void)
{
	vec2 blue_noise = texelFetch(blueNoise, ivec2(gl_FragCoord.xy), 0).xy;

	float noise = fract(blue_noise.y + offsets.z);
	FragColor.x = fract(blue_noise.x + offsets.x);
	FragColor.y = fract(blue_noise.y + offsets.y);
	FragColor.z = cos(noise * M_2PI);
	FragColor.w = sin(noise * M_2PI);
}

```
>
- 跟util_tex的初始化第3个layer很类似


