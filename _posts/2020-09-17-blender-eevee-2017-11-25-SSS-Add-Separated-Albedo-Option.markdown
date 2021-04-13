---
layout:     post
title:      "blender eevee SSS Add Separated Albedo Option"
subtitle:   ""
date:       2021-4-13 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/11/22  *   Eevee : SSS: Add separated Albedo option. <br> 

>
This option prevent from automatically blurring the albedo color applied to the SSS.

>
While this is great for preserving details it can bleed more light onto the nearby objects since the blurring will be done on pure "white" irradiance.
This issue is to be tackled in a separate commit.


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
实现 SSS separated Albedo 效果
<br><br>

## 效果
*没有使用 separated Albedo option*
![](/img/Eevee/SSS/03/1.png)
<br><br>
*使用 separated Albedo option*
![](/img/Eevee/SSS/03/2.png)

<br><br>

## 准备
先阅读 [这里](http://shaderstore.cn/2021/04/09/blender-eevee-2017-11-14-Initial-Separable-Subsurface-Scattering-Implementation/)
<br><br>

通过这里，可以知道 sss_resolve_ps 由 effect_subsurface_frag.glsl 组成，定义了宏 #define SECOND_PASS

### sssAlbedo  RT

#### 1.初始化
```
int EEVEE_subsurface_init(EEVEE_ViewLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
    ...
    if (effects->sss_separate_albedo && (txl->sss_albedo == NULL)) {
        txl->sss_albedo = DRW_texture_create_2D((int)viewport_size[0], (int)viewport_size[1],
                                                DRW_TEX_RGB_11_11_10, 0, NULL);
    }
    ...
}

void EEVEE_subsurface_data_render(EEVEE_ViewLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_SSS) != 0) {
		float clear[4] = {0.0f, 0.0f, 0.0f, 0.0f};
		/* Clear sss_data texture only... can this be done in a more clever way? */
		DRW_framebuffer_bind(fbl->sss_clear_fb);
		DRW_framebuffer_clear(true, false, false, clear, 0.0f);


		DRW_framebuffer_texture_detach(txl->sss_data);
		if ((effects->enabled_effects & EFFECT_NORMAL_BUFFER) != 0) {
			DRW_framebuffer_texture_detach(txl->ssr_normal_input);
		}
		if ((effects->enabled_effects & EFFECT_SSR) != 0) {
			DRW_framebuffer_texture_detach(txl->ssr_specrough_input);
		}

		/* Start at slot 1 because slot 0 is txl->color */
		int tex_slot = 1;
		DRW_framebuffer_texture_attach(fbl->main, txl->sss_data, tex_slot++, 0);
		if (effects->sss_separate_albedo) {
			DRW_framebuffer_texture_attach(fbl->main, txl->sss_albedo, tex_slot++, 0);
		}
		if ((effects->enabled_effects & EFFECT_NORMAL_BUFFER) != 0) {
			DRW_framebuffer_texture_attach(fbl->main, txl->ssr_normal_input, tex_slot++, 0);
		}
		if ((effects->enabled_effects & EFFECT_SSR) != 0) {
			DRW_framebuffer_texture_attach(fbl->main, txl->ssr_specrough_input, tex_slot++, 0);
		}
		DRW_framebuffer_bind(fbl->main);

		DRW_draw_pass(psl->sss_pass);

		/* Restore */
		DRW_framebuffer_texture_detach(txl->sss_data);
		if (effects->sss_separate_albedo) {
			DRW_framebuffer_texture_detach(txl->sss_albedo);
		}
		if ((effects->enabled_effects & EFFECT_NORMAL_BUFFER) != 0) {
			DRW_framebuffer_texture_detach(txl->ssr_normal_input);
		}
		if ((effects->enabled_effects & EFFECT_SSR) != 0) {
			DRW_framebuffer_texture_detach(txl->ssr_specrough_input);
		}

		DRW_framebuffer_texture_attach(fbl->sss_clear_fb, txl->sss_data, 0, 0);
		if ((effects->enabled_effects & EFFECT_NORMAL_BUFFER) != 0) {
			DRW_framebuffer_texture_attach(fbl->main, txl->ssr_normal_input, 1, 0);
		}
		if ((effects->enabled_effects & EFFECT_SSR) != 0) {
			DRW_framebuffer_texture_attach(fbl->main, txl->ssr_specrough_input, 2, 0);
		}
	}
}
```
>
- 这里可以知道，在 DRW_draw_pass(psl->sss_pass)之前，如果 sss_separate_albedo 打开了，会进行绑定 txl->sss_albedo 到 fbl->main 上，然后再进行 psl->sss_pass 渲染，把东西渲染到 txl->sss_albedo 中
<br><br>


#### 2.渲染

*bsdf_common_lib.glsl*
```
layout(location = 0) out vec4 fragColor;
#ifdef USE_SSS
#ifdef USE_SSS_ALBEDO
layout(location = 1) out vec4 sssData;
layout(location = 2) out vec4 sssAlbedo;
layout(location = 3) out vec4 ssrNormals;
layout(location = 4) out vec4 ssrData;
#else
layout(location = 1) out vec4 sssData;
layout(location = 2) out vec4 ssrNormals;
layout(location = 3) out vec4 ssrData;
#endif /* USE_SSS_ALBEDO */
#else
layout(location = 1) out vec4 ssrNormals;
layout(location = 2) out vec4 ssrData;
#endif /* USE_SSS */

```
>
- 这里输出 sssAlbedo

<br><br>

*gpu_shader_material.glsl*
```
void node_subsurface_scattering(
        vec4 color, float scale, vec3 radius, float sharpen, float texture_blur, vec3 N, float sss_id,
        out Closure result)
{
#if defined(EEVEE_ENGINE) && defined(USE_SSS)
	vec3 out_diff, out_trans;
	vec3 vN = normalize(mat3(ViewMatrix) * N);
	result = CLOSURE_DEFAULT;
	result.ssr_data = vec4(0.0);
	result.ssr_normal = normal_encode(vN, viewCameraVec);
	result.ssr_id = -1;
	result.sss_data.a = scale;
	eevee_closure_subsurface(N, color.rgb, 1.0, scale, out_diff, out_trans);
	result.sss_data.rgb = out_diff + out_trans;
#ifdef USE_SSS_ALBEDO
	/* Not perfect for texture_blur not exaclty equal to 0.0 or 1.0. */
	result.sss_albedo.rgb = mix(color.rgb, vec3(1.0), texture_blur);
	result.sss_data.rgb *= mix(vec3(1.0), color.rgb, texture_blur);
#else
	result.sss_data.rgb *= color.rgb;
#endif
#else
	node_bsdf_diffuse(color, 0.0, N, result);
#endif
}
```
>
- 这里可以看到 如果nodetree 使用了 subsurface scattering 的话, USE_SSS_ALBEDO 定义了的话，就会记录 sss_albedo
<br><br>
- 是否定义宏 USE_SSS_ALBEDO ，主要是 打开了 UI 界面 separated Albedo 的开关，就会定义 USE_SSS_ALBEDO 宏

<br><br>
#### 3.应用
*eevee_subsurface.c*
```
static void eevee_create_shader_subsurface(void)
{
	e_data.sss_sh[0] = DRW_shader_create_fullscreen(datatoc_effect_subsurface_frag_glsl, "#define FIRST_PASS\n");
	e_data.sss_sh[1] = DRW_shader_create_fullscreen(datatoc_effect_subsurface_frag_glsl, "#define SECOND_PASS\n");
	e_data.sss_sh[2] = DRW_shader_create_fullscreen(datatoc_effect_subsurface_frag_glsl, "#define SECOND_PASS\n"
	                                                                                     "#define USE_SEP_ALBEDO\n");
}

void EEVEE_subsurface_add_pass(EEVEE_Data *vedata, unsigned int sss_id, struct GPUUniformBuffer *sss_profile)
{
    ...
    struct GPUShader *sh = (effects->sss_separate_albedo) ? e_data.sss_sh[2] : e_data.sss_sh[1];
	grp = DRW_shgroup_create(sh, psl->sss_resolve_ps);
	DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)vedata->stl->g_data->viewvecs, 2);
	DRW_shgroup_uniform_texture(grp, "utilTex", EEVEE_materials_get_util_tex());
	DRW_shgroup_uniform_buffer(grp, "depthBuffer", &dtxl->depth);
	DRW_shgroup_uniform_buffer(grp, "sssData", &txl->sss_blur);
	DRW_shgroup_uniform_block(grp, "sssProfile", sss_profile);
	DRW_shgroup_uniform_float(grp, "jitterThreshold", &effects->sss_jitter_threshold, 1);
	DRW_shgroup_stencil_mask(grp, sss_id);
	DRW_shgroup_call_add(grp, quad, NULL);

	if (effects->sss_separate_albedo) {
		DRW_shgroup_uniform_buffer(grp, "sssAlbedo", &txl->sss_albedo);
	}
    ...
}
```
>
- psl->sss_resolve_ps 在渲染之前，会根据 sss_separate_albedo 选项 来判断是否使用 e_data.sss_sh[2] 还是 e_data.sss_sh[1] 
<br><br>
- e_data.sss_sh[2] 和 e_data.sss_sh[1] 的差别就是 多了一个 USE_SEP_ALBEDO 的宏
<br><br>
- 如果 sss_separate_albedo打开的话，就会传入 RT sss_albedo 到 psl->sss_resolve_ps 中, SECOND_PASS + USE_SEP_ALBEDO

<br><br>

### sss_resolve_ps Pass

*effect_subsurface_frag.glsl*
```
/* Based on Separable SSS. by Jorge Jimenez and Diego Gutierrez */

#define MAX_SSS_SAMPLES 65
layout(std140) uniform sssProfile {
	vec4 kernel[MAX_SSS_SAMPLES];
	vec4 radii_max_radius;
	int sss_samples;
};

uniform float jitterThreshold;
uniform sampler2D depthBuffer;
uniform sampler2D sssData;
uniform sampler2D sssAlbedo;
uniform sampler2DArray utilTex;

out vec4 FragColor;

uniform mat4 ProjectionMatrix;
uniform vec4 viewvecs[2];

float get_view_z_from_depth(float depth)
{
	if (ProjectionMatrix[3][3] == 0.0) {
		float d = 2.0 * depth - 1.0;
		return -ProjectionMatrix[3][2] / (d + ProjectionMatrix[2][2]);
	}
	else {
		return viewvecs[0].z + depth * viewvecs[1].z;
	}
}

#define LUT_SIZE 64
#define M_PI_2     1.5707963267948966        /* pi/2 */
#define M_2PI      6.2831853071795865        /* 2*pi */

void main(void)
{
	vec2 pixel_size = 1.0 / vec2(textureSize(depthBuffer, 0).xy); /* TODO precompute */
	vec2 uvs = gl_FragCoord.xy * pixel_size;
	vec4 sss_data = texture(sssData, uvs).rgba;
	float depth_view = get_view_z_from_depth(texture(depthBuffer, uvs).r);

	float rand = texelFetch(utilTex, ivec3(ivec2(gl_FragCoord.xy) % LUT_SIZE, 2), 0).r;
#ifdef FIRST_PASS
	float angle = M_2PI * rand + M_PI_2;
	vec2 dir = vec2(1.0, 0.0);
#else /* SECOND_PASS */
	float angle = M_2PI * rand;
	vec2 dir = vec2(0.0, 1.0);
#endif
	vec2 dir_rand = vec2(cos(angle), sin(angle));

	/* Compute kernel bounds in 2D. */
	float homcoord = ProjectionMatrix[2][3] * depth_view + ProjectionMatrix[3][3];
	vec2 scale = vec2(ProjectionMatrix[0][0], ProjectionMatrix[1][1]) * sss_data.aa / homcoord;
	vec2 finalStep = scale * radii_max_radius.w;
	finalStep *= 0.5; /* samples range -1..1 */

	/* Center sample */
	vec3 accum = sss_data.rgb * kernel[0].rgb;

	for (int i = 1; i < sss_samples && i < MAX_SSS_SAMPLES; i++) {
		vec2 sample_uv = uvs + kernel[i].a * finalStep * ((abs(kernel[i].a) > jitterThreshold) ? dir : dir_rand);
		vec3 color = texture(sssData, sample_uv).rgb;
		float sample_depth = texture(depthBuffer, sample_uv).r;
		sample_depth = get_view_z_from_depth(sample_depth);

		/* Depth correction factor. */
		float depth_delta = depth_view - sample_depth;
		float s = clamp(1.0 - exp(-(depth_delta * depth_delta) / (2.0 * sss_data.a)), 0.0, 1.0);

		/* Out of view samples. */
		if (any(lessThan(sample_uv, vec2(0.0))) || any(greaterThan(sample_uv, vec2(1.0)))) {
			s = 1.0;
		}

		accum += kernel[i].rgb * mix(color, sss_data.rgb, s);
	}

#ifdef FIRST_PASS
	FragColor = vec4(accum, sss_data.a);
#else /* SECOND_PASS */
	#ifdef USE_SEP_ALBEDO
	FragColor = vec4(accum * texture(sssAlbedo, uvs).rgb, 1.0);
	#else
	FragColor = vec4(accum, 1.0);
	#endif
#endif
}


```
>
- 这里最主要是 在 SECOND_PASS 宏中，判断了 USE_SEP_ALBEDO 的开关，如果 使用了 USE_SEP_ALBEDO 的话，就 需要采样 texture(sssAlbedo, uvs)



