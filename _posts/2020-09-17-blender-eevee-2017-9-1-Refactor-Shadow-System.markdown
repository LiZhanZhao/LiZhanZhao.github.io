---
layout:     post
title:      "blender eevee Refactor Shadow System"
subtitle:   ""
date:       2021-3-21 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/9/1  *   Eevee : Refactor Shadow System <br> 

> 
- Use only one 2d texture array to store all shadowmaps.
- Allow to change shadow maps resolution.
- Do not output radial distance when rendering shadowmaps. This will allow fast rendering of shadowmaps when we will drop the use of geometry shaders.


> SVN : 2017/8/17  [windows] add numpy headers to python include dir, required for D2716


## 效果
![](/img/Eevee/Shadow-2/1.png)

## 作用
重构影子系统

<br>

### A. 渲染入口
*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
    ...
    /* Refresh shadows */
    DRW_stats_group_start("Shadows");
    EEVEE_draw_shadows(sldata, psl);
    DRW_stats_group_end();
    ...
}
```
>
- 渲染入口在 EEVEE_draw_shadows 函数中

<br>

### B. EEVEE_draw_shadows

```
if (!sldata->shadow_cube_target) {
	/* TODO render everything on the same 2d render target using clip planes and no Geom Shader. */
	/* Cubemaps */

	sldata->shadow_cube_target = DRW_texture_create_cube(linfo->shadow_cube_target_size, DRW_TEX_DEPTH_24, 0, NULL);
}


/* Initialize Textures Array first so DRW_framebuffer_init just bind them. */
if (!sldata->shadow_pool) {
	/* All shadows fit in this array */
	sldata->shadow_pool = DRW_texture_create_2D_array(
			linfo->shadow_size, linfo->shadow_size, max_ff(1, linfo->num_cube + linfo->num_cascade),
			shadow_pool_format, DRW_TEX_FILTER, NULL);
}
```
>
- shadow_cube_target 和 shadow_pool 的初始化

<br>


*eevee_lights.c*
```
/* this refresh lamps shadow buffers */
void EEVEE_draw_shadows(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	EEVEE_LampsInfo *linfo = sldata->lamps;
	Object *ob;
	int i;
	float clear_col[4] = {FLT_MAX};

	/* Cube Shadow Maps */
	/* Render each shadow to one layer of the array */
	for (i = 0; (ob = linfo->shadow_cube_ref[i]) && (i < MAX_SHADOW_CUBE); i++) {
		EEVEE_LampEngineData *led = EEVEE_lamp_data_get(ob);
		Lamp *la = (Lamp *)ob->data;

		if (led->need_update) {
			EEVEE_ShadowCubeData *evscd = (EEVEE_ShadowCubeData *)led->storage;
			EEVEE_ShadowRender *srd = &linfo->shadow_render_data;

			srd->shadow_samples_ct = 32.0f;
			srd->shadow_inv_samples_ct = 1.0f / srd->shadow_samples_ct;
			srd->clip_near = la->clipsta;
			srd->clip_far = la->clipend;
			copy_v3_v3(srd->position, ob->obmat[3]);
			for (int j = 0; j < 6; j++) {
				float tmp[4][4];

				unit_m4(tmp);
				negate_v3_v3(tmp[3], ob->obmat[3]);
				mul_m4_m4m4(srd->viewmat[j], cubefacemat[j], tmp);

				copy_m4_m4(srd->shadowmat[j], evscd->viewprojmat[j]);
			}
			DRW_uniformbuffer_update(sldata->shadow_render_ubo, srd);

			DRW_framebuffer_texture_attach(sldata->shadow_target_fb, sldata->shadow_cube_target, 0, 0);
			DRW_framebuffer_bind(sldata->shadow_target_fb);
			DRW_framebuffer_clear(true, true, false, clear_col, 1.0f);
			/* Render shadow cube */
			DRW_draw_pass(psl->shadow_cube_pass);

			/* Push it to shadowmap array */
			DRW_framebuffer_texture_layer_attach(sldata->shadow_store_fb, sldata->shadow_pool, 0, i, 0);
			DRW_framebuffer_bind(sldata->shadow_store_fb);
			DRW_draw_pass(psl->shadow_cube_store_pass);

			led->need_update = false;
		}
	}
	linfo->update_flag &= ~LIGHT_UPDATE_SHADOW_CUBE;

	DRW_framebuffer_texture_detach(sldata->shadow_cube_target);

	/* Cascaded Shadow Maps */
    ...
}

```

>
- 渲染影子的RT主要是 EEVEE_draw_shadows 函数进行渲染
<br><br>
- 这里主要讲述 Cube Shadow Maps 步骤				
<br><br>
- Render shadow cube 使用了 shadow_cube_pass 			
<br><br>
- Push it to shadowmap array 使用了 shadow_cube_store_pass。		
<br><br>
- shadow_cube_pass 渲染到 RT shadow_cube_target上。					
<br><br>
- shadow_cube_store_pass 渲染到 RT shadow_pool 上。					
<br><br>

<br>

### C. shadow_cube_pass

#### 1.初始化

```
void EEVEE_lights_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
    ...
    {
		psl->shadow_cube_pass = DRW_pass_create("Shadow Cube Pass", DRW_STATE_WRITE_COLOR | DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS);
	}

	{
		psl->shadow_cascade_pass = DRW_pass_create("Shadow Cascade Pass", DRW_STATE_WRITE_COLOR | DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS);
	}
    ...
}
```

<br>


*eevee_materials.c*
```
void EEVEE_materials_cache_populate(EEVEE_Data *vedata, EEVEE_SceneLayerData *sldata, Object *ob)
{
    ...
    /* Shadow Pass */
    if (ma->use_nodes && ma->nodetree && (ma->blend_method != MA_BM_SOLID)) {
        struct GPUMaterial *gpumat;
        switch (ma->blend_shadow) {
            case MA_BS_SOLID:
                EEVEE_lights_cache_shcaster_add(sldata, psl, mat_geom[i], ob->obmat);
                break;
            case MA_BS_CLIP:
                gpumat = EEVEE_material_mesh_depth_get(scene, ma, false, true);
                EEVEE_lights_cache_shcaster_material_add(sldata, psl, gpumat, mat_geom[i], ob->obmat, &ma->alpha_threshold);
                break;
            case MA_BS_HASHED:
                gpumat = EEVEE_material_mesh_depth_get(scene, ma, true, true);
                EEVEE_lights_cache_shcaster_material_add(sldata, psl, gpumat, mat_geom[i], ob->obmat, NULL);
                break;
            case MA_BS_NONE:
            default:
                break;
        }
    }
    else {
        EEVEE_lights_cache_shcaster_add(sldata, psl, mat_geom[i], ob->obmat);
    }
    ...
}
```
>
- 这里分两种情况，一种不透明模型，一种是半透明模型 <br><br>
- 不透明模型 使用 EEVEE_lights_cache_shcaster_add 	<br><br>
- 半透明模型，还有根据材质上的 Transparent Shadow 来决定，Opaque， Clip，Hashed	<br><br>
- Transparent Shadow  + Opaque -> EEVEE_lights_cache_shcaster_add	<br><br>
- Transparent Shadow  + Clip -> EEVEE_material_mesh_depth_get 和 EEVEE_lights_cache_shcaster_material_add	<br><br>
- Transparent Shadow  + Hashed -> EEVEE_material_mesh_depth_get 和 EEVEE_lights_cache_shcaster_material_add	<br><br>

<br>

#### 2. 不透明模型 和 半透明 + Shadow Opaque 

*eevee_lights.c*
```

void EEVEE_lights_init(EEVEE_SceneLayerData *sldata)
{
    ...
    if (!e_data.shadow_sh) {
		e_data.shadow_sh = DRW_shader_create(
		        datatoc_shadow_vert_glsl, datatoc_shadow_geom_glsl, datatoc_shadow_frag_glsl, NULL);

		e_data.shadow_store_cube_sh = DRW_shader_create_fullscreen(datatoc_shadow_store_frag_glsl, NULL);
		e_data.shadow_store_cascade_sh = DRW_shader_create_fullscreen(datatoc_shadow_store_frag_glsl, "#define CSM");
	}
    ...

}

/* Add a shadow caster to the shadowpasses */
void EEVEE_lights_cache_shcaster_add(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, struct Gwn_Batch *geom, float (*obmat)[4])
{
	DRWShadingGroup *grp = DRW_shgroup_instance_create(e_data.shadow_sh, psl->shadow_cube_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	for (int i = 0; i < 6; ++i)
		DRW_shgroup_call_dynamic_add_empty(grp);

	grp = DRW_shgroup_instance_create(e_data.shadow_sh, psl->shadow_cascade_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	for (int i = 0; i < MAX_CASCADE_NUM; ++i)
		DRW_shgroup_call_dynamic_add_empty(grp);
}
```
>
- 当渲染 不透明模型 和 半透明+Opaque 的Shadow 使用 Shader .  <br><br>
- shadow_cube_pass 使用 shadow_vert.glsl 和 shadow_geom.glsl 和 shadow_frag.glsl <br><br>

<br>

#### 3. 半透明 + Shadow Clip  和 半透明 + Shadow Hashed

*eevee_materials.c*

```
struct GPUMaterial *EEVEE_material_mesh_depth_get(
        struct Scene *scene, Material *ma,
        bool use_hashed_alpha, bool is_shadow)
{
	const void *engine = &DRW_engine_viewport_eevee_type;
	int options = VAR_MAT_MESH;

	if (use_hashed_alpha) {
		options |= VAR_MAT_HASH;
	}
	else {
		options |= VAR_MAT_CLIP;
	}

	if (is_shadow)
		options |= VAR_MAT_SHADOW;

	GPUMaterial *mat = GPU_material_from_nodetree_find(&ma->gpumaterial, engine, options);
	if (mat) {
		return mat;
	}

	char *defines = eevee_get_defines(options);

	DynStr *ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, e_data.frag_shader_lib);
	BLI_dynstr_append(ds_frag, datatoc_prepass_frag_glsl);
	char *frag_str = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	mat = GPU_material_from_nodetree(
	        scene, ma->nodetree, &ma->gpumaterial, engine, options,
	        (is_shadow) ? datatoc_shadow_vert_glsl : datatoc_lit_surface_vert_glsl,
	        (is_shadow) ? datatoc_shadow_geom_glsl : NULL,
	        frag_str,
	        defines);

	MEM_freeN(frag_str);
	MEM_freeN(defines);

	return mat;
}


void EEVEE_lights_cache_shcaster_material_add(
	EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, struct GPUMaterial *gpumat, struct Gwn_Batch *geom, float (*obmat)[4], float *alpha_threshold)
{
	DRWShadingGroup *grp = DRW_shgroup_material_instance_create(gpumat, psl->shadow_cube_pass, geom);

	if (grp == NULL) return;

	DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	if (alpha_threshold != NULL)
		DRW_shgroup_uniform_float(grp, "alphaThreshold", alpha_threshold, 1);

	for (int i = 0; i < 6; ++i)
		DRW_shgroup_call_dynamic_add_empty(grp);

	grp = DRW_shgroup_material_instance_create(gpumat, psl->shadow_cascade_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	if (alpha_threshold != NULL)
		DRW_shgroup_uniform_float(grp, "alphaThreshold", alpha_threshold, 1);

	for (int i = 0; i < MAX_CASCADE_NUM; ++i)
		DRW_shgroup_call_dynamic_add_empty(grp);
}
```
>
- 当渲染 半透明 + Clip 和 半透明 + Hashed 的 Shadow 使用 Shader . <br><br>
- shadow_cube_pass 使用 shadow_vert.glsl 和 shadow_geom.glsl 和 prepass_frag.glsl. <br><br>

<br>

#### 4. Shader
*shadow_vert.glsl*
```
uniform mat4 ShadowModelMatrix;
#ifdef MESH_SHADER
uniform mat3 WorldNormalMatrix;
#endif

in vec3 pos;
#ifdef MESH_SHADER
in vec3 nor;
#endif

out vec4 vPos;
#ifdef MESH_SHADER
out vec3 vNor;
#endif

flat out int face;

void main() {
	vPos = ShadowModelMatrix * vec4(pos, 1.0);
	face = gl_InstanceID;

#ifdef MESH_SHADER
	vNor = WorldNormalMatrix * nor;
#ifdef ATTRIB
	pass_attrib(pos);
#endif
#endif
}
```

<br>


*shadow_geom.glsl*
```

layout(std140) uniform shadow_render_block {
	mat4 ShadowMatrix[6];
	mat4 FaceViewMatrix[6];
	vec4 lampPosition;
	float cubeTexelSize;
	float storedTexelSize;
	float nearClip;
	float farClip;
	float shadowSampleCount;
	float shadowInvSampleCount;
};

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

in vec4 vPos[];
flat in int face[];

#ifdef MESH_SHADER
in vec3 vNor[];
#endif

out vec3 worldPosition;
#ifdef MESH_SHADER
out vec3 viewPosition; /* Required. otherwise generate linking error. */
out vec3 worldNormal; /* Required. otherwise generate linking error. */
out vec3 viewNormal; /* Required. otherwise generate linking error. */
flat out int shFace;
#else
int shFace;
#endif

void main() {
	shFace = face[0];
	gl_Layer = shFace;

	for (int v = 0; v < 3; ++v) {
		gl_Position = ShadowMatrix[shFace] * vPos[v];
		worldPosition = vPos[v].xyz;
#ifdef MESH_SHADER
		worldNormal = vNor[v];
		viewPosition = (FaceViewMatrix[shFace] * vec4(worldPosition, 1.0)).xyz;
		viewNormal = (FaceViewMatrix[shFace] * vec4(worldNormal, 0.0)).xyz;
#ifdef ATTRIB
		pass_attrib(v);
#endif
#endif
		EmitVertex();
	}

	EndPrimitive();
}
```

<br>


*shadow_frag.glsl*
```
void main() {
	/* Do nothing */
}

```

>
- 实体模型 执行，不做任何其他操作，只是进行写深度


<br>

*prepass_frag.glsl*
```
#ifdef USE_ALPHA_HASH

/* From the paper "Hashed Alpha Testing" by Chris Wyman and Morgan McGuire */
float hash(vec2 a) {
	return fract(1e4 * sin(17.0 * a.x + 0.1 * a.y) * (0.1 + abs(sin(13.0 * a.y + a.x))));
}

float hash3d(vec3 a) {
	return hash(vec2(hash(a.xy), a.z));
}

//uniform float hashScale;
float hashScale = 0.001;

float hashed_alpha_threshold(vec3 co)
{
	/* Find the discretized derivatives of our coordinates. */
	float max_deriv = max(length(dFdx(co)), length(dFdy(co)));
	float pix_scale = 1.0 / (hashScale * max_deriv);

	/* Find two nearest log-discretized noise scales. */
	float pix_scale_log = log2(pix_scale);
	vec2 pix_scales;
	pix_scales.x = exp2(floor(pix_scale_log));
	pix_scales.y = exp2(ceil(pix_scale_log));

	/* Compute alpha thresholds at our two noise scales. */
	vec2 alpha;
	alpha.x = hash3d(floor(pix_scales.x * co));
	alpha.y = hash3d(floor(pix_scales.y * co));

	/* Factor to interpolate lerp with. */
	float fac = fract(log2(pix_scale));

	/* Interpolate alpha threshold from noise at two scales. */
	float x = mix(alpha.x, alpha.y, fac);

	/* Pass into CDF to compute uniformly distrib threshold. */
	float a = min(fac, 1.0 - fac);
	float one_a = 1.0 - a;
	float denom = 1.0 / (2 * a * one_a);
	float one_x = (1 - x);
	vec3 cases = vec3(
		(x * x) * denom,
		(x - 0.5 * a) / one_a,
		1.0 - (one_x * one_x * denom)
	);

	/* Find our final, uniformly distributed alpha threshold. */
	float threshold = (x < one_a) ?	((x < a) ? cases.x : cases.y) :	cases.z;

	/* Avoids threshold == 0. */
	threshold = clamp(threshold, 1.0e-6, 1.0);

	return threshold;
}

#endif

#ifdef USE_ALPHA_CLIP
uniform float alphaThreshold;
#endif

void main()
{
	/* For now do nothing.
	 * In the future, output object motion blur. */

#if defined(USE_ALPHA_HASH) || defined(USE_ALPHA_CLIP)
#define NODETREE_EXEC

	Closure cl = nodetree_exec();

#if defined(USE_ALPHA_HASH)
	/* Hashed Alpha Testing */
	if (cl.opacity < hashed_alpha_threshold(worldPosition))
		discard;
#elif defined(USE_ALPHA_CLIP)
	/* Alpha clip */
	if (cl.opacity <= alphaThreshold)
		discard;
#endif
#endif
}
```
>
- 进行Alpha Clip 和 Alpha Hashed，并且是 Nodetree 代码生成的

<br><br>

### D. shadow_cube_store_pass

#### 1. 初始化
*eevee_lights.c*
```
void EEVEE_lights_init(EEVEE_SceneLayerData *sldata)
{
    ...
    e_data.shadow_store_cube_sh = DRW_shader_create_fullscreen(datatoc_shadow_store_frag_glsl, NULL);
    ...
}


void EEVEE_lights_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
    ...
    {
		psl->shadow_cube_store_pass = DRW_pass_create("Shadow Storage Pass", DRW_STATE_WRITE_COLOR);

		DRWShadingGroup *grp = DRW_shgroup_create(e_data.shadow_store_cube_sh, psl->shadow_cube_store_pass);
		DRW_shgroup_uniform_buffer(grp, "shadowTexture", &sldata->shadow_cube_target);
		DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
		DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL);
	}
    ...
}
```
>
- shadow_cube_store_pass 使用 shadow_store_frag.glsl
- shadow_cube_store_pass 的作用是 Push it to shadowmap array
- shadow_cube_store_pass 传入了 RT shadow_cube_target, 这个RT shadow_cube_target 是由上面的 shadow_cube_pass 生成的

<br>

#### 2. Shader
```

layout(std140) uniform shadow_render_block {
	mat4 ShadowMatrix[6];
	mat4 FaceViewMatrix[6];
	vec4 lampPosition;
	float cubeTexelSize;
	float storedTexelSize;
	float nearClip;
	float farClip;
	float shadowSampleCount;
	float shadowInvSampleCount;
};

#ifdef CSM
uniform sampler2DArray shadowTexture;
#else
uniform samplerCube shadowTexture;
#endif

out vec4 FragColor;

vec3 octahedral_to_cubemap_proj(vec2 co)
{
	co = co * 2.0 - 1.0;

	vec2 abs_co = abs(co);
	vec3 v = vec3(co, 1.0 - (abs_co.x + abs_co.y));

	if ( abs_co.x + abs_co.y > 1.0 ) {
		v.xy = (abs(co.yx) - 1.0) * -sign(co.xy);
	}

	return v;
}

void make_orthonormal_basis(vec3 N, float rot, out vec3 T, out vec3 B)
{
	vec3 UpVector = (abs(N.z) < max(abs(N.x), abs(N.y))) ? vec3(0.0, 0.0, 1.0) : vec3(1.0, 0.0, 0.0);
	vec3 nT = normalize(cross(UpVector, N));
	vec3 nB = cross(N, nT);

	/* Rotate tangent space */
	vec2 dir = vec2(cos(rot * 3.1415 * 2.0), sin(rot * 3.1415 * 2.0));
	T =  dir.x * nT + dir.y * nB;
	B = -dir.y * nT + dir.x * nB;
}

float linear_depth(float z)
{
	return (nearClip  * farClip) / (z * (nearClip - farClip) + farClip);
}

float get_cube_radial_distance(vec3 cubevec)
{
	float zdepth = texture(shadowTexture, cubevec).r;
	float linear_zdepth = linear_depth(zdepth);
	cubevec = normalize(abs(cubevec));
	float cos_vec = max(cubevec.x, max(cubevec.y, cubevec.z));
	return linear_zdepth / cos_vec;
}

/* Marco Salvi's GDC 2008 presentation about shadow maps pre-filtering techniques slide 24 */
float ln_space_prefilter(float w0, float x, float w1, float y)
{
    return x + log(w0 + w1 * exp(y - x));
}

const int SAMPLE_NUM = 32;
const float INV_SAMPLE_NUM = 1.0 / float(SAMPLE_NUM);
const vec2 poisson[32] = vec2[32](
	vec2(-0.31889129888, 0.945170187163),
	vec2(0.0291070069348, 0.993645382622),
	vec2(0.453968568675, 0.882119488776),
	vec2(-0.59142811398, 0.775098624552),
	vec2(0.0672147039953, 0.677233646792),
	vec2(0.632546991242, 0.60080388224),
	vec2(-0.846282545004, 0.478266943968),
	vec2(-0.304563967348, 0.550414788876),
	vec2(0.343951542639, 0.482122717676),
	vec2(0.903371461134, 0.419225918868),
	vec2(-0.566433506581, 0.326544955645),
	vec2(-0.0174468029403, 0.345927250589),
	vec2(-0.970838848328, 0.131541221423),
	vec2(-0.317404956404, 0.102175571059),
	vec2(0.309107085158, 0.136502232088),
	vec2(0.67009683403, 0.198922062526),
	vec2(-0.62544683989, -0.0237682928336),
	vec2(0.0, 0.0),
	vec2(0.260779995092, -0.192490308513),
	vec2(0.555635503398, -0.0918935341973),
	vec2(0.989587880961, -0.03629312269),
	vec2(-0.93440130633, -0.213478602005),
	vec2(-0.615716455579, -0.335329659339),
	vec2(0.813589336772, -0.292544036149),
	vec2(-0.821106257666, -0.568279197395),
	vec2(-0.298092257627, -0.457929494012),
	vec2(0.263233114326, -0.515552889911),
	vec2(-0.0311374378304, -0.643310533036),
	vec2(0.785838482787, -0.615972502555),
	vec2(-0.444079211316, -0.836548440017),
	vec2(-0.0253421088433, -0.96112294526),
	vec2(0.350411908643, -0.89783206142)
);

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

#define ESM

void main() {
	vec2 uvs = gl_FragCoord.xy * storedTexelSize;

	/* add a 2 pixel border to ensure filtering is correct */
	uvs.xy *= 1.0 + storedTexelSize * 2.0;
	uvs.xy -= storedTexelSize;

	float pattern = 1.0;

	/* edge mirroring : only mirror if directly adjacent
	 * (not diagonally adjacent) */
	vec2 m = abs(uvs - 0.5) + 0.5;
	vec2 f = floor(m);
	if (f.x - f.y != 0.0) {
		uvs.xy = 1.0 - uvs.xy;
	}

	/* clamp to [0-1] */
	uvs.xy = fract(uvs.xy);

	/* get cubemap vector */
	vec3 cubevec = octahedral_to_cubemap_proj(uvs.xy);

/* TODO Can be optimized by groupping fetches
 * and by converting to radial distance beforehand. */
#if defined(ESM)
	vec3 T, B;
	make_orthonormal_basis(cubevec, wang_hash_noise(0u), T, B);

	T *= 0.01;
	B *= 0.01;

	float accum = 0.0;

	/* Poisson disc blur in log space. */
	float depth1 = get_cube_radial_distance(cubevec + poisson[0].x * T + poisson[0].y * B);
	float depth2 = get_cube_radial_distance(cubevec + poisson[1].x * T + poisson[1].y * B);
	accum = ln_space_prefilter(INV_SAMPLE_NUM, depth1, INV_SAMPLE_NUM, depth2);

	for (int i = 2; i < SAMPLE_NUM; ++i) {
		depth1 = get_cube_radial_distance(cubevec + poisson[i].x * T + poisson[i].y * B);
		accum = ln_space_prefilter(1.0, accum, INV_SAMPLE_NUM, depth1);
	}

	FragColor = vec4(accum);

#else /* PCF (no prefilter) */
	FragColor = vec4(get_cube_radial_distance(cubevec));
#endif
}
```
>
- 这里 将 shadow_cube_pass 步骤渲染出来的RT shadow_cube_target 进行 Push it to shadowmap array 。
- 但是这里不是简单地进行 Push，还是进行 计算 radial_distance，然后再进行 prefilter，得到 辐射距离，用于之后 影子的计算。


### E. Shadow_method 和 Shadow_size

```
void EEVEE_lights_init(EEVEE_SceneLayerData *sldata)
{
	...
	int sh_method = BKE_collection_engine_property_value_get_int(props, "shadow_method");
	int sh_size = BKE_collection_engine_property_value_get_int(props, "shadow_size");
	UNUSED_VARS(sh_method);

	EEVEE_LampsInfo *linfo = sldata->lamps;
	if (linfo->shadow_size != sh_size) {
		BLI_assert((sh_size > 0) && (sh_size <= 8192));
		DRW_TEXTURE_FREE_SAFE(sldata->shadow_pool);
		DRW_TEXTURE_FREE_SAFE(sldata->shadow_cascade_target);

		linfo->shadow_size = sh_size;
		linfo->shadow_render_data.stored_texel_size = 1.0 / (float)linfo->shadow_size;

		/* Compute adequate size for the cubemap render target.
		 * The 3.0f factor is here to make sure there is no under sampling between
		 * the octahedron mapping and the cubemap. */
		int new_cube_target_size = (int)ceil(sqrt((float)(sh_size * sh_size) / 6.0f) * 3.0f);

		CLAMP(new_cube_target_size, 1, 4096);

		if (linfo->shadow_cube_target_size != new_cube_target_size) {
			linfo->shadow_cube_target_size = new_cube_target_size;
			DRW_TEXTURE_FREE_SAFE(sldata->shadow_cube_target);
			linfo->shadow_render_data.cube_texel_size = 1.0 / (float)linfo->shadow_cube_target_size;
		}
	}
	...
}
```
>
- 这里进行修改 shadow size