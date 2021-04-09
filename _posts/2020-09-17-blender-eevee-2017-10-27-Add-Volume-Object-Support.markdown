---
layout:     post
title:      "blender eevee Volumetrics Add Volume object support"
subtitle:   ""
date:       2021-4-8 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/10/27  *   Eevee : Volumetrics: Add Volume object support. <br> 

> This is quite basic as it only support boundbing boxes.
But the material can refine the volume shape in anyway the user like.

>
To overcome this limitation, a voxelisation should be done on the mesh (generating a SDF maybe?) and tested against every volumetric cell.

> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
物体可以进行 Volume


## 效果
*想要看到效果, Viewport Samples(TAA Sample) 要先设置为 1*
![](/img/Eevee/Volumetrics+/02/1.png)

<br><br>


### 渲染入口

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
		DRW_draw_pass(psl->volumetric_world_ps);
		DRW_draw_pass(psl->volumetric_objects_ps);

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
>
- 这里最主要是 Step 1: Participating Media Properties 中，多了 DRW_draw_pass(psl->volumetric_objects_ps);

<br><br>

### volumetric_objects_ps

#### 1.初始化

*eevee_materials.c*

```
static char *eevee_get_volume_defines(int options)
{
	char *str = NULL;

	DynStr *ds = BLI_dynstr_new();
	BLI_dynstr_appendf(ds, SHADER_DEFINES);
	BLI_dynstr_appendf(ds, "#define VOLUMETRICS\n");

	if ((options & VAR_MAT_VOLUME) != 0) {
		BLI_dynstr_appendf(ds, "#define MESH_SHADER\n");
	}

	str = BLI_dynstr_get_cstring(ds);
	BLI_dynstr_free(ds);

	return str;
}

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

struct GPUMaterial *EEVEE_material_mesh_volume_get(struct Scene *scene, Material *ma)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	int options = VAR_MAT_VOLUME;

	GPUMaterial *mat = GPU_material_from_nodetree_find(&ma->gpumaterial, engine, options);
	if (mat != NULL) {
		return mat;
	}

	char *defines = eevee_get_volume_defines(options);

	mat = GPU_material_from_nodetree(
	        scene, ma->nodetree, &ma->gpumaterial, engine, options,
	        datatoc_volumetric_vert_glsl, datatoc_volumetric_geom_glsl, e_data.volume_shader_lib,
	        defines);

	MEM_freeN(defines);

	return mat;
}
```

<br><br>

*eevee_effects.c*
```

void BKE_mesh_texspace_get_reference(Mesh *me, short **r_texflag,  float **r_loc, float **r_rot, float **r_size)
{
	if (me->bb == NULL || (me->bb->flag & BOUNDBOX_DIRTY)) {
		BKE_mesh_texspace_calc(me);
	}

	if (r_texflag != NULL) *r_texflag = &me->texflag;
	if (r_loc != NULL) *r_loc = me->loc;
	if (r_rot != NULL) *r_rot = me->rot;
	if (r_size != NULL) *r_size = me->size;
}

void EEVEE_effects_cache_volume_object_add(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata, Scene *scene, Object *ob)
{
	float *texcoloc = NULL;
	float *texcosize = NULL;
	EEVEE_VolumetricsInfo *volumetrics = sldata->volumetrics;
	Material *ma = give_current_material(ob, 1);

	if (ma == NULL) {
		return;
	}

	struct GPUMaterial *mat = EEVEE_material_mesh_volume_get(scene, ma);

	DRWShadingGroup *grp = DRW_shgroup_material_empty_tri_batch_create(mat, vedata->psl->volumetric_objects_ps, volumetrics->froxel_tex_size[2]);

	/* Making sure it's updated. */
	invert_m4_m4(ob->imat, ob->obmat);

	BKE_mesh_texspace_get_reference((struct Mesh *)ob->data, NULL, &texcoloc, NULL, &texcosize);

	if (grp) {
		DRW_shgroup_uniform_mat4(grp, "volumeObjectMatrix", (float *)ob->imat);
		DRW_shgroup_uniform_vec3(grp, "volumeOrcoLoc", texcoloc, 1);
		DRW_shgroup_uniform_vec3(grp, "volumeOrcoSize", texcosize, 1);
		DRW_shgroup_uniform_vec4(grp, "viewvecs[0]", (float *)vedata->stl->g_data->viewvecs, 2);
		DRW_shgroup_uniform_ivec3(grp, "volumeTextureSize", (int *)volumetrics->froxel_tex_size, 1);
		DRW_shgroup_uniform_vec2(grp, "volume_uv_ratio", (float *)volumetrics->volume_coord_scale, 1);
		DRW_shgroup_uniform_vec3(grp, "volume_param", (float *)volumetrics->depth_param, 1);
		DRW_shgroup_uniform_vec3(grp, "volume_jitter", (float *)volumetrics->jitter, 1);
	}
}

void EEVEE_effects_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	/* Volumetric Objects */
	psl->volumetric_objects_ps = DRW_pass_create("Volumetric Properties", DRW_STATE_WRITE_COLOR | DRW_STATE_ADDITIVE);
	...
}
```
>
- volumetric_objects_ps 使用了 volumetric_vert.glsl, volumetric_geom.glsl, volumetric_frag.glsl，并且定义了宏 MESH_SHADER


<br><br>

#### 2.应用
*eevee_materials.c*
```
void EEVEE_materials_cache_populate(EEVEE_Data *vedata, EEVEE_SceneLayerData *sldata, Object *ob)
{
	...
	/* Only support single volume material for now. */
	/* XXX We rely on the previously compiled surface shader
		* to know if the material has a "volume nodetree".
		*/
	bool use_volume_material = (gpumat_array[0] && GPU_material_use_domain_volume(gpumat_array[0]));
	...
	/* Volumetrics */
	if (vedata->stl->effects->use_volumetrics && use_volume_material) {
		EEVEE_effects_cache_volume_object_add(sldata, vedata, scene, ob);
	}
	...
}
```
>
- 这里如果物体的材质使用了Volume的话，就会加入到 volumetric_objects_ps 中进行渲染

<br><br>

#### 3. Shader

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
- 可以参考 [Overhaul The Volumetric System](http://shaderstore.cn/2021/04/07/blender-eevee-2017-10-24-Overhaul-The-Volumetric-System/) 


<br><br>
*volumetric_geom.glsl*
```
#ifdef MESH_SHADER
/* TODO tight slices */
layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;
#else /* World */
layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;
#endif

in vec4 vPos[];

flat out int slice;

#ifdef MESH_SHADER
/* TODO tight slices */
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

#else /* World */

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

#endif
```
>
- 注意这里要留意宏 MESH_SHADER 的分支


<br><br>

*volumetric_frag.glsl*
```

/* Based on Frosbite Unified Volumetric.
 * https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite */

#define NODETREE_EXEC

uniform ivec3 volumeTextureSize;
uniform vec3 volume_jitter;

#ifdef MESH_SHADER
uniform mat4 volumeObjectMatrix;
uniform vec3 volumeOrcoLoc;
uniform vec3 volumeOrcoSize;
#endif

flat in int slice;

/* Warning: theses are not attributes, theses are global vars. */
vec3 worldPosition = vec3(0.0);
vec3 viewPosition = vec3(0.0);
vec3 viewNormal = vec3(0.0);
#ifdef MESH_SHADER
vec3 volumeObjectLocalCoord = vec3(0.0);
#endif

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
#ifdef MESH_SHADER
	volumeObjectLocalCoord = transform_point(volumeObjectMatrix, worldPosition);
	volumeObjectLocalCoord = (volumeObjectLocalCoord - volumeOrcoLoc + volumeOrcoSize) / (volumeOrcoSize * 2.0);

	if (any(lessThan(volumeObjectLocalCoord, vec3(0.0))) ||
	    any(greaterThan(volumeObjectLocalCoord, vec3(1.0))))
	    discard;
#endif

#ifdef CLEAR
	Closure cl = CLOSURE_DEFAULT;
#else
	Closure cl = nodetree_exec();
#endif

	volumeScattering = vec4(cl.scatter, 1.0);
	volumeExtinction = vec4(max(vec3(1e-4), cl.absorption + cl.scatter), 1.0);
	volumeEmissive = vec4(cl.emission, 1.0);

	/* Do not add phase weight if no scattering. */
	if (all(equal(cl.scatter, vec3(0.0)))) {
		volumePhase = vec4(0.0);
	}
	else {
		volumePhase = vec4(cl.anisotropy, vec3(1.0));
	}
}

```
>
- 注意这里要留意宏 MESH_SHADER 的分支
- 这里主要是 传入 物体的 World -> Object 矩阵，然后计算全屏的的WorldPosition 把Volume限制在 Object的范围内

<br><br>


*gpu_shader_material.glsl*
```
void node_output_material(Closure surface, Closure volume, float displacement, out Closure result)
{
#ifdef VOLUMETRICS
	result = volume;
#else
	result = surface;
#endif
}
```