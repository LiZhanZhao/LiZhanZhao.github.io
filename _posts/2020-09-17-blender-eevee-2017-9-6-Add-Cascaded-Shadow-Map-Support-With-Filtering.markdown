---
layout:     post
title:      "blender eevee Add Cascaded Shadow Map support with filtering"
subtitle:   ""
date:       2021-3-23 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/9/6  *   Eevee : Add Cascaded Shadow Map support with filtering <br> 

> This brings some data structure changes.
Shared shadow data are stored in ShadowData (in glsl) (aka EEVEE_Shadow in C).
This structure contains the array indices of the first shadow element of this shadow "object".
It also contains how many shadow to evaluate (to be used for Multiple shadow maps).

> The filtering is noisy and needs improvement.



> SVN : 2017/8/17  [windows] add numpy headers to python include dir, required for D2716


## 效果
![](/img/Eevee/CSM/02//1.png)

## 作用
添加 Cascaded Shadow Map 效果


## 准备
先阅读  [Cascaded Shadow Maps](http://shaderstore.cn/2020/07/02/blender-eevee-2017-4-21-eevee-Cascaded-Shadow-Maps/)
和 [Refactor Shadow System](http://shaderstore.cn/2021/03/21/blender-eevee-2017-9-1-Refactor-Shadow-System/)


<br>


### 渲染

#### 1. 添加灯光的一些变量
*eevee_lights.c*
```
void EEVEE_lights_cache_add(EEVEE_SceneLayerData *sldata, Object *ob)
{
	...
	if (la->type == LA_SUN)	{
		int sh_nbr = 1; /* TODO : MSM */
		int cascade_nbr = MAX_CASCADE_NUM; /* TODO : Custom cascade number */

		if ((linfo->gpu_cascade_ct + sh_nbr) <= MAX_SHADOW_CASCADE) {
			/* Save Light object. */
			linfo->shadow_cascade_ref[linfo->cpu_cascade_ct] = ob;

			/* Create storage and store indices. */
			EEVEE_ShadowCascadeData *data = MEM_mallocN(sizeof(EEVEE_ShadowCascadeData), "EEVEE_ShadowCascadeData");
			data->shadow_id = linfo->gpu_shadow_ct;
			data->cascade_id = linfo->gpu_cascade_ct;
			data->layer_id = linfo->num_layer;
			led->storage = data;

			/* Increment indices. */
			linfo->gpu_shadow_ct += 1;
			linfo->gpu_cascade_ct += sh_nbr;
			linfo->num_layer += sh_nbr * cascade_nbr;

			linfo->cpu_cascade_ct += 1;
		}
	}
	else if (la->type == LA_SPOT || la->type == LA_LOCAL || la->type == LA_AREA) {
		int sh_nbr = 1; /* TODO : MSM */

		if ((linfo->gpu_cube_ct + sh_nbr) <= MAX_SHADOW_CUBE) {
			/* Save Light object. */
			linfo->shadow_cube_ref[linfo->cpu_cube_ct] = ob;

			/* Create storage and store indices. */
			EEVEE_ShadowCubeData *data = MEM_mallocN(sizeof(EEVEE_ShadowCubeData), "EEVEE_ShadowCubeData");
			data->shadow_id = linfo->gpu_shadow_ct;
			data->cube_id = linfo->gpu_cube_ct;
			data->layer_id = linfo->num_layer;
			led->storage = data;

			/* Increment indices. */
			linfo->gpu_shadow_ct += 1;
			linfo->gpu_cube_ct += sh_nbr;
			linfo->num_layer += sh_nbr;

			linfo->cpu_cube_ct += 1;
		}
	}
	...
}
```
>
- linfo->gpu_shadow_ct 记录的是 有多少个光源产生影子
- linfo->gpu_cascade_ct 记录的是 有多少个光源产生cascade影子
- data->layer_id 记录的是，这个光源的影子会用到 shadow_pool RT array 的第几个。
- linfo->num_layer 记录的是，这个影子会使用 shadow_pool RT array 多少个 RT。

<br><br>


#### 2. 渲染入口
*eevee_lights.c*
```
/* this refresh lamps shadow buffers */
void EEVEE_draw_shadows(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	...
	/* Cascaded Shadow Maps */
	DRW_framebuffer_texture_attach(sldata->shadow_target_fb, sldata->shadow_cascade_target, 0, 0);
	for (i = 0; (ob = linfo->shadow_cascade_ref[i]) && (i < MAX_SHADOW_CASCADE); i++) {
		EEVEE_LampEngineData *led = EEVEE_lamp_data_get(ob);
		Lamp *la = (Lamp *)ob->data;

		EEVEE_ShadowCascadeData *evscd = (EEVEE_ShadowCascadeData *)led->storage;
		EEVEE_ShadowRender *srd = &linfo->shadow_render_data;

		srd->shadow_samples_ct = 32.0f;
		srd->shadow_inv_samples_ct = 1.0f / srd->shadow_samples_ct;
		srd->clip_near = la->clipsta;
		srd->clip_far = la->clipend;
		for (int j = 0; j < MAX_CASCADE_NUM; ++j) {
			copy_m4_m4(srd->shadowmat[j], evscd->viewprojmat[j]);
		}
		DRW_uniformbuffer_update(sldata->shadow_render_ubo, &linfo->shadow_render_data);

		DRW_framebuffer_bind(sldata->shadow_target_fb);
		DRW_framebuffer_clear(false, true, false, NULL, 1.0);

		/* Render shadow cascades */
		DRW_draw_pass(psl->shadow_cascade_pass);

		for (linfo->current_shadow_cascade = 0;
		     linfo->current_shadow_cascade < MAX_CASCADE_NUM;
		     ++linfo->current_shadow_cascade)
		{
			linfo->filter_size = la->soft * 0.0005f / (evscd->radius[linfo->current_shadow_cascade] * 0.05f);

			/* Push it to shadowmap array */
			int layer = evscd->layer_id + linfo->current_shadow_cascade;
			DRW_framebuffer_texture_layer_attach(sldata->shadow_store_fb, sldata->shadow_pool, 0, layer, 0);
			DRW_framebuffer_bind(sldata->shadow_store_fb);
			DRW_draw_pass(psl->shadow_cascade_store_pass);
		}
	}

	DRW_framebuffer_texture_detach(sldata->shadow_cube_target);
	...
}
```
>
- 先用 shadow_cascade_pass 把东西渲染到 RT shadow_cascade_target 上
- 再用 shadow_cascade_store_pass 把 shadow_cascade_target 添加到 shadow_pool RT 的 evscd->layer_id + linfo->current_shadow_cascade 索引上，添加4个
- MAX_SHADOW_CASCADE 是最多有多少个方向光产生 Shadow Cascade 
- MAX_CASCADE_NUM 是 一个 Shadow Cascade  会多少 段, 目前是4段，用到了4个RT

<br><br>

#### 2.渲染贴图

*eevee_lights.c*
```
if (!sldata->shadow_cube_target) {
	/* TODO render everything on the same 2d render target using clip planes and no Geom Shader. */
	/* Cubemaps */
	sldata->shadow_cube_target = DRW_texture_create_cube(linfo->shadow_cube_target_size, DRW_TEX_DEPTH_24, 0, NULL);
}

if (!sldata->shadow_cascade_target) {
	/* CSM */
	sldata->shadow_cascade_target = DRW_texture_create_2D_array(
			linfo->shadow_size, linfo->shadow_size, MAX_CASCADE_NUM, DRW_TEX_DEPTH_24, 0, NULL);
}

/* Initialize Textures Array first so DRW_framebuffer_init just bind them. */
	if (!sldata->shadow_pool) {
		/* All shadows fit in this array */
		sldata->shadow_pool = DRW_texture_create_2D_array(
		        linfo->shadow_size, linfo->shadow_size, max_ff(1, linfo->num_layer),
		        shadow_pool_format, DRW_TEX_FILTER, NULL);
	}

```
>
- RT shadow_cascade_target 是一个 textureArray
- shadow_pool RT 也是一个 textureArray，但是这个 shadow_pool RT 在这里除了了存储 RT shadow_cascade_target ，还会存储 shadow_cube_target RT, 这个就要留意一下  [这里](http://shaderstore.cn/2021/03/21/blender-eevee-2017-9-1-Refactor-Shadow-System/)


<br><br>

### shadow_cascade_pass

#### 1. 初始化
*eevee_lights.c*
```
void EEVEE_lights_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	...
	{
		psl->shadow_cascade_pass = DRW_pass_create("Shadow Cascade Pass", DRW_STATE_WRITE_COLOR | DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_LESS);
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

- 不透明模型 和 半透明 + Shadow Opaque : shadow_cascade_pass 使用 shadow_vert.glsl 和 shadow_geom.glsl 和 shadow_frag.glsl
- 半透明 + Shadow Clip 和 半透明 + Shadow Hashed : shadow_cascade_pass 使用 shadow_vert.glsl 和 shadow_geom.glsl 和 prepass_frag.glsl.
- shadow_cascade_pass 会实例化渲染 MAX_CASCADE_NUM 次

<br><br>


#### 2. Shader
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
>
- ShadowModelMatrix 是灯光的世界坐标矩阵， DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);
- face = gl_InstanceID; 由于 shadow_cascade_pass 会实例化渲染 MAX_CASCADE_NUM 次，这里 face 就是 0-3，但是这里 shadow_cascade_pass 渲染的是 Texture Array, 所以只是渲染到 Texture Array 上

<br><br>

*shadow_geom.glsl*
```
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
>
- ShadowMatrix[shFace] 是下面进行计算的，CSM每一段的 灯光投影矩阵


<br><br>


#### 3. Shader 参数
*eevee_lights.c*
```
static void eevee_shadow_cascade_setup(Object *ob, EEVEE_LampsInfo *linfo, EEVEE_LampEngineData *led)
{
	/* Camera Matrices */
	float persmat[4][4], persinv[4][4];
	float viewprojmat[4][4], projinv[4][4];
	float near, far;
	float near_v[4] = {0.0f, 0.0f, -1.0f, 1.0f};
	float far_v[4] = {0.0f, 0.0f,  1.0f, 1.0f};
	bool is_persp = DRW_viewport_is_persp_get();
	DRW_viewport_matrix_get(persmat, DRW_MAT_PERS);
	invert_m4_m4(persinv, persmat);
	/* FIXME : Get near / far from Draw manager? */
	DRW_viewport_matrix_get(viewprojmat, DRW_MAT_WIN);
	invert_m4_m4(projinv, viewprojmat);
	mul_m4_v4(projinv, near_v);
	mul_m4_v4(projinv, far_v);
	near = near_v[2];
	far = far_v[2]; /* TODO: Should be a shadow parameter */
	if (is_persp) {
		near /= near_v[3];
		far /= far_v[3];
	}

	/* Lamps Matrices */
	float viewmat[4][4], projmat[4][4];
	int sh_nbr = 1; /* TODO : MSM */
	int cascade_nbr = MAX_CASCADE_NUM; /* TODO : Custom cascade number */

	EEVEE_ShadowCascadeData *sh_data = (EEVEE_ShadowCascadeData *)led->storage;
	EEVEE_Light *evli = linfo->light_data + sh_data->light_id;
	EEVEE_Shadow *ubo_data = linfo->shadow_data + sh_data->shadow_id;
	EEVEE_ShadowCascade *cascade_data = linfo->shadow_cascade_data + sh_data->cascade_id;
	Lamp *la = (Lamp *)ob->data;

	/* The technique consists into splitting
	 * the view frustum into several sub-frustum
	 * that are individually receiving one shadow map */

	/* init near/far */
	for (int c = 0; c < MAX_CASCADE_NUM; ++c) {
		cascade_data->split[c] = far;
	}

	/* Compute split planes */
	float splits_ndc[MAX_CASCADE_NUM + 1];
	splits_ndc[0] = -1.0f;
	splits_ndc[cascade_nbr] = 1.0f;
	for (int c = 1; c < cascade_nbr; ++c) {
		const float lambda = 0.8f; /* TODO : Parameter */

		/* View Space */
		float linear_split = LERP(((float)(c) / (float)cascade_nbr), near, far);
		float exp_split = near * powf(far / near, (float)(c) / (float)cascade_nbr);

		if (is_persp) {
			cascade_data->split[c-1] = LERP(lambda, linear_split, exp_split);
		}
		else {
			cascade_data->split[c-1] = linear_split;
		}

		/* NDC Space */
		float p[4] = {1.0f, 1.0f, cascade_data->split[c-1], 1.0f};
		mul_m4_v4(viewprojmat, p);
		splits_ndc[c] = p[2];

		if (is_persp) {
			splits_ndc[c] /= p[3];
		}
	}

	/* For each cascade */
	for (int c = 0; c < cascade_nbr; ++c) {
		/* Given 8 frustum corners */
		float corners[8][4] = {
			/* Near Cap */
			{-1.0f, -1.0f, splits_ndc[c], 1.0f},
			{ 1.0f, -1.0f, splits_ndc[c], 1.0f},
			{-1.0f,  1.0f, splits_ndc[c], 1.0f},
			{ 1.0f,  1.0f, splits_ndc[c], 1.0f},
			/* Far Cap */
			{-1.0f, -1.0f, splits_ndc[c+1], 1.0f},
			{ 1.0f, -1.0f, splits_ndc[c+1], 1.0f},
			{-1.0f,  1.0f, splits_ndc[c+1], 1.0f},
			{ 1.0f,  1.0f, splits_ndc[c+1], 1.0f}
		};

		/* Transform them into world space */
		for (int i = 0; i < 8; ++i)	{
			mul_m4_v4(persinv, corners[i]);
			mul_v3_fl(corners[i], 1.0f / corners[i][3]);
			corners[i][3] = 1.0f;
		}

		/* Project them into light space */
		invert_m4_m4(viewmat, ob->obmat);
		normalize_v3(viewmat[0]);
		normalize_v3(viewmat[1]);
		normalize_v3(viewmat[2]);

		for (int i = 0; i < 8; ++i)	{
			mul_m4_v4(viewmat, corners[i]);
		}

		float center[3];
		frustum_min_bounding_sphere(corners, center, &(sh_data->radius[c]));

		/* Snap projection center to nearest texel to cancel shimmering. */
		float shadow_origin[2], shadow_texco[2];
		mul_v2_v2fl(shadow_origin, center, linfo->shadow_size / (2.0f * sh_data->radius[c])); /* Light to texture space. */

		/* Find the nearest texel. */
		shadow_texco[0] = round(shadow_origin[0]);
		shadow_texco[1] = round(shadow_origin[1]);

		/* Compute offset. */
		sub_v2_v2(shadow_texco, shadow_origin);
		mul_v2_fl(shadow_texco, (2.0f * sh_data->radius[c]) / linfo->shadow_size); /* Texture to light space. */

		/* Apply offset. */
		add_v2_v2(center, shadow_texco);

		/* Expand the projection to cover frustum range */
		orthographic_m4(projmat,
		                center[0] - sh_data->radius[c],
		                center[0] + sh_data->radius[c],
		                center[1] - sh_data->radius[c],
		                center[1] + sh_data->radius[c],
		                la->clipsta, la->clipend);

		mul_m4_m4m4(sh_data->viewprojmat[c], projmat, viewmat);
		mul_m4_m4m4(cascade_data->shadowmat[c], texcomat, sh_data->viewprojmat[c]);
	}

	ubo_data->bias = 0.05f * la->bias;
	ubo_data->near = la->clipsta;
	ubo_data->far = la->clipend;
	ubo_data->exp = (linfo->shadow_method == SHADOW_VSM) ? la->bleedbias : la->bleedexp;

	evli->shadowid = (float)(sh_data->shadow_id);
	ubo_data->shadow_start = (float)(sh_data->layer_id);
	ubo_data->data_start = (float)(sh_data->cascade_id);
	ubo_data->multi_shadow_count = (float)(sh_nbr);
}
```
>
- 这里计算出来的 sh_data->viewprojmat[c] 是CSM每一段的灯光投影矩阵
- 这里可以参考 [Cascaded Shadow Maps](http://shaderstore.cn/2020/07/02/blender-eevee-2017-4-21-eevee-Cascaded-Shadow-Maps/)


<br><br>


#### 4. 应用
*lamps_lib.glsl*
```
/* ----------------------------------------------------------- */
/* ----------------------- Shadow tests ---------------------- */
/* ----------------------------------------------------------- */

float shadow_test_esm(float z, float dist, float exponent)
{
	return saturate(exp(exponent * (z - dist)));
}

float shadow_test_pcf(float z, float dist)
{
	return step(0, z - dist);
}

float shadow_test_vsm(vec2 moments, float dist, float bias, float bleed_bias)
{
	float p = 0.0;

	if (dist <= moments.x)
		p = 1.0;

	float variance = moments.y - (moments.x * moments.x);
	variance = max(variance, bias / 10.0);

	float d = moments.x - dist;
	float p_max = variance / (variance + d * d);

	/* Now reduce light-bleeding by removing the [0, x] tail and linearly rescaling (x, 1] */
	p_max = clamp((p_max - bleed_bias) / (1.0 - bleed_bias), 0.0, 1.0);

	return max(p, p_max);
}


/* ----------------------------------------------------------- */
/* ----------------------- Shadow types ---------------------- */
/* ----------------------------------------------------------- */

float shadow_cubemap(ShadowData sd, ShadowCubeData scd, float texid, vec3 W)
{
	vec3 cubevec = W - scd.position.xyz;
	float dist = length(cubevec);

	cubevec /= dist;

#if defined(SHADOW_VSM)
	vec2 moments = texture_octahedron(shadowTexture, vec4(cubevec, texid)).rg;
#else
	float z = texture_octahedron(shadowTexture, vec4(cubevec, texid)).r;
#endif

#if defined(SHADOW_VSM)
	return shadow_test_vsm(moments, dist, sd.sh_bias, sd.sh_bleed);
#elif defined(SHADOW_ESM)
	return shadow_test_esm(z, dist - sd.sh_bias, sd.sh_exp);
#else
	return shadow_test_pcf(z, dist - sd.sh_bias);
#endif
}

float shadow_cascade(ShadowData sd, ShadowCascadeData scd, float texid, vec3 W)
{
	/* Finding Cascade index */
	vec4 view_z = vec4(dot(W - cameraPos, cameraForward));
	vec4 comp = step(view_z, scd.split_distances);
	float cascade = dot(comp, comp);
	mat4 shadowmat;

	/* Manual Unrolling of a loop for better performance.
	 * Doing fetch directly with cascade index leads to
	 * major performance impact. (0.27ms -> 10.0ms for 1 light) */
	if (cascade == 0.0) {
		shadowmat = scd.shadowmat[0];
	}
	else if (cascade == 1.0) {
		shadowmat = scd.shadowmat[1];
	}
	else if (cascade == 2.0) {
		shadowmat = scd.shadowmat[2];
	}
	else {
		shadowmat = scd.shadowmat[3];
	}

	vec4 shpos = shadowmat * vec4(W, 1.0);
	float dist = shpos.z * abs(sd.sh_far - sd.sh_near); /* Same factor as in get_cascade_world_distance(). */

#if defined(SHADOW_VSM)
	vec2 moments = texture(shadowTexture, vec3(shpos.xy, texid + cascade)).rg;
#else
	float z = texture(shadowTexture, vec3(shpos.xy, texid + cascade)).r;
#endif

#if defined(SHADOW_VSM)
	return shadow_test_vsm(moments, dist, sd.sh_bias, sd.sh_bleed);
#elif defined(SHADOW_ESM)
	return shadow_test_esm(z, dist - sd.sh_bias, sd.sh_exp);
#else
	return shadow_test_pcf(z, dist - sd.sh_bias);
#endif
}

/* ----------------------------------------------------------- */
/* --------------------- Light Functions --------------------- */
/* ----------------------------------------------------------- */
#define MAX_MULTI_SHADOW 4

float light_visibility(LightData ld, vec3 W, vec4 l_vector)
{
	float vis = 1.0;

	if (ld.l_type == SPOT) {
		float z = dot(ld.l_forward, l_vector.xyz);
		vec3 lL = l_vector.xyz / z;
		float x = dot(ld.l_right, lL) / ld.l_sizex;
		float y = dot(ld.l_up, lL) / ld.l_sizey;

		float ellipse = 1.0 / sqrt(1.0 + x * x + y * y);

		float spotmask = smoothstep(0.0, 1.0, (ellipse - ld.l_spot_size) / ld.l_spot_blend);

		vis *= spotmask;
		vis *= step(0.0, -dot(l_vector.xyz, ld.l_forward));
	}
	else if (ld.l_type == AREA) {
		vis *= step(0.0, -dot(l_vector.xyz, ld.l_forward));
	}

#if !defined(VOLUMETRICS) || defined(VOLUME_SHADOW)
	/* shadowing */
	if (ld.l_shadowid >= 0.0) {
		ShadowData data = shadows_data[int(ld.l_shadowid)];
		if (ld.l_type == SUN) {
			/* TODO : MSM */
			// for (int i = 0; i < MAX_MULTI_SHADOW; ++i) {
				vis *= shadow_cascade(
					data, shadows_cascade_data[int(data.sh_data_start)],
					data.sh_tex_start, W);
			// }
		}
		else {
			/* TODO : MSM */
			// for (int i = 0; i < MAX_MULTI_SHADOW; ++i) {
				vis *= shadow_cubemap(
					data, shadows_cube_data[int(data.sh_data_start)],
					data.sh_tex_start, W);
			// }
		}
	}
#endif

	return vis;
}

```
>
- 主要关注 shadow_cascade 函数，计算的 cascade 是确定属于那一段分段，然后再去对应的 RT 和灯光矩阵

<br><br>


### shadow_cascade_store_pass

#### 1. 初始化

```
void EEVEE_lights_init(EEVEE_SceneLayerData *sldata)
{
	...
	e_data.shadow_store_cascade_sh[SHADOW_ESM] = DRW_shader_create_fullscreen(datatoc_shadow_store_frag_glsl, "#define ESM\n"
		                                                                                                          "#define CSM\n");
	...

	e_data.shadow_store_cascade_sh[SHADOW_VSM] = DRW_shader_create_fullscreen(datatoc_shadow_store_frag_glsl, "#define VSM\n"
		                                                                                                          "#define CSM\n");

}

void EEVEE_lights_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	...
	
	{
		psl->shadow_cascade_store_pass = DRW_pass_create("Shadow Cascade Storage Pass", DRW_STATE_WRITE_COLOR);

		DRWShadingGroup *grp = DRW_shgroup_create(e_data.shadow_store_cascade_sh[linfo->shadow_method], psl->shadow_cascade_store_pass);
		DRW_shgroup_uniform_buffer(grp, "shadowTexture", &sldata->shadow_cascade_target);
		DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
		DRW_shgroup_uniform_int(grp, "cascadeId", &linfo->current_shadow_cascade, 1);
		DRW_shgroup_uniform_float(grp, "shadowFilterSize", &linfo->filter_size, 1);
		DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL);
	}
	...
}
```
>
- shadow_cascade_store_pass 使用 shadow_store_frag.glsl, 但是会根据 ESM 和 VSM 来决定，还有开启的宏 CSM



#### 2. Shader
*shadow_store_frag.glsl*
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
uniform int cascadeId;
#else
uniform samplerCube shadowTexture;
#endif
uniform float shadowFilterSize;

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
	vec3 UpVector = (abs(N.z) < 0.999) ? vec3(0.0, 0.0, 1.0) : vec3(1.0, 0.0, 0.0);
	vec3 nT = normalize(cross(UpVector, N));
	vec3 nB = cross(N, nT);

	/* Rotate tangent space */
	float angle = rot * 3.1415 * 2.0;
	vec2 dir = vec2(cos(angle), sin(angle));
	T =  dir.x * nT + dir.y * nB;
	B = -dir.y * nT + dir.x * nB;
}

float linear_depth(float z)
{
	return (nearClip  * farClip) / (z * (nearClip - farClip) + farClip);
}

#ifdef CSM
float get_cascade_world_distance(vec2 uvs)
{
	float zdepth = texture(shadowTexture, vec3(uvs, float(cascadeId))).r;
	return zdepth * abs(farClip - nearClip); /* Same factor as in shadow_cascade(). */
}
#else
float get_cube_radial_distance(vec3 cubevec)
{
	float zdepth = texture(shadowTexture, cubevec).r;
	float linear_zdepth = linear_depth(zdepth);
	cubevec = normalize(abs(cubevec));
	float cos_vec = max(cubevec.x, max(cubevec.y, cubevec.z));
	return linear_zdepth / cos_vec;
}
#endif

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

#ifdef CSM
void main() {
	vec2 uvs = gl_FragCoord.xy * storedTexelSize;

	vec2 X, Y;
	X.x = cos(wang_hash_noise(0u) * 3.1415 * 2.0);
	X.y = sqrt(1.0 - X.x * X.x);

	Y = vec2(-X.y, X.x);

	X *= shadowFilterSize;
	Y *= shadowFilterSize;

/* TODO Can be optimized by groupping fetches
 * and by converting to world distance beforehand. */
#if defined(ESM) || defined(VSM)
#ifdef ESM
	float accum = 0.0;

	/* Poisson disc blur in log space. */
	float depth1 = get_cascade_world_distance(uvs + X * poisson[0].x + Y * poisson[0].y);
	float depth2 = get_cascade_world_distance(uvs + X * poisson[1].x + Y * poisson[1].y);
	accum = ln_space_prefilter(INV_SAMPLE_NUM, depth1, INV_SAMPLE_NUM, depth2);

	for (int i = 2; i < SAMPLE_NUM; ++i) {
		depth1 = get_cascade_world_distance(uvs + X * poisson[i].x + Y * poisson[i].y);
		accum = ln_space_prefilter(1.0, accum, INV_SAMPLE_NUM, depth1);
	}

	FragColor = vec4(accum);
#else /* VSM */
	vec2 accum = vec2(0.0);

	/* Poisson disc blur. */
	for (int i = 0; i < SAMPLE_NUM; ++i) {
		float dist = get_cascade_world_distance(uvs + X * poisson[i].x + Y * poisson[i].y);
		float dist_sqr = dist * dist;
		accum += vec2(dist, dist_sqr);
	}

	FragColor = accum.xyxy * shadowInvSampleCount;
#endif /* Prefilter */
#else /* PCF (no prefilter) */
	FragColor = vec4(get_cascade_world_distance(uvs));
#endif
}

#else /* CUBEMAP */

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
	vec3 cubevec = normalize(octahedral_to_cubemap_proj(uvs.xy));

/* TODO Can be optimized by groupping fetches
 * and by converting to radial distance beforehand. */
#if defined(ESM) || defined(VSM)
	vec3 T, B;
	make_orthonormal_basis(cubevec, wang_hash_noise(0u), T, B);

	T *= shadowFilterSize;
	B *= shadowFilterSize;

#ifdef ESM
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
#else /* VSM */
	vec2 accum = vec2(0.0);

	/* Poisson disc blur. */
	for (int i = 0; i < SAMPLE_NUM; ++i) {
		float dist = get_cube_radial_distance(cubevec + poisson[i].x * T + poisson[i].y * B);
		float dist_sqr = dist * dist;
		accum += vec2(dist, dist_sqr);
	}

	FragColor = accum.xyxy * shadowInvSampleCount;
#endif /* Prefilter */
#else /* PCF (no prefilter) */
	FragColor = vec4(get_cube_radial_distance(cubevec));
#endif
}
#endif
```
>
- 主要留意 CSM 宏定义那一块，get_cascade_world_distance 和 main 函数的不一样
- 这里需要留意的是，方向光 CSM 保存的RT 并不是 cubemap，而是普通 texture 2D Array，这里的pass 就是把texture 2D Array 加入到 shadow_pool RT 的对应的index 上 之后的4个 rt 上， 而 其他的光源保存的是 cubemap texture，然后也是加入到 shadow_pool RT 的对应的index 的1 个 rt 上

<br><br>

### 总结
- shadow_cascade_pass Pass 主要是渲染场景的深度到一个 texture Array 中，实例化渲染，渲染多少次，取决于把这个场景分开多少段，每一个段渲染出来的深度都对应到 texture array 上的某一个索引上.
<br><br>
- CSM 会把 视锥体分开几块，每一块构建一个 正交矩阵 ，保证正交矩阵是可以包围这个区域，对应的多个矩阵,可以参考 [Cascaded Shadow Maps](http://shaderstore.cn/2020/07/02/blender-eevee-2017-4-21-eevee-Cascaded-Shadow-Maps/)
<br><br>
- shadow_cascade_store_pass Pass 就把 以上的 texture Array 再保存到另一张 shadow_pool RT上，shadow_pool RT 也是 texture Array，之后物体计算影子的都是用到 shadow_pool RT 
<br><br>
- 计算影子的时候, 会进行判断物体在CSM的哪一段里面，然后再进行取对应的那一个RT
