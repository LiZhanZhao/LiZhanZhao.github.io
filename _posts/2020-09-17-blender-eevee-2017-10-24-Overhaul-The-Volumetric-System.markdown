---
layout:     post
title:      "blender eevee Overhaul The Volumetric System"
subtitle:   ""
date:       2021-4-7 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/10/24  *   Eevee : Overhaul the volumetric system. <br> 

> The system now uses several 3D textures in order to decouple every steps of the volumetric rendering.

> See https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite for more details.

> On the technical side, instead of using a compute shader to populate the 3D textures we use layered rendering with a geometry shader to render 1 fullscreen triangle per 3D texture slice.


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用

重构 volumetric system
<br><br>

## 效果
*想要看到效果, Viewport Samples(TAA Sample) 要先设置为 1*
![](/img/Eevee/Volumetrics+/01/1.png)

<br><br>

### 渲染入口

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Volumetrics */
	DRW_stats_group_start("Volumetrics");
	EEVEE_effects_do_volumetrics(sldata, vedata);
	DRW_stats_group_end();
	...
}
```


<br>


### EEVEE_effects_do_volumetrics
*eevee_effects.c*
```
void EEVEE_effects_do_volumetrics(EEVEE_SceneLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_EffectsInfo *effects = stl->effects;

	if ((effects->enabled_effects & EFFECT_VOLUMETRIC) != 0) {
		DefaultTextureList *dtxl = DRW_viewport_texture_list_get();

		e_data.color_src = txl->color;
		e_data.depth_src = dtxl->depth;

		/* Step 1: Participating Media Properties */
		DRW_framebuffer_bind(fbl->volumetric_fb);
		DRW_draw_pass(psl->volumetric_ps);

		/* Step 2: Scatter Light */
		DRW_framebuffer_bind(fbl->volumetric_scat_fb);
		DRW_draw_pass(psl->volumetric_scatter_ps);

		/* Step 3: Integration */
		DRW_framebuffer_bind(fbl->volumetric_integ_fb);
		DRW_draw_pass(psl->volumetric_integration_ps);

		/* Step 4: Apply for opaque */
		DRW_framebuffer_bind(fbl->effect_fb);
		DRW_draw_pass(psl->volumetric_resolve_ps);

		/* Swap volume history buffers */
		SWAP(struct GPUFrameBuffer *, fbl->volumetric_scat_fb, fbl->volumetric_integ_fb);
		SWAP(GPUTexture *, txl->volume_scatter, txl->volume_scatter_history);
		SWAP(GPUTexture *, txl->volume_transmittance, txl->volume_transmittance_history);

		/* Swap the buffers and rebind depth to the current buffer */
		DRW_framebuffer_texture_detach(dtxl->depth);
		SWAP(struct GPUFrameBuffer *, fbl->main, fbl->effect_fb);
		SWAP(GPUTexture *, txl->color, txl->color_post);
		DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0, 0);
	}
}
```
<br>


### Step 1: Participating Media Properties

#### 1. volumetric_fb 初始化
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	txl->volume_prop_scattering = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
	txl->volume_prop_extinction = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
	txl->volume_prop_emission = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
	txl->volume_prop_phase = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RG_16, DRW_TEX_FILTER, NULL);
	...
	/* Framebuffer setup */
	DRWFboTexture tex_vol[4] = {&txl->volume_prop_scattering, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER},
								{&txl->volume_prop_extinction, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER},
								{&txl->volume_prop_emission, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER},
								{&txl->volume_prop_phase, DRW_TEX_RG_16, DRW_TEX_FILTER};

	DRW_framebuffer_init(&fbl->volumetric_fb, &draw_engine_eevee_type,
							(int)froxel_tex_size[0], (int)froxel_tex_size[1],
							tex_vol, 4);
	...
}
```
>
- 这里需要留意的是, volume_prop_scattering + volume_prop_extinction + volume_prop_emission + volume_prop_phase 都是 3D Texture
<br><br>
- Shader是把东西渲染到 3D Texture 
<br>

#### 2. volumetric_ps 初始化


*eevee_materials.c*
```
void EEVEE_materials_init(EEVEE_StorageList *stl)
{
	...
	ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, datatoc_bsdf_common_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_ambient_occlusion_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_octahedron_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_irradiance_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_lightprobe_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_ltc_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_bsdf_direct_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_lamps_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_volumetric_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_volumetric_frag_glsl);
	e_data.volume_shader_lib = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);
	...
}

static char *eevee_get_volume_defines(int UNUSED(options))
{
	char *str = NULL;

	DynStr *ds = BLI_dynstr_new();
	BLI_dynstr_appendf(ds, SHADER_DEFINES);
	BLI_dynstr_appendf(ds, "#define VOLUMETRICS\n");

	str = BLI_dynstr_get_cstring(ds);
	BLI_dynstr_free(ds);

	return str;
}


struct GPUMaterial *EEVEE_material_world_volume_get(struct Scene *scene, World *wo)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	int options = VAR_WORLD_VOLUME;

	GPUMaterial *mat = GPU_material_from_nodetree_find(&wo->gpumaterial, engine, options);
	if (mat != NULL) {
		return mat;
	}

	char *defines = eevee_get_volume_defines(options);

	mat = GPU_material_from_nodetree(
	        scene, wo->nodetree, &wo->gpumaterial, engine, options,
	        datatoc_volumetric_vert_glsl, datatoc_volumetric_geom_glsl, e_data.volume_shader_lib,
	        defines);

	MEM_freeN(defines);

	return mat;
}

```
<br>


*eevee_effects.c*
```
void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	/* World pass is not additive as it also clear the buffer. */
	psl->volumetric_ps = DRW_pass_create("Volumetric Properties", DRW_STATE_WRITE_COLOR);

	/* World Volumetric */
	struct GPUMaterial *mat = EEVEE_material_world_volume_get(scene, wo);

	grp = DRW_shgroup_material_empty_tri_batch_create(mat, psl->volumetric_ps, volumetrics->froxel_tex_size[2]);

	if (grp) {
		DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)stl->g_data->viewvecs, 2);
		DRW_shgroup_uniform_ivec3(grp, "volumeTextureSize", (int *)volumetrics->froxel_tex_size, 1);
		DRW_shgroup_uniform_vec2(grp, "volume_uv_ratio", (float *)volumetrics->volume_coord_scale, 1);
		DRW_shgroup_uniform_vec3(grp, "volume_param", (float *)volumetrics->depth_param, 1);
		DRW_shgroup_uniform_vec3(grp, "volume_jitter", (float *)volumetrics->jitter, 1);
	}
	...
}
```
>
- 可以看到，这里 volumetric_ps 使用了 node tree, vs 使用了 volumetric_vert.glsl, gs 使用了 volumetric_geom.glsl，ps 使用了 volumetric_lib.glsl + volumetric_frag.glsl
<br><br>
- Shader 中 定义了 #define VOLUMETRICS
<br><br>
- 值得留意得是，这里渲染是使用了 DRW_shgroup_material_empty_tri_batch_create 函数进行得，经过截帧可以知道 DRW_shgroup_material_empty_tri_batch_create 和构造一个 Geom，这个 Geom 是由volumetrics->froxel_tex_size[2] 个三角形， volumetrics->froxel_tex_size[2] x 3 个顶点组成，例如 volumetrics->froxel_tex_size[2] 是 64 的话，就是 64 个三角形， Geom 就是 64 x 3 = 192 个顶点，但是这些三角形的pos都是没有任何数据的，例如
![](/img/Eevee/Volumetrics+/01/2.png)
<br><br>
- DRW_shgroup_material_empty_tri_batch_create 也就是说 会把东西渲染对应的 3D Texture 上的每一个Texture上。(3D Texture 可以理解为 Texture2D Array)

<br><br>

#### 3. volumetric_ps Shader
*volumetric_vert.glsl*
```
out vec4 vPos;

void main()
{
	/* Generate Triangle : less memory fetches from a VBO */
	int v_id = gl_VertexID % 3; /* Vertex Id */
	int t_id = gl_VertexID / 3; /* Triangle Id */

	/* Crappy diagram
	 * ex 1
	 *    | \
	 *    |   \
	 *  1 |     \
	 *    |       \
	 *    |         \
	 *  0 |           \
	 *    |             \
	 *    |               \
	 * -1 0 --------------- 2
	 *   -1     0     1     ex
	 **/
	vPos.x = float(v_id / 2) * 4.0 - 1.0; /* int divisor round down */
	vPos.y = float(v_id % 2) * 4.0 - 1.0;
	vPos.z = float(t_id);
	vPos.w = 1.0;
}

```
>
- v_id 的取值范围是 0,1,2
<br><br>

> 当 v_id = 0 的时候
- vPos.x = float(0 / 2) * 4.0 - 1.0 = -1
- vPos.y = float(0 % 2) * 4.0 - 1.0 = -1
<br><br>

> 当 v_id = 1 的时候
- vPos.x = float(1 / 2) * 4.0 - 1.0 = -1		因为 float(1/2) = 0, 1/2 是int ，所以是 0
- vPos.y = float(1 % 2) * 4.0 - 1.0 = 3
<br><br>

> 当 v_id = 2 的时候
- vPos.x = float(2 / 2) * 4.0 - 1.0 = 3
- vPos.y = float(2 % 2) * 4.0 - 1.0 = -1
<br><br>


>
- 参考 [Rendering a Screen Covering Triangle in OpenGL (with no buffers)](https://rauwendaal.net/2014/06/14/rendering-a-screen-covering-triangle-in-opengl/)
- ![](/img/Eevee/Volumetrics+/01/3.png)


<br><br>
<br><br>

*volumetric_geom.glsl*
```

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

in vec4 vPos[];

flat out int slice;

/* This is just a pass-through geometry shader that send the geometry
 * to the layer corresponding to it's depth. */

void main() {
	gl_Layer = slice = int(vPos[0].z);

	gl_Position = vPos[0].xyww;
	EmitVertex();

	gl_Position = vPos[1].xyww;
	EmitVertex();

	gl_Position = vPos[2].xyww;
	EmitVertex();

	EndPrimitive();
}
```
>
- gl_Layer = slice = int(vPos[0].z); 这里的意思就是，把东西渲染到 3D Texture 的对应的 第几个 Texture 中
<br><br>
- 因为 vs 中设置了 vPos.z = float(t_id); t_id 就是对应的第几个三角形
<br><br>
- slice = int(vPos[0].z); 在PS中也用到了 slice，对应的是第几个 Texture


<br><br>


*volumetric_lib.glsl*
```

/* Based on Frosbite Unified Volumetric.
 * https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite */

uniform float volume_light_clamp;

uniform vec3 volume_param; /* Parameters to the volume Z equation */

uniform vec2 volume_uv_ratio; /* To convert volume uvs to screen uvs */

/* Volume slice to view space depth. */
float volume_z_to_view_z(float z)
{
	// Perspective Projection

	if (ProjectionMatrix[3][3] == 0.0) {
		/* Exponential distribution */
		return (exp2(z / volume_param.z) - volume_param.x) / volume_param.y;
	}
	else {
		/* Linear distribution */
		return mix(volume_param.x, volume_param.y, z);
	}
}

float view_z_to_volume_z(float depth)
{
	// Perspective Projection

	if (ProjectionMatrix[3][3] == 0.0) {
		/* Exponential distribution */
		return volume_param.z * log2(depth * volume_param.y + volume_param.x);
	}
	else {
		/* Linear distribution */
		return (depth - volume_param.x) * volume_param.z;
	}
}

/* Volume texture normalized coordinates to NDC (special range [0, 1]). */
vec3 volume_to_ndc(vec3 cos)
{
	cos.z = volume_z_to_view_z(cos.z);
	cos.z = get_depth_from_view_z(cos.z);
	cos.xy /= volume_uv_ratio;
	return cos;
}

```
>
- volume_param 参数在下面会看到是怎么计算的

<br><br>

*volumetric_frag.glsl*
```
/* Based on Frosbite Unified Volumetric.
 * https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite */

#define NODETREE_EXEC

uniform ivec3 volumeTextureSize;
uniform vec3 volume_jitter;

flat in int slice;

/* Warning: theses are not attributes, theses are global vars. */
vec3 worldPosition = vec3(0.0);
vec3 viewPosition = vec3(0.0);
vec3 viewNormal = vec3(0.0);

layout(location = 0) out vec4 volumeScattering;
layout(location = 1) out vec4 volumeExtinction;
layout(location = 2) out vec4 volumeEmissive;
layout(location = 3) out vec4 volumePhase;

/* Store volumetric properties into the froxel textures. */

void main()
{
	ivec3 volume_cell = ivec3(gl_FragCoord.xy, slice);
	
	vec3 ndc_cell = volume_to_ndc((vec3(volume_cell) + volume_jitter) / volumeTextureSize);

	viewPosition = get_view_space_from_depth(ndc_cell.xy, ndc_cell.z);
	worldPosition = transform_point(ViewMatrixInverse, viewPosition);

	Closure cl = nodetree_exec();

	volumeScattering = vec4(cl.scatter, 1.0);
	volumeExtinction = vec4(max(vec3(1e-4), cl.absorption + cl.scatter), 1.0);
	volumeEmissive = vec4(cl.emission, 1.0);
	volumePhase = vec4(cl.anisotropy, vec3(1.0));
}

```
>
- volume_to_ndc 的作用就是 把 Volume texture normalized coordinates to NDC ，就是 Texture coordinates 变成 [0, 1]

>
- 看到 这里  Closure cl = nodetree_exec(), 可以看到是 nodetree 代码生成的，nodetree 的源码在 *source/blender/gpu/shaders/gpu_shader_materials.glsl* 中

>
- 例如 

```
/* volumes */

void node_volume_scatter(vec4 color, float density, float anisotropy, out Closure result)
{
#ifdef VOLUMETRICS
	result = Closure(vec3(0.0), color.rgb * density, vec3(0.0), anisotropy);
#else
	result = CLOSURE_DEFAULT;
#endif
}

void node_volume_absorption(vec4 color, float density, out Closure result)
{
#ifdef VOLUMETRICS
	result = Closure((1.0 - color.rgb) * density, vec3(0.0), vec3(0.0), 0.0);
#else
	result = CLOSURE_DEFAULT;
#endif
}
```
<br><br>


#### 4.Shader 参数
*eevee_effects.c*
```
/* Find Froxel Texture resolution. */
int froxel_tex_size[3];

froxel_tex_size[0] = (int)ceilf(fmaxf(1.0f, viewport_size[0] / (float)tile_size));
froxel_tex_size[1] = (int)ceilf(fmaxf(1.0f, viewport_size[1] / (float)tile_size));
froxel_tex_size[2] = max_ii(BKE_collection_engine_property_value_get_int(props, "volumetric_samples"), 1);

volumetrics->volume_coord_scale[0] = viewport_size[0] / (float)(tile_size * froxel_tex_size[0]);
volumetrics->volume_coord_scale[1] = viewport_size[1] / (float)(tile_size * froxel_tex_size[1]);

...

if (DRW_viewport_is_persp_get()) {
	const float clip_start = stl->g_data->viewvecs[0][2];
	/* Negate */
	float near = volumetrics->integration_start = min_ff(-volumetrics->integration_start, clip_start - 1e-4f);
	float far = volumetrics->integration_end = min_ff(-volumetrics->integration_end, near - 1e-4f);

	volumetrics->depth_param[0] = (far - near * exp2(1.0f / volumetrics->sample_distribution)) / (far - near);
	volumetrics->depth_param[1] = (1.0f - volumetrics->depth_param[0]) / near;
	volumetrics->depth_param[2] = volumetrics->sample_distribution;
}
else {
	const float clip_start = stl->g_data->viewvecs[0][2];
	const float clip_end = stl->g_data->viewvecs[1][2];
	volumetrics->integration_start = min_ff(volumetrics->integration_end, clip_start);
	volumetrics->integration_end = max_ff(-volumetrics->integration_end, clip_end);

	volumetrics->depth_param[0] = volumetrics->integration_start;
	volumetrics->depth_param[1] = volumetrics->integration_end;
	volumetrics->depth_param[2] = 1.0f / (volumetrics->integration_end - volumetrics->integration_start);
}
```
>
- 这里可以看到 volume_param 和 volume_coord_scale 参数是怎么计算的

<br><br>

### Step 2: Scatter Light

#### 1. volumetric_scat_fb 初始化
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...

	/* Volume scattering: We compute for each froxel the
	* Scattered light towards the view. We also resolve temporal
	* super sampling during this stage. */
	txl->volume_scatter = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
	txl->volume_transmittance = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
	...

	...
	DRWFboTexture tex_vol_scat[2] = {&txl->volume_scatter, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER},
		                                 {&txl->volume_transmittance, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER};

	DRW_framebuffer_init(&fbl->volumetric_scat_fb, &draw_engine_eevee_type,
							(int)froxel_tex_size[0], (int)froxel_tex_size[1],
							tex_vol_scat, 2);
	...
}
```
>
- volume_scatter + volume_transmittance 是 3D Texture

<br>

#### 2. volumetric_scatter_ps 初始化
*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, datatoc_bsdf_common_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_bsdf_direct_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_irradiance_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_octahedron_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_lamps_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_volumetric_lib_glsl);
	e_data.volumetric_common_lamps_lib = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	e_data.volumetric_scatter_sh = DRW_shader_create_with_lib(datatoc_volumetric_vert_glsl,
		                                                          datatoc_volumetric_geom_glsl,
		                                                          datatoc_volumetric_scatter_frag_glsl,
	                                                              e_data.volumetric_common_lamps_lib,
	                                                              SHADER_DEFINES
	                                                              "#define VOLUMETRICS\n"
	                                                              "#define VOLUME_SHADOW\n");
	...
}


void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	psl->volumetric_scatter_ps = DRW_pass_create("Volumetric Scattering", DRW_STATE_WRITE_COLOR);
	grp = DRW_shgroup_empty_tri_batch_create(e_data.volumetric_scatter_sh, psl->volumetric_scatter_ps, volumetrics->froxel_tex_size[2]);
	DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)stl->g_data->viewvecs, 2);
	DRW_shgroup_uniform_vec2(grp, "volume_uv_ratio", (float *)volumetrics->volume_coord_scale, 1);
	DRW_shgroup_uniform_vec3(grp, "volume_param", (float *)volumetrics->depth_param, 1);
	DRW_shgroup_uniform_vec3(grp, "volume_jitter", (float *)volumetrics->jitter, 1);
	DRW_shgroup_uniform_mat4(grp, "PastViewProjectionMatrix", (float *)stl->g_data->prev_persmat);
	DRW_shgroup_uniform_block(grp, "light_block", sldata->light_ubo);
	DRW_shgroup_uniform_block(grp, "shadow_block", sldata->shadow_ubo);
	DRW_shgroup_uniform_block(grp, "grid_block", sldata->grid_ubo);
	DRW_shgroup_uniform_int(grp, "light_count", (volumetrics->use_lights) ? &sldata->lamps->num_light : &zero, 1);
	DRW_shgroup_uniform_int(grp, "grid_count", &sldata->probes->num_render_grid, 1);
	DRW_shgroup_uniform_buffer(grp, "irradianceGrid", &sldata->irradiance_pool);
	DRW_shgroup_uniform_buffer(grp, "shadowTexture", &sldata->shadow_pool);
	DRW_shgroup_uniform_float(grp, "volume_light_clamp", &volumetrics->light_clamp, 1);
	DRW_shgroup_uniform_float(grp, "volume_shadows_steps", &volumetrics->shadow_step_count, 1);
	DRW_shgroup_uniform_float(grp, "volume_history_alpha", &volumetrics->history_alpha, 1);
	DRW_shgroup_uniform_buffer(grp, "volumeScattering", &txl->volume_prop_scattering);
	DRW_shgroup_uniform_buffer(grp, "volumeExtinction", &txl->volume_prop_extinction);
	DRW_shgroup_uniform_buffer(grp, "volumeEmission", &txl->volume_prop_emission);
	DRW_shgroup_uniform_buffer(grp, "volumePhase", &txl->volume_prop_phase);
	DRW_shgroup_uniform_buffer(grp, "historyScattering", &txl->volume_scatter_history);
	DRW_shgroup_uniform_buffer(grp, "historyTransmittance", &txl->volume_transmittance_history);
	...
}
```
>
- vs 使用了 volumetric_vert.glsl, gs 使用了 volumetric_geom.glsl，ps 使用了 volumetric_scatter_frag.glsl + volumetric_lib.glsl
<br><br>
- 这里使用了 DRW_shgroup_empty_tri_batch_create,也是先把东西渲染到 3D Texture 上

<br><br>

#### 3. volumetric_scatter_ps Shader

*volumetric_lib.glsl*
```

/* Based on Frosbite Unified Volumetric.
 * https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite */

uniform float volume_light_clamp;

uniform vec3 volume_param; /* Parameters to the volume Z equation */

uniform vec2 volume_uv_ratio; /* To convert volume uvs to screen uvs */

/* Volume slice to view space depth. */
float volume_z_to_view_z(float z)
{
	if (ProjectionMatrix[3][3] == 0.0) {
		/* Exponential distribution */
		return (exp2(z / volume_param.z) - volume_param.x) / volume_param.y;
	}
	else {
		/* Linear distribution */
		return mix(volume_param.x, volume_param.y, z);
	}
}

float view_z_to_volume_z(float depth)
{
	if (ProjectionMatrix[3][3] == 0.0) {
		/* Exponential distribution */
		return volume_param.z * log2(depth * volume_param.y + volume_param.x);
	}
	else {
		/* Linear distribution */
		return (depth - volume_param.x) * volume_param.z;
	}
}

/* Volume texture normalized coordinates to NDC (special range [0, 1]). */
vec3 volume_to_ndc(vec3 cos)
{
	cos.z = volume_z_to_view_z(cos.z);
	cos.z = get_depth_from_view_z(cos.z);
	cos.xy /= volume_uv_ratio;
	return cos;
}

vec3 ndc_to_volume(vec3 cos)
{
	cos.z = get_view_z_from_depth(cos.z);
	cos.z = view_z_to_volume_z(cos.z);
	cos.xy *= volume_uv_ratio;
	return cos;
}

float phase_function_isotropic()
{
	return 1.0 / (4.0 * M_PI);
}

float phase_function(vec3 v, vec3 l, float g)
{
	/* Henyey-Greenstein */
	float cos_theta = dot(v, l);
	g = clamp(g, -1.0 + 1e-3, 1.0 - 1e-3);
	float sqr_g = g * g;
	return (1- sqr_g) / (4.0 * M_PI * pow(1 + sqr_g - 2 * g * cos_theta, 3.0 / 2.0));
}

#ifdef LAMPS_LIB
vec3 light_volume(LightData ld, vec4 l_vector)
{
	float power;
	float dist = max(1e-4, abs(l_vector.w - ld.l_radius));
	/* TODO : Area lighting ? */
	/* XXX : Removing Area Power. */
	/* TODO : put this out of the shader. */
	if (ld.l_type == AREA) {
		power = 0.0962 * (ld.l_sizex * ld.l_sizey * 4.0f * M_PI);
	}
	else {
		power = 0.0248 * (4.0 * ld.l_radius * ld.l_radius * M_PI * M_PI);
	}

	/* OPTI: find a better way than calculating this on the fly */
	float lum = dot(ld.l_color, vec3(0.3, 0.6, 0.1)); /* luminance approx. */
	vec3 tint = (lum > 0.0) ? ld.l_color / lum : vec3(1.0); /* normalize lum. to isolate hue+sat */

	lum = min(lum * power / (l_vector.w * l_vector.w), volume_light_clamp);

	return tint * lum;
}

#define VOLUMETRIC_SHADOW_MAX_STEP 32.0

uniform float volume_shadows_steps;

vec3 participating_media_extinction(vec3 wpos, sampler3D volume_extinction)
{
	/* Waiting for proper volume shadowmaps and out of frustum shadow map. */
	vec3 ndc = project_point(ViewProjectionMatrix, wpos);
	vec3 volume_co = ndc_to_volume(ndc * 0.5 + 0.5);

	/* Let the texture be clamped to edge. This reduce visual glitches. */
	return texture(volume_extinction, volume_co).rgb;
}

vec3 light_volume_shadow(LightData ld, vec3 ray_wpos, vec4 l_vector, sampler3D volume_extinction)
{
#if defined(VOLUME_SHADOW)
	/* Heterogeneous volume shadows */
	float dd = l_vector.w / volume_shadows_steps;
	vec3 L = l_vector.xyz * l_vector.w;
	vec3 shadow = vec3(1.0);
	for (float s = 0.5; s < VOLUMETRIC_SHADOW_MAX_STEP && s < (volume_shadows_steps - 0.1); s += 1.0) {
		vec3 pos = ray_wpos + L * (s / volume_shadows_steps);
		vec3 s_extinction = participating_media_extinction(pos, volume_extinction);
		shadow *= exp(-s_extinction * dd);
	}
	return shadow;
#else
	return vec3(1.0);
#endif /* VOLUME_SHADOW */
}
#endif

#ifdef IRRADIANCE_LIB
vec3 irradiance_volumetric(vec3 wpos)
{
	IrradianceData ir_data = load_irradiance_cell(0, vec3(1.0));
	vec3 irradiance = ir_data.cubesides[0] + ir_data.cubesides[1] + ir_data.cubesides[2];
	ir_data = load_irradiance_cell(0, vec3(-1.0));
	irradiance += ir_data.cubesides[0] + ir_data.cubesides[1] + ir_data.cubesides[2];
	irradiance *= 0.16666666; /* 1/6 */
	return irradiance;
}
#endif


```

<br>

*volumetric_scatter_frag.glsl*
```

/* Based on Frosbite Unified Volumetric.
 * https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite */

/* Step 2 : Evaluate all light scattering for each froxels.
 * Also do the temporal reprojection to fight aliasing artifacts. */

uniform sampler3D volumeScattering;
uniform sampler3D volumeExtinction;
uniform sampler3D volumeEmission;
uniform sampler3D volumePhase;

uniform sampler3D historyScattering;
uniform sampler3D historyTransmittance;

uniform vec3 volume_jitter;
uniform float volume_history_alpha;
uniform int light_count;
uniform mat4 PastViewProjectionMatrix;

flat in int slice;

layout(location = 0) out vec4 outScattering;
layout(location = 1) out vec4 outTransmittance;

#define VOLUME_LIGHTING

void main()
{
	vec3 volume_tex_size = vec3(textureSize(volumeScattering, 0));
	ivec3 volume_cell = ivec3(gl_FragCoord.xy, slice);

	/* Emission */
	outScattering = texelFetch(volumeEmission, volume_cell, 0);
	outTransmittance = texelFetch(volumeExtinction, volume_cell, 0);
	vec3 s_scattering = texelFetch(volumeScattering, volume_cell, 0).rgb;
	vec3 volume_ndc = volume_to_ndc((vec3(volume_cell) + volume_jitter) / volume_tex_size);
	vec3 worldPosition = get_world_space_from_depth(volume_ndc.xy, volume_ndc.z);
	vec3 wdir = cameraVec;

	vec2 phase = texelFetch(volumePhase, volume_cell, 0).rg;
	float s_anisotropy = phase.x / phase.y;

	/* Environment : Average color. */
	outScattering.rgb += irradiance_volumetric(worldPosition) * s_scattering * phase_function_isotropic();

#ifdef VOLUME_LIGHTING /* Lights */
	for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {

		LightData ld = lights_data[i];

		vec4 l_vector;
		l_vector.xyz = ld.l_position - worldPosition;
		l_vector.w = length(l_vector.xyz);

		float Vis = light_visibility(ld, worldPosition, l_vector);

		vec3 Li = light_volume(ld, l_vector) * light_volume_shadow(ld, worldPosition, l_vector, volumeExtinction);

		outScattering.rgb += Li * Vis * s_scattering * phase_function(-wdir, l_vector.xyz / l_vector.w, s_anisotropy);
	}
#endif

	/* Temporal supersampling */
	/* Note : this uses the cell non-jittered position (texel center). */
	vec3 curr_ndc = volume_to_ndc(vec3(gl_FragCoord.xy, float(slice) + 0.5) / volume_tex_size);
	vec3 wpos = get_world_space_from_depth(curr_ndc.xy, curr_ndc.z);
	vec3 prev_ndc = project_point(PastViewProjectionMatrix, wpos);
	vec3 prev_volume = ndc_to_volume(prev_ndc * 0.5 + 0.5);

	if ((volume_history_alpha > 0.0) && all(greaterThan(prev_volume, vec3(0.0))) && all(lessThan(prev_volume, vec3(1.0)))) {
		vec4 h_Scattering = texture(historyScattering, prev_volume);
		vec4 h_Transmittance = texture(historyTransmittance, prev_volume);
		outScattering = mix(outScattering, h_Scattering, volume_history_alpha);
		outTransmittance = mix(outTransmittance, h_Transmittance, volume_history_alpha);
	}

	/* Catch NaNs */
	if (any(isnan(outScattering)) || any(isnan(outTransmittance))) {
		outScattering = vec4(0.0);
		outTransmittance = vec4(0.0);
	}
}
```

<br><br>

### Step 3: Integration

#### 1. volumetric_integ_fb 初始化

*eevee_effects.c*
```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	/* Final integration: We compute for each froxel the
	* amount of scattered light and extinction coef at this
	* given depth. We use theses textures as double buffer
	* for the volumetric history. */
	txl->volume_scatter_history = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
	txl->volume_transmittance_history = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);

	...
	DRWFboTexture tex_vol_integ[2] = {&txl->volume_scatter_history, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER},
		                                  {&txl->volume_transmittance_history, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER};

	DRW_framebuffer_init(&fbl->volumetric_integ_fb, &draw_engine_eevee_type,
							(int)froxel_tex_size[0], (int)froxel_tex_size[1],
							tex_vol_integ, 2);
	...
}
```


#### 2. volumetric_integration_ps 初始化
*eevee_effects.c*
```

void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, datatoc_bsdf_common_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_volumetric_lib_glsl);
	e_data.volumetric_common_lib = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	...
	e_data.volumetric_integration_sh = DRW_shader_create_with_lib(datatoc_volumetric_vert_glsl,
		                                                              datatoc_volumetric_geom_glsl,
		                                                              datatoc_volumetric_integration_frag_glsl,
		                                                              e_data.volumetric_common_lib, NULL);
	...
}

void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	psl->volumetric_integration_ps = DRW_pass_create("Volumetric Integration", DRW_STATE_WRITE_COLOR);
	grp = DRW_shgroup_empty_tri_batch_create(e_data.volumetric_integration_sh, psl->volumetric_integration_ps, volumetrics->froxel_tex_size[2]);
	DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)stl->g_data->viewvecs, 2);
	DRW_shgroup_uniform_vec2(grp, "volume_uv_ratio", (float *)volumetrics->volume_coord_scale, 1);
	DRW_shgroup_uniform_vec3(grp, "volume_param", (float *)volumetrics->depth_param, 1);
	DRW_shgroup_uniform_buffer(grp, "volumeScattering", &txl->volume_scatter);
	DRW_shgroup_uniform_buffer(grp, "volumeExtinction", &txl->volume_transmittance);
	...
}
```
>
- ps 使用了 volumetric_integration_frag.glsl
<br><br>
- 这里也是使用 DRW_shgroup_empty_tri_batch_create，把东西渲染到 3D Texture 上

<br><br>

#### 3. volumetric_integration_ps Shader
*volumetric_integration_frag.glsl*
```

/* Based on Frosbite Unified Volumetric.
 * https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite */

/* Step 3 : Integrate for each froxel the final amount of light
 * scattered back to the viewer and the amout of transmittance. */

uniform sampler3D volumeScattering; /* Result of the scatter step */
uniform sampler3D volumeExtinction;

flat in int slice;

layout(location = 0) out vec4 finalScattering;
layout(location = 1) out vec4 finalTransmittance;

void main()
{
	/* Start with full transmittance and no scattered light. */
	finalScattering = vec4(0.0);
	finalTransmittance = vec4(1.0);

	vec3 tex_size = vec3(textureSize(volumeScattering, 0).xyz);

	/* Compute view ray. */
	vec2 uvs = gl_FragCoord.xy / tex_size.xy;
	vec3 ndc_cell = volume_to_ndc(vec3(uvs, 1e-5));
	vec3 view_cell = get_view_space_from_depth(ndc_cell.xy, ndc_cell.z);

	/* Ortho */
	float prev_ray_len = view_cell.z;
	float orig_ray_len = 1.0;

	/* Persp */
	if (ProjectionMatrix[3][3] == 0.0) {
		prev_ray_len = length(view_cell);
		orig_ray_len = prev_ray_len / view_cell.z;
	}

	/* Without compute shader and arbitrary write we need to
	 * accumulate from the beginning of the ray for each cell. */
	float integration_end = float(slice);
	for (int i = 0; i < slice; ++i) {
		ivec3 volume_cell = ivec3(gl_FragCoord.xy, i);

		vec4 Lscat = texelFetch(volumeScattering, volume_cell, 0);
		vec4 s_extinction = texelFetch(volumeExtinction, volume_cell, 0);

		float cell_depth = volume_z_to_view_z((float(i) + 1.0) / tex_size.z);
		float ray_len = orig_ray_len * cell_depth;

		/* Evaluate Scattering */
		float s_len = abs(ray_len - prev_ray_len);
		prev_ray_len = ray_len;
		vec4 Tr = exp(-s_extinction * s_len);

		/* integrate along the current step segment */
		Lscat = (Lscat - Lscat * Tr) / s_extinction;
		/* accumulate and also take into account the transmittance from previous steps */
		finalScattering += finalTransmittance * Lscat;

		finalTransmittance *= Tr;
	}
}
```


<br><br>


### Step 4: Apply for opaque

#### 1. >volumetric_resolve_ps 初始化

```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	e_data.volumetric_resolve_sh = DRW_shader_create_with_lib(datatoc_gpu_shader_fullscreen_vert_glsl, NULL,
																datatoc_volumetric_resolve_frag_glsl,
																e_data.volumetric_common_lib, NULL);
	...
}


void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	psl->volumetric_resolve_ps = DRW_pass_create("Volumetric Resolve", DRW_STATE_WRITE_COLOR);
	grp = DRW_shgroup_create(e_data.volumetric_resolve_sh, psl->volumetric_resolve_ps);
	DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)stl->g_data->viewvecs, 2);
	DRW_shgroup_uniform_vec2(grp, "volume_uv_ratio", (float *)volumetrics->volume_coord_scale, 1);
	DRW_shgroup_uniform_vec3(grp, "volume_param", (float *)volumetrics->depth_param, 1);
	DRW_shgroup_uniform_buffer(grp, "inScattering", &txl->volume_scatter_history);
	DRW_shgroup_uniform_buffer(grp, "inTransmittance", &txl->volume_transmittance_history);
	DRW_shgroup_uniform_buffer(grp, "inSceneColor", &e_data.color_src);
	DRW_shgroup_uniform_buffer(grp, "inSceneDepth", &e_data.depth_src);
	DRW_shgroup_call_add(grp, quad, NULL);
	...
}
```
>
- ps 使用了 volumetric_resolve_frag.glsl
- 这里没有使用 DRW_shgroup_empty_tri_batch_create 函数，而是使用了 quad 作为gero

<br><br>

#### 2. >volumetric_resolve_ps Shader

*volumetric_resolve_frag.glsl*
```

/* Based on Frosbite Unified Volumetric.
 * https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite */

/* Step 4 : Apply final integration on top of the scene color.
 * Note that we do the blending ourself instead of relying
 * on hardware blending which would require 2 pass. */

uniform sampler3D inScattering;
uniform sampler3D inTransmittance;

uniform sampler2D inSceneColor;
uniform sampler2D inSceneDepth;

out vec4 FragColor;

void main()
{
	vec2 uvs = gl_FragCoord.xy / vec2(textureSize(inSceneDepth, 0));
	vec3 volume_cos = ndc_to_volume(vec3(uvs, texture(inSceneDepth, uvs).r));

	vec3 scene_color = texture(inSceneColor, uvs).rgb;
	vec3 scattering = texture(inScattering, volume_cos).rgb;
	vec3 transmittance = texture(inTransmittance, volume_cos).rgb;

	FragColor = vec4(scene_color * transmittance + scattering, 1.0);
}

```

<br><br>

### Volumetric Shader 参数

```
void EEVEE_effects_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	if (BKE_collection_engine_property_value_get_bool(props, "volumetric_enable")) {

		effects->enabled_effects |= EFFECT_VOLUMETRIC;

		if (sldata->volumetrics == NULL) {
			sldata->volumetrics = MEM_callocN(sizeof(EEVEE_VolumetricsInfo), "EEVEE_VolumetricsInfo");
		}

		EEVEE_VolumetricsInfo *volumetrics = sldata->volumetrics;

		int tile_size = BKE_collection_engine_property_value_get_int(props, "volumetric_tile_size");

		/* Find Froxel Texture resolution. */
		int froxel_tex_size[3];

		froxel_tex_size[0] = (int)ceilf(fmaxf(1.0f, viewport_size[0] / (float)tile_size));
		froxel_tex_size[1] = (int)ceilf(fmaxf(1.0f, viewport_size[1] / (float)tile_size));
		froxel_tex_size[2] = max_ii(BKE_collection_engine_property_value_get_int(props, "volumetric_samples"), 1);

		volumetrics->volume_coord_scale[0] = viewport_size[0] / (float)(tile_size * froxel_tex_size[0]);
		volumetrics->volume_coord_scale[1] = viewport_size[1] / (float)(tile_size * froxel_tex_size[1]);

		/* TODO compute snap to maxZBuffer for clustered rendering */

		if ((volumetrics->froxel_tex_size[0] != froxel_tex_size[0]) ||
		    (volumetrics->froxel_tex_size[1] != froxel_tex_size[1]) ||
		    (volumetrics->froxel_tex_size[2] != froxel_tex_size[2]))
		{
			DRW_TEXTURE_FREE_SAFE(txl->volume_prop_scattering);
			DRW_TEXTURE_FREE_SAFE(txl->volume_prop_extinction);
			DRW_TEXTURE_FREE_SAFE(txl->volume_prop_emission);
			DRW_TEXTURE_FREE_SAFE(txl->volume_prop_phase);
			DRW_TEXTURE_FREE_SAFE(txl->volume_scatter);
			DRW_TEXTURE_FREE_SAFE(txl->volume_transmittance);
			DRW_TEXTURE_FREE_SAFE(txl->volume_scatter_history);
			DRW_TEXTURE_FREE_SAFE(txl->volume_transmittance_history);
			DRW_FRAMEBUFFER_FREE_SAFE(fbl->volumetric_fb);
			DRW_FRAMEBUFFER_FREE_SAFE(fbl->volumetric_scat_fb);
			DRW_FRAMEBUFFER_FREE_SAFE(fbl->volumetric_integ_fb);
			volumetrics->froxel_tex_size[0] = froxel_tex_size[0];
			volumetrics->froxel_tex_size[1] = froxel_tex_size[1];
			volumetrics->froxel_tex_size[2] = froxel_tex_size[2];

			volumetrics->inv_tex_size[0] = 1.0f / (float)(volumetrics->froxel_tex_size[0]);
			volumetrics->inv_tex_size[1] = 1.0f / (float)(volumetrics->froxel_tex_size[1]);
			volumetrics->inv_tex_size[2] = 1.0f / (float)(volumetrics->froxel_tex_size[2]);
		}

		/* Like frostbite's paper, 5% blend of the new frame. */
		volumetrics->history_alpha = (txl->volume_prop_scattering == NULL) ? 0.0f : 0.95f;

		if (txl->volume_prop_scattering == NULL) {
			/* Volume properties: We evaluate all volumetric objects
			 * and store their final properties into each froxel */
			txl->volume_prop_scattering = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
			txl->volume_prop_extinction = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
			txl->volume_prop_emission = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
			txl->volume_prop_phase = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RG_16, DRW_TEX_FILTER, NULL);

			/* Volume scattering: We compute for each froxel the
			 * Scattered light towards the view. We also resolve temporal
			 * super sampling during this stage. */
			txl->volume_scatter = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
			txl->volume_transmittance = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);

			/* Final integration: We compute for each froxel the
			 * amount of scattered light and extinction coef at this
			 * given depth. We use theses textures as double buffer
			 * for the volumetric history. */
			txl->volume_scatter_history = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
			txl->volume_transmittance_history = DRW_texture_create_3D(froxel_tex_size[0], froxel_tex_size[1], froxel_tex_size[2], DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER, NULL);
		}

		/* Temporal Super sampling jitter */
		double ht_point[3];
		double ht_offset[3] = {0.0, 0.0};
		unsigned int ht_primes[3] = {3, 7, 2};
		static unsigned int current_sample = 0;
		struct wmWindowManager *wm = CTX_wm_manager(draw_ctx->evil_C);

		if (((effects->enabled_effects & EFFECT_TAA) != 0) && (ED_screen_animation_no_scrub(wm) == NULL)) {
			/* If TAA is in use do not use the history buffer. */
			volumetrics->history_alpha = 0.0f;
			current_sample = effects->taa_current_sample - 1;
		}
		else {
			current_sample = (current_sample + 1) % (ht_primes[0] * ht_primes[1] * ht_primes[2]);
		}
		BLI_halton_3D(ht_primes, ht_offset, current_sample, ht_point);

		volumetrics->jitter[0] = (float)ht_point[0];
		volumetrics->jitter[1] = (float)ht_point[1];
		volumetrics->jitter[2] = (float)ht_point[2];

		/* Framebuffer setup */
		DRWFboTexture tex_vol[4] = {&txl->volume_prop_scattering, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER},
		                            {&txl->volume_prop_extinction, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER},
		                            {&txl->volume_prop_emission, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER},
		                            {&txl->volume_prop_phase, DRW_TEX_RG_16, DRW_TEX_FILTER};

		DRW_framebuffer_init(&fbl->volumetric_fb, &draw_engine_eevee_type,
		                     (int)froxel_tex_size[0], (int)froxel_tex_size[1],
		                     tex_vol, 4);

		DRWFboTexture tex_vol_scat[2] = {&txl->volume_scatter, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER},
		                                 {&txl->volume_transmittance, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER};

		DRW_framebuffer_init(&fbl->volumetric_scat_fb, &draw_engine_eevee_type,
		                     (int)froxel_tex_size[0], (int)froxel_tex_size[1],
		                     tex_vol_scat, 2);

		DRWFboTexture tex_vol_integ[2] = {&txl->volume_scatter_history, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER},
		                                  {&txl->volume_transmittance_history, DRW_TEX_RGB_11_11_10, DRW_TEX_FILTER};

		DRW_framebuffer_init(&fbl->volumetric_integ_fb, &draw_engine_eevee_type,
		                     (int)froxel_tex_size[0], (int)froxel_tex_size[1],
		                     tex_vol_integ, 2);

		volumetrics->integration_start = BKE_collection_engine_property_value_get_float(props, "volumetric_start");
		volumetrics->integration_end = BKE_collection_engine_property_value_get_float(props, "volumetric_end");
		volumetrics->sample_distribution = 4.0f * (1.00001f - BKE_collection_engine_property_value_get_float(props, "volumetric_sample_distribution"));
		volumetrics->use_volume_shadows = BKE_collection_engine_property_value_get_bool(props, "volumetric_shadows");
		volumetrics->light_clamp = BKE_collection_engine_property_value_get_float(props, "volumetric_light_clamp");

		if (volumetrics->use_volume_shadows) {
			volumetrics->shadow_step_count = (float)BKE_collection_engine_property_value_get_int(props, "volumetric_shadow_samples");
		}
		else {
			volumetrics->shadow_step_count = 0;
		}

		if (DRW_viewport_is_persp_get()) {
			const float clip_start = stl->g_data->viewvecs[0][2];
			/* Negate */
			float near = volumetrics->integration_start = min_ff(-volumetrics->integration_start, clip_start - 1e-4f);
			float far = volumetrics->integration_end = min_ff(-volumetrics->integration_end, near - 1e-4f);

			volumetrics->depth_param[0] = (far - near * exp2(1.0f / volumetrics->sample_distribution)) / (far - near);
			volumetrics->depth_param[1] = (1.0f - volumetrics->depth_param[0]) / near;
			volumetrics->depth_param[2] = volumetrics->sample_distribution;
		}
		else {
			const float clip_start = stl->g_data->viewvecs[0][2];
			const float clip_end = stl->g_data->viewvecs[1][2];
			volumetrics->integration_start = min_ff(volumetrics->integration_end, clip_start);
			volumetrics->integration_end = max_ff(-volumetrics->integration_end, clip_end);

			volumetrics->depth_param[0] = volumetrics->integration_start;
			volumetrics->depth_param[1] = volumetrics->integration_end;
			volumetrics->depth_param[2] = 1.0f / (volumetrics->integration_end - volumetrics->integration_start);
		}

		/* Disable clamp if equal to 0. */
		if (volumetrics->light_clamp == 0.0) {
			volumetrics->light_clamp = FLT_MAX;
		}

		volumetrics->use_lights = BKE_collection_engine_property_value_get_bool(props, "volumetric_lights");
	}
	else {
		/* Cleanup */
		DRW_TEXTURE_FREE_SAFE(txl->volume_prop_scattering);
		DRW_TEXTURE_FREE_SAFE(txl->volume_prop_extinction);
		DRW_TEXTURE_FREE_SAFE(txl->volume_prop_emission);
		DRW_TEXTURE_FREE_SAFE(txl->volume_prop_phase);
		DRW_TEXTURE_FREE_SAFE(txl->volume_scatter);
		DRW_TEXTURE_FREE_SAFE(txl->volume_transmittance);
		DRW_TEXTURE_FREE_SAFE(txl->volume_scatter_history);
		DRW_TEXTURE_FREE_SAFE(txl->volume_transmittance_history);
		DRW_FRAMEBUFFER_FREE_SAFE(fbl->volumetric_fb);
		DRW_FRAMEBUFFER_FREE_SAFE(fbl->volumetric_scat_fb);
		DRW_FRAMEBUFFER_FREE_SAFE(fbl->volumetric_integ_fb);
	}
	...
}

```