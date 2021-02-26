---
layout:     post
title:      "blender eevee SSR Output ssr datas to buffers"
subtitle:   ""
date:       2021-2-24 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/17  * Eevee: SSR: Output ssr datas to buffers.<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/SSR/01/1.png)
![](/img/Eevee/SSR/01/2.png)


## 作用
还没有进行SSR效果的编写，这一步先把SSR的渲染框架进行搭建，目前利用后处理输出物体的粗糙度


## 编译
- git 定位到  2017/7/17  * Eevee: SSR: Output ssr datas to buffers.<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 渲染
*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
    ...
    /* Screen Space Reflections */
	EEVEE_effects_do_ssr(sldata, vedata);
    ...
}
```
>
- 在主渲染管线多了EEVEE_effects_do_ssr 对SSR的处理

<br>
<br><br>

*eevee_effects.c*
```
void EEVEE_effects_do_ssr(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_SSR) != 0) {
		DefaultTextureList *dtxl = DRW_viewport_texture_list_get();

		/* Raytrace at halfres. */
		DRW_framebuffer_bind(fbl->screen_tracing_fb);
		DRW_draw_pass(psl->ssr_raytrace);

		/* Resolve at fullres */
		DRW_framebuffer_texture_detach(dtxl->depth);
		DRW_framebuffer_texture_detach(txl->ssr_normal_input);
		DRW_framebuffer_texture_detach(txl->ssr_specrough_input);
		DRW_framebuffer_bind(fbl->main);
		DRW_draw_pass(psl->ssr_resolve);

		/* Restore */
		DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0, 0);
		DRW_framebuffer_texture_attach(fbl->main, txl->ssr_normal_input, 1, 0);
		DRW_framebuffer_texture_attach(fbl->main, txl->ssr_specrough_input, 2, 0);
	}
}
```
>
- 这里可以看到 EEVEE_effects_do_ssr 函数中最主要是两个pass，一个是 psl->ssr_raytrace, 一个是 psl->ssr_resolve 


## 1. ssr_raytrace Pass

### 渲染到RT

- ssr_raytrace Pass 渲染到 fbl->screen_tracing_fb frameBuffer 中, frameBuffer 绑定了两个texture, txl->ssr_hit_output 和 txl->ssr_pdf_output，这两个texture之后会传入到下一个ssr_resolve Pass 使用。

```
DRWFboTexture tex_output[2] = {&txl->ssr_hit_output, (record_two_hit) ? DRW_TEX_RGBA_16 : DRW_TEX_RG_16, 0},
		                               {&txl->ssr_pdf_output, (record_two_hit) ? DRW_TEX_RG_16 : DRW_TEX_R_16, 0};

DRW_framebuffer_init(&fbl->screen_tracing_fb, &draw_engine_eevee_type, tracing_res[0], tracing_res[1], tex_output, 2);
```

<br>

### 初始化
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
    ...
    e_data.ssr_raytrace_sh = DRW_shader_create_fullscreen(datatoc_effect_ssr_frag_glsl, "#define STEP_RAYTRACE\n");
    ...
}

void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
    ...
    psl->ssr_raytrace = DRW_pass_create("Raytrace", DRW_STATE_WRITE_COLOR);
    DRWShadingGroup *grp = DRW_shgroup_create(e_data.ssr_raytrace_sh, psl->ssr_raytrace);
    DRW_shgroup_uniform_buffer(grp, "depthBuffer", &e_data.depth_src);
    DRW_shgroup_uniform_buffer(grp, "normalBuffer", &stl->g_data->minmaxz);
    DRW_shgroup_uniform_buffer(grp, "specRoughBuffer", &stl->g_data->minmaxz);
    DRW_shgroup_call_add(grp, quad, NULL);
    ...
}
```
>
- ssr_raytrace Pass 后处理，由 effect_ssr_frag.glsl 组成，定义了宏 STEP_RAYTRACE

<br>

### Shader
*effect_ssr_frag.glsl*
```
#ifdef STEP_RAYTRACE

uniform sampler2D depthBuffer;
uniform sampler2D normalBuffer;
uniform sampler2D specroughBuffer;

layout(location = 0) out vec4 hitData;
layout(location = 1) out vec4 pdfData;

void main()
{
	hitData = vec4(0.2);
	pdfData = vec4(0.5);
}

#else /* STEP_RESOLVE */
...

#endif
```
>
- Shader很简单，直接 ssr_hit_output RT 输出为 vec4(0.2), ssr_pdf_output RT 输出为 vec4(0.5)

<br>

## 2. ssr_resolve Pass

### 渲染到RT
- ssr_raytrace Pass 渲染到 fbl->main frameBuffer 中

<br>

### 初始化
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
    ...
    e_data.ssr_resolve_sh = DRW_shader_create_fullscreen(datatoc_effect_ssr_frag_glsl, "#define STEP_RESOLVE\n");
    ...
}


void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
    ...
    psl->ssr_resolve = DRW_pass_create("Raytrace", DRW_STATE_WRITE_COLOR);
    grp = DRW_shgroup_create(e_data.ssr_resolve_sh, psl->ssr_resolve);
    DRW_shgroup_uniform_buffer(grp, "depthBuffer", &e_data.depth_src);
    DRW_shgroup_uniform_buffer(grp, "normalBuffer", &txl->ssr_normal_input);
    DRW_shgroup_uniform_buffer(grp, "specroughBuffer", &txl->ssr_specrough_input);
    DRW_shgroup_uniform_buffer(grp, "hitBuffer", &txl->ssr_hit_output);
    DRW_shgroup_uniform_buffer(grp, "pdfBuffer", &txl->ssr_pdf_output);
    DRW_shgroup_call_add(grp, quad, NULL);
    ...
}
```
>
- ssr_resolve Pass 后处理，由 effect_ssr_frag.glsl 组成，定义了宏 STEP_RESOLVE
- hitBuffer 和 pdfBuffer 传入的就是上一步 ssr_raytrace Pass 渲染好的 ssr_hit_output 和 ssr_pdf_output 
- normalBuffer 和 specroughBuffer 传入的是 ssr_normal_input 和 ssr_specrough_input

<br>

### Shader
*effect_ssr_frag.glsl*
```
...

#else /* STEP_RESOLVE */

uniform sampler2D depthBuffer;
uniform sampler2D normalBuffer;
uniform sampler2D specroughBuffer;

uniform sampler2D hitBuffer;
uniform sampler2D pdfBuffer;

out vec4 fragColor;

void main()
{
	ivec2 halfres_texel = ivec2(gl_FragCoord.xy / 2.0);
	ivec2 fullres_texel = ivec2(gl_FragCoord.xy);

	fragColor = vec4(texelFetch(specroughBuffer, fullres_texel, 0).aaa, 1.0);
}

#endif

```
>
- 这里也比较简单，直接采样 specroughBuffer，输出alpha，alpha保存的其实就是物体的粗糙度，pecroughBuffer 也就是 ssr_specrough_input



## 3. ssr_normal_input And ssr_specrough_input
### 初始化
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
    ...
    if (BKE_collection_engine_property_value_get_bool(props, "ssr_enable")) {
		effects->enabled_effects |= EFFECT_SSR;

		int tracing_res[2] = {(int)viewport_size[0] / 2, (int)viewport_size[1] / 2};
		const bool record_two_hit = false;
		const bool high_qual_input = true; /* TODO dither low quality input */

		/* MRT for the shading pass in order to output needed data for the SSR pass. */
		/* TODO create one texture layer per lobe */
		if (txl->ssr_normal_input == NULL) {
			DRWTextureFormat nor_format = DRW_TEX_RG_16;
			txl->ssr_normal_input = DRW_texture_create_2D((int)viewport_size[0], (int)viewport_size[1], nor_format, 0, NULL);
			DRW_framebuffer_texture_attach(fbl->main, txl->ssr_normal_input, 1, 0);
		}

		if (txl->ssr_specrough_input == NULL) {
			DRWTextureFormat specrough_format = (high_qual_input) ? DRW_TEX_RGBA_16 : DRW_TEX_RGBA_8;
			txl->ssr_specrough_input = DRW_texture_create_2D((int)viewport_size[0], (int)viewport_size[1], specrough_format, 0, NULL);
			DRW_framebuffer_texture_attach(fbl->main, txl->ssr_specrough_input, 2, 0);
		}

		/* Raytracing output */
		/* TODO try integer format for hit coord to increase precision */
		DRWFboTexture tex_output[2] = {&txl->ssr_hit_output, (record_two_hit) ? DRW_TEX_RGBA_16 : DRW_TEX_RG_16, 0},
		                               {&txl->ssr_pdf_output, (record_two_hit) ? DRW_TEX_RG_16 : DRW_TEX_R_16, 0};

		DRW_framebuffer_init(&fbl->screen_tracing_fb, &draw_engine_eevee_type, tracing_res[0], tracing_res[1], tex_output, 2);
	}
    ...
}

```
>
- DRW_framebuffer_texture_attach(fbl->main, txl->ssr_normal_input, 1, 0); DRW_framebuffer_texture_attach(fbl->main, txl->ssr_specrough_input, 2, 0);
- txl->ssr_normal_input , txl->ssr_specrough_input  Attach 在 fbl->main 上

<br>

### Shader
*bsdf_common_lib.glsl*
```

/* ------- Structures -------- */
#ifdef VOLUMETRICS

...

#else

struct Closure {
	vec3 radiance;
	float opacity;
	vec4 ssr_data;
	vec2 ssr_normal;
	int ssr_id;
};

#define CLOSURE_DEFAULT Closure(vec3(0.0), 1.0, vec4(0.0), vec2(0.0), -1)

uniform int outputSsrId;

Closure closure_mix(Closure cl1, Closure cl2, float fac)
{
	Closure cl;
	if (cl1.ssr_id == outputSsrId) {
		cl.ssr_data = mix(cl1.ssr_data.xyzw, vec4(vec3(0.0), cl1.ssr_data.w), fac); /* do not blend roughness */
		cl.ssr_normal = cl1.ssr_normal;
		cl.ssr_id = cl1.ssr_id;
	}
	else {
		cl.ssr_data = mix(vec4(vec3(0.0), cl2.ssr_data.w), cl2.ssr_data.xyzw, fac); /* do not blend roughness */
		cl.ssr_normal = cl2.ssr_normal;
		cl.ssr_id = cl2.ssr_id;
	}
	cl.radiance = mix(cl1.radiance, cl2.radiance, fac);
	cl.opacity = mix(cl1.opacity, cl2.opacity, fac);
	return cl;
}

Closure closure_add(Closure cl1, Closure cl2)
{
	Closure cl = (cl1.ssr_id == outputSsrId) ? cl1 : cl2;
	cl.radiance = cl1.radiance + cl2.radiance;
	cl.opacity = cl1.opacity + cl2.opacity;
	return cl;
}

#if defined(MESH_SHADER) && !defined(SHADOW_SHADER)
layout(location = 0) out vec4 fragColor;
layout(location = 1) out vec4 ssrNormals;
layout(location = 2) out vec4 ssrData;

Closure nodetree_exec(void); /* Prototype */

#define NODETREE_EXEC
void main()
{
	Closure cl = nodetree_exec();
	fragColor = vec4(cl.radiance, cl.opacity);
	ssrNormals = cl.ssr_normal.xyyy;
	ssrData = cl.ssr_data;
}

#endif /* MESH_SHADER && !SHADOW_SHADER */

#endif /* VOLUMETRICS */

```

<br>

*default_frag.glsl*
```

uniform vec3 basecol;
uniform float metallic;
uniform float specular;
uniform float roughness;

Closure nodetree_exec(void)
{
	vec3 dielectric = vec3(0.034) * specular * 2.0;
	vec3 diffuse = mix(basecol, vec3(0.0), metallic);
	vec3 f0 = mix(dielectric, basecol, metallic);
	vec3 ssr_spec;
	vec3 radiance = eevee_surface_lit((gl_FrontFacing) ? worldNormal : -worldNormal, diffuse, f0, roughness, 1.0, 0, ssr_spec);

	Closure result = Closure(radiance, 1.0, vec4(ssr_spec, roughness), viewNormal.xy, 0);

#if !defined(USE_ALPHA_BLEND)
	result.opacity = length(viewPosition);
#endif

	return result;
}

```
>
- 这里的 default_frag.glsl 已经没有main函数了，main去到了 bsdf_common_lib.glsl 中了
- 注意 bsdf_common_lib.glsl
```
layout(location = 0) out vec4 fragColor;
layout(location = 1) out vec4 ssrNormals;
layout(location = 2) out vec4 ssrData;
```
这里就对应了 fbl->main frameBuffer 的 Attach 的RT，那就是 输出的 ssrNormals 就渲染到 ssr_normal_input RT，ssrData 就渲染到 ssr_specrough_input RT 上
<br><br><br><br>
- 
```
Closure result = Closure(radiance, 1.0, vec4(ssr_spec, roughness), viewNormal.xy, 0);
...
ssrNormals = cl.ssr_normal.xyyy;
ssrData = cl.ssr_data;
```
- default_frag.glsl 输出的 ssrNormals = viewNormal.xyyy
- default_frag.glsl 输出的 ssrData = vec4(ssr_spec, roughness)
- ssr_spec 在 eevee_surface_lit 函数中进行计算，eevee_surface_lit 在 lit_sufrace_frag.glsl 中进行定义

<br>
<br>

## 4. ssr_spec And ssr_id
*lit_sufrace_frag.glsl*
```
vec3 eevee_surface_lit(vec3 N, vec3 albedo, vec3 f0, float roughness, float ao, int ssr_id, out vec3 ssr_spec)
{
    ...
    /* SSR lobe is applied later in a defered style */
	if (ssr_id != outputSsrId) {
		/* Planar Reflections */
		for (int i = 0; i < MAX_PLANAR && i < planar_count && spec_accum.a < 0.999; ++i) {
			PlanarData pd = planars_data[i];

			float fade = probe_attenuation_planar(pd, worldPosition, N);

			if (fade > 0.0) {
				vec3 spec = probe_evaluate_planar(float(i), pd, worldPosition, N, V, rand.r, cameraPos, roughness, fade);
				accumulate_light(spec, fade, spec_accum);
			}
		}

		/* Specular probes */
		vec3 spec_dir = get_specular_dominant_dir(N, V, roughnessSquared);

		/* Starts at 1 because 0 is world probe */
		for (int i = 1; i < MAX_PROBE && i < probe_count && spec_accum.a < 0.999; ++i) {
			CubeData cd = probes_data[i];

			float fade = probe_attenuation_cube(cd, worldPosition);

			if (fade > 0.0) {
				vec3 spec = probe_evaluate_cube(float(i), cd, worldPosition, spec_dir, roughness);
				accumulate_light(spec, fade, spec_accum);
			}
		}

		/* World Specular */
		if (spec_accum.a < 0.999) {
			vec3 spec = probe_evaluate_world_spec(spec_dir, roughness);
			accumulate_light(spec, 1.0, spec_accum);
		}
	}

	/* Ambient Occlusion */
	vec3 bent_normal;
	float final_ao = occlusion_compute(N, viewPosition, ao, rand.rg, bent_normal);

	/* Get Brdf intensity */
	vec2 uv = lut_coords(dot(N, V), roughness);
	vec2 brdf_lut = texture(utilTex, vec3(uv, 1.0)).rg;

	ssr_spec = F_ibl(f0, brdf_lut) * specular_occlusion(dot(N, V), final_ao, roughness);
	out_light += spec_accum.rgb * ssr_spec * float(specToggle);
    ...
}

```
>
- ssr_spec 这里的计算 ssr_spec = F_ibl(f0, brdf_lut) * specular_occlusion(dot(N, V), final_ao, roughness);<br><br>
- ssr_id 和 outputSsrId 不一样的话， 才计算 Planar Reflections 和 Probe Reflection Cubemap 和 World Probe(环境球)，如果相同的话就不计算<br><br>
- 如果打开了 ssr 开关的话，传入到Shader的 outputSsrId 为 0，如果不打开的话，就是-1 (只考虑实体模型)<br><br>
- ssr_id 是nodetree 节点自身传进去的，看  default_frag.glsl ，传入ssr_id 为 0，如果是nodetree的话，就是节点本身去设置了<br><br>
- 下面是node_bsdf_glossy生成的，默认就是cons5 =  0，也就是ssr_id = 0 <br><br>
```
const float cons5 = float(0.000000000000);
layout (std140) uniform nodeTree {
	vec4 unf2;
	float unf3;
};
Closure nodetree_exec(void)
{
	vec3 tmp1;
	Closure tmp6;
	Closure tmp8;
	world_normals_get(tmp1);
	node_bsdf_glossy(unf2, unf3, tmp1, cons5, tmp6);
	node_output_eevee_material(tmp6, tmp8);
	return tmp8;
}
```
<br><br>
- 也就是说，如果打开了SSR选项，outputSsrId = 0, 然后 节点的 ssd_id 也是 0 的话，outputSsrId和节点ssd_id相同，不会 Planar Reflections 和 Probe Reflection Cubemap 和 World Probe(环境球)。
<br><br>
- 打开 SSR选项，不会计算eevee_surface_lit 的 Planar Reflections 和 Probe Reflection Cubemap 和 World Probe(环境球)。
<br><br>
- 关闭 SSR选项，会计算eevee_surface_lit 的 Planar Reflections 和 Probe Reflection Cubemap 和 World Probe(环境球)。

## 5. ssr_id in NodeTree

```
void node_bsdf_glossy(vec4 color, float roughness, vec3 N, float ssr_id, out Closure result)
{
#ifdef EEVEE_ENGINE
	vec3 ssr_spec;
	vec3 L = eevee_surface_glossy_lit(N, vec3(1.0), roughness, 1.0, int(ssr_id), ssr_spec);
	vec3 vN = mat3(ViewMatrix) * N;
	result = Closure(L * color.rgb, 1.0, vec4(ssr_spec * color.rgb, roughness), vN.xy, int(ssr_id));
#else
	/* ambient light */
	vec3 L = vec3(0.2);

	direction_transform_m4v3(N, ViewMatrix, N);

	/* directional lights */
	for (int i = 0; i < NUM_LIGHTS; i++) {
		vec3 light_position = glLightSource[i].position.xyz;
		vec3 H = glLightSource[i].halfVector.xyz;
		vec3 light_diffuse = glLightSource[i].diffuse.rgb;
		vec3 light_specular = glLightSource[i].specular.rgb;

		/* we mix in some diffuse so low roughness still shows up */
		float bsdf = 0.5 * pow(max(dot(N, H), 0.0), 1.0 / roughness);
		bsdf += 0.5 * max(dot(N, light_position), 0.0);
		L += light_specular * bsdf;
	}

	result = Closure(L * color.rgb, 1.0);
#endif
}
```
<br>
```
static int node_shader_gpu_bsdf_glossy(GPUMaterial *mat, bNode *node, bNodeExecData *UNUSED(execdata), GPUNodeStack *in, GPUNodeStack *out)
{
	if (!in[2].link)
		GPU_link(mat, "world_normals_get", &in[2].link);
	return GPU_stack_link(mat, node, "node_bsdf_glossy", in, out, GPU_uniform(&node->ssr_id));
}
```
>
- node_bsdf_glossy 节点有参数 ssr_id，在node_shader_bsdf_glossy.c 会进行设置 ssr_id