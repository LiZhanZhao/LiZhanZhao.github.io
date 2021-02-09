---
layout:     post
title:      "blender eevee Remove Geometry shader usage for background"
subtitle:   ""
date:       2021-2-10 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/29  * Eevee : Remove Geometry shader usage for background.

> This fix the behaviour of the light path node that separates the probes background from the viewport background.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
	

## 作用 
这篇文章主要是分清 background_pass 和 probe_background .

## 编译
- 重新生成SLN
- git 定位到  2017/6/29  * Eevee : Remove Geometry shader usage for background .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## background_pass

### 初始化
*eevee_materials.c*
```
void EEVEE_materials_init(void)
{
	...
	e_data.default_background = DRW_shader_create_fullscreen(
			datatoc_default_world_frag_glsl, NULL);
}


struct GPUMaterial *EEVEE_material_world_background_get(struct Scene *scene, World *wo)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	int options = VAR_WORLD_BACKGROUND;

	GPUMaterial *mat = GPU_material_from_nodetree_find(&wo->gpumaterial, engine, options);
	if (mat != NULL) {
		return mat;
	}
	return GPU_material_from_nodetree(
	        scene, wo->nodetree, &wo->gpumaterial, engine, options,
	        datatoc_background_vert_glsl, NULL, e_data.frag_shader_lib,
	        SHADER_DEFINES "#define WORLD_BACKGROUND\n");
}


void EEVEE_materials_cache_init(EEVEE_Data *vedata)
{
	...
	{
		psl->background_pass = DRW_pass_create("Background Pass", DRW_STATE_WRITE_DEPTH | DRW_STATE_WRITE_COLOR);

		struct Gwn_Batch *geom = DRW_cache_fullscreen_quad_get();
		DRWShadingGroup *grp = NULL;

		const DRWContextState *draw_ctx = DRW_context_state_get();
		Scene *scene = draw_ctx->scene;
		World *wo = scene->world;

		float *col = ts.colorBackground;

		if (wo) {
			col = &wo->horr;

			if (wo->use_nodes && wo->nodetree) {
				struct GPUMaterial *gpumat = EEVEE_material_world_background_get(scene, wo);
				grp = DRW_shgroup_material_create(gpumat, psl->background_pass);

				if (grp) {
					DRW_shgroup_uniform_float(grp, "backgroundAlpha", &stl->g_data->background_alpha, 1);
					DRW_shgroup_call_add(grp, geom, NULL);
				}
				else {
					/* Shader failed : pink background */
					static float pink[3] = {1.0f, 0.0f, 1.0f};
					col = pink;
				}
			}
		}

		/* Fallback if shader fails or if not using nodetree. */
		if (grp == NULL) {
			grp = DRW_shgroup_create(e_data.default_background, psl->background_pass);
			DRW_shgroup_uniform_vec3(grp, "color", col, 1);
			DRW_shgroup_uniform_float(grp, "backgroundAlpha", &stl->g_data->background_alpha, 1);
			DRW_shgroup_call_add(grp, geom, NULL);
		}
	}
	...
}
```
>
- background_pass 在初始化的时候分两种情况，一种是不使用 nodetree，这种情况，background_pass 的Shader 使用的就是 default_background，那就是default_world_frag.glsl, 后处理 DRW_cache_fullscreen_quad_get。<br><br>


- *default_world_frag.glsl*
```
uniform float backgroundAlpha;
uniform vec3 color;
out vec4 FragColor;
void main() {
	FragColor = vec4(color, backgroundAlpha);
}
```
<br><br>


- background_pass 另外的一种情况就是使用 nodetree, 生成代码，值得主要的就是 EEVEE_material_world_background_get 函数，生成出来的 Shader 使用 background_vert.glsl，还有定义了宏 WORLD_BACKGROUND, 后处理 DRW_cache_fullscreen_quad_get <br><br>

- *background_vert.glsl*
```
in vec2 pos;
out vec3 varposition;
out vec3 varnormal;
out vec3 viewPosition;
/* necessary for compilation*/
out vec3 worldPosition;
out vec3 worldNormal;
out vec3 viewNormal;
void main()
{
	gl_Position = vec4(pos, 1.0, 1.0);
	varposition = viewPosition = vec3(pos, -1.0);
	varnormal = normalize(-varposition);
}
```

### 渲染 
*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	DRW_draw_pass(psl->background_pass);
	...
}
```
>
- background_pass 只会在 主渲染管线出现，用来渲染背景用的




## probe_background

### 初始化

*eevee_lightprobes.c*
```
void EEVEE_lightprobes_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *UNUSED(vedata))
{
	...
	e_data.probe_default_sh = DRW_shader_create(
		        datatoc_lightprobe_vert_glsl, datatoc_lightprobe_geom_glsl, datatoc_default_world_frag_glsl, NULL);
	...
}


struct GPUMaterial *EEVEE_material_world_lightprobe_get(struct Scene *scene, World *wo)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	const int options = VAR_WORLD_PROBE;

	GPUMaterial *mat = GPU_material_from_nodetree_find(&wo->gpumaterial, engine, options);
	if (mat != NULL) {
		return mat;
	}
	return GPU_material_from_nodetree(
	        scene, wo->nodetree, &wo->gpumaterial, engine, options,
	        datatoc_background_vert_glsl, NULL, e_data.frag_shader_lib,
	        SHADER_DEFINES "#define PROBE_CAPTURE\n");
}


void EEVEE_lightprobes_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	{
		psl->probe_background = DRW_pass_create("World Probe Pass", DRW_STATE_WRITE_COLOR);

		struct Gwn_Batch *geom = DRW_cache_fullscreen_quad_get();
		DRWShadingGroup *grp = NULL;

		const DRWContextState *draw_ctx = DRW_context_state_get();
		Scene *scene = draw_ctx->scene;
		World *wo = scene->world;

		float *col = ts.colorBackground;
		if (wo) {
			col = &wo->horr;
			e_data.update_world = (wo->update_flag != 0);
			wo->update_flag = 0;

			if (wo->use_nodes && wo->nodetree) {
				struct GPUMaterial *gpumat = EEVEE_material_world_lightprobe_get(scene, wo);

				grp = DRW_shgroup_material_create(gpumat, psl->probe_background);

				if (grp) {
					DRW_shgroup_call_add(grp, geom, NULL);
				}
				else {
					/* Shader failed : pink background */
					static float pink[3] = {1.0f, 0.0f, 1.0f};
					col = pink;
				}
			}
		}

		/* Fallback if shader fails or if not using nodetree. */
		if (grp == NULL) {
			grp = DRW_shgroup_instance_create(e_data.probe_default_sh, psl->probe_background, geom);
			DRW_shgroup_uniform_vec3(grp, "color", col, 1);
			DRW_shgroup_call_add(grp, geom, NULL);
		}
	}
	...
}
```
- probe_background 在初始化的时候分两种情况，一种是不使用 nodetree，这种情况，probe_background 的Shader 使用的就是 lightprobe_vert.glsl, lightprobe_geom.glsl, default_world_frag.glsl, 后处理 DRW_cache_fullscreen_quad_get, 然后这里需要注意的事，没有使用instance渲染(被修改了)，所以geom shader 里gl_InstanceID 没有值<br><br>

- *lightprobe_vert.glsl*<br>
```
in vec3 pos;
out vec4 vPos;
flat out int face;
void main() {
	vPos = vec4(pos, 1.0);
	face = gl_InstanceID;
}
```
<br><br>

- *lightprobe_geom.glsl*<br>
```
layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;
uniform int Layer;
in vec4 vPos[];
flat in int face[];
out vec3 worldPosition;
out vec3 viewPosition; /* Required. otherwise generate linking error. */
out vec3 worldNormal; /* Required. otherwise generate linking error. */
out vec3 viewNormal; /* Required. otherwise generate linking error. */
const vec3 maj_axes[6] = vec3[6](vec3(1.0,  0.0,  0.0), vec3(-1.0,  0.0, 0.0), vec3(0.0, 1.0, 0.0), vec3(0.0, -1.0,  0.0), vec3( 0.0,  0.0, 1.0), vec3( 0.0,  0.0, -1.0));
const vec3 x_axis[6]   = vec3[6](vec3(0.0,  0.0, -1.0), vec3( 0.0,  0.0, 1.0), vec3(1.0, 0.0, 0.0), vec3(1.0,  0.0,  0.0), vec3( 1.0,  0.0, 0.0), vec3(-1.0,  0.0,  0.0));
const vec3 y_axis[6]   = vec3[6](vec3(0.0, -1.0,  0.0), vec3( 0.0, -1.0, 0.0), vec3(0.0, 0.0, 1.0), vec3(0.0,  0.0, -1.0), vec3( 0.0, -1.0, 0.0), vec3( 0.0, -1.0,  0.0));
void main() {
	int f = face[0];
	gl_Layer = Layer + f;
	for (int v = 0; v < 3; ++v) {
		gl_Position = vPos[v];
		worldPosition = x_axis[f] * vPos[v].x + y_axis[f] * vPos[v].y + maj_axes[f];
#ifdef ATTRIB
		pass_attrib(v);
#endif
		EmitVertex();
	}
	EndPrimitive();
}
```
<br><br>


- *default_world_frag.glsl*<br>
```
uniform float backgroundAlpha;
uniform vec3 color;
out vec4 FragColor;
void main() {
	FragColor = vec4(color, backgroundAlpha);
}
```
<br><br>


- probe_background 另外的一种情况就是使用 nodetree, 生成代码，值得主要的就是 EEVEE_material_world_lightprobe_get 函数，生成出来的 Shader 使用 background_vert.glsl，没有geom shader，还有定义了宏 PROBE_CAPTURE, 后处理 DRW_cache_fullscreen_quad_get <br><br>

- *background_vert.glsl*
```
in vec2 pos;
out vec3 varposition;
out vec3 varnormal;
out vec3 viewPosition;
/* necessary for compilation*/
out vec3 worldPosition;
out vec3 worldNormal;
out vec3 viewNormal;
void main()
{
	gl_Position = vec4(pos, 1.0, 1.0);
	varposition = viewPosition = vec3(pos, -1.0);
	varnormal = normalize(-varposition);
}
```


### 渲染
*eevee_lightprobes.c*
```
static void render_scene_to_probe(...)
{
	...
	for (int i = 0; i < 6; ++i) {
		...
		/* Setup custom matrices */
		mul_m4_m4m4(viewmat, cubefacemat[i], posmat);
		mul_m4_m4m4(persmat, winmat, viewmat);
		invert_m4_m4(persinv, persmat);
		invert_m4_m4(viewinv, viewmat);
		...
		DRW_draw_pass(psl->probe_background);
		...
		/* Shading pass */
		EEVEE_draw_default_passes(psl);
		DRW_draw_pass(psl->material_pass);
		...
	}
	...
}

static void render_scene_to_planar(...){
	...
	/* Background */
	DRW_draw_pass(psl->probe_background);
	...
	/* Shading pass */
	EEVEE_draw_default_passes(psl);
	DRW_draw_pass(psl->material_pass);
	...
}

static void render_world_to_probe(...)
{
	...
	for (int i = 0; i < 6; ++i) {
		...
		/* Setup custom matrices */
		copy_m4_m4(viewmat, cubefacemat[i]);
		mul_m4_m4m4(persmat, winmat, viewmat);
		invert_m4_m4(persinv, persmat);
		invert_m4_m4(viewinv, viewmat);
		...
		DRW_draw_pass(psl->probe_background);
	}
	...
}
```
>
- render_scene_to_probe 是一个 Grid 每一个 Cell 进行渲染6次，把背景和场景的东西都渲染到一个Cubemap的每一个面上，渲染到cubemap上的每一个面的背景不是一样的，因为定义了宏PROBE_CAPTURE，在nodetree生成的代码中，会有区分，例如
- *gpu_shader_material.glsl*
```
void background_transform_to_world(vec3 viewvec, out vec3 worldvec)
{
	vec4 v = (ProjectionMatrix[3][3] == 0.0) ? vec4(viewvec, 1.0) : vec4(0.0, 0.0, 1.0, 1.0);
	vec4 co_homogenous = (ProjectionMatrixInverse * v);
	vec4 co = vec4(co_homogenous.xyz / co_homogenous.w, 0.0);
#ifdef WORLD_BACKGROUND
	worldvec = (ViewMatrixInverse * co).xyz;
#else
	worldvec = (ModelViewMatrixInverse * co).xyz;
#endif
}
```
```
#if defined(PROBE_CAPTURE) || defined(WORLD_BACKGROUND)
void environment_default_vector(out vec3 worldvec)
{
#ifdef WORLD_BACKGROUND
	background_transform_to_world(viewPosition, worldvec);
#else
	worldvec = normalize(worldPosition);
#endif
}
#endif
```
- render_scene_to_planar 只是渲染到 planar上，所以把背景和场景渲染到planar上
- render_world_to_probe 只是把背景渲染到 cubemap 上，只是渲染背景，不会渲染场景物体, 这个是world Probe逻辑