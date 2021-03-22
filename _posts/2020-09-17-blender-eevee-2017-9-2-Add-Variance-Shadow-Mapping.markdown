---
layout:     post
title:      "blender eevee Add Variance Shadow Mapping"
subtitle:   ""
date:       2021-3-22 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/9/2  *   Eevee : Add Variance Shadow Mapping <br> 

> 
- This is an alternative to ESM. It does not suffer the same bleeding artifacts.


> SVN : 2017/8/17  [windows] add numpy headers to python include dir, required for D2716


## 效果
![](/img/Eevee/VarianceShadowMapping/1.png)
![](/img/Eevee/VarianceShadowMapping/2.png)

## 作用
添加 Variance Shadow Mapping 效果

<br>

## 准备
先看看 [Refactor Shadow System](http://shaderstore.cn/2021/03/21/blender-eevee-2017-9-1-Refactor-Shadow-System/)
本文章大部分都是基于这里开始的


### ESM 和 VSM 

ESM (Exponential Shadow Map) <br>
VSN (Variance Shadow Mapping) <br>

#### 1. 初始化
*eevee_private.h*
```
/* Shadow Technique */
enum {
	SHADOW_ESM = 1,
	SHADOW_VSM = 2,
	SHADOW_METHOD_MAX = 3,
};
```
<br>

*eevee_lihghts.c*
```
static struct {
	struct GPUShader *shadow_sh;
	struct GPUShader *shadow_store_cube_sh[SHADOW_METHOD_MAX];
	struct GPUShader *shadow_store_cascade_sh[SHADOW_METHOD_MAX];
} e_data = {NULL}; /* Engine data */

void EEVEE_lights_init(EEVEE_SceneLayerData *sldata)
{
	...
	if (!e_data.shadow_sh) {
		e_data.shadow_sh = DRW_shader_create(
		        datatoc_shadow_vert_glsl, datatoc_shadow_geom_glsl, datatoc_shadow_frag_glsl, NULL);

		e_data.shadow_store_cube_sh[SHADOW_ESM] = DRW_shader_create_fullscreen(datatoc_shadow_store_frag_glsl, "#define ESM\n");
		e_data.shadow_store_cascade_sh[SHADOW_ESM] = DRW_shader_create_fullscreen(datatoc_shadow_store_frag_glsl, "#define ESM\n"
		                                                                                                          "#define CSM\n");

		e_data.shadow_store_cube_sh[SHADOW_VSM] = DRW_shader_create_fullscreen(datatoc_shadow_store_frag_glsl, "#define VSM\n");
		e_data.shadow_store_cascade_sh[SHADOW_VSM] = DRW_shader_create_fullscreen(datatoc_shadow_store_frag_glsl, "#define VSM\n"
		                                                                                                          "#define CSM\n");
	}
	...
	int sh_method = BKE_collection_engine_property_value_get_int(props, "shadow_method");
	int sh_size = BKE_collection_engine_property_value_get_int(props, "shadow_size");

	EEVEE_LampsInfo *linfo = sldata->lamps;
	if ((linfo->shadow_size != sh_size) || (linfo->shadow_method != sh_method)) {
		BLI_assert((sh_size > 0) && (sh_size <= 8192));
		DRW_TEXTURE_FREE_SAFE(sldata->shadow_pool);
		DRW_TEXTURE_FREE_SAFE(sldata->shadow_cascade_target);

		linfo->shadow_method = sh_method;
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
- ESM 和 VSM 的区别在于 是用使用 shadow_store_cube_sh[SHADOW_ESM] 还是 shadow_store_cube_sh[SHADOW_VSM]
- 上一篇文章有说，shadow_store_cube_sh 主要是 shadow_cube_store_pass Pass 使用，shadow_cube_store_pass 的作用是 Push it to shadowmap array
- ESM 和 VSM 都是使用 shadow_store_frag.glsl ,但是使用的宏不一样 define ESM 和 #define VSM

<br>

```

void EEVEE_lights_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	...
	{
		psl->shadow_cube_store_pass = DRW_pass_create("Shadow Storage Pass", DRW_STATE_WRITE_COLOR);

		DRWShadingGroup *grp = DRW_shgroup_create(e_data.shadow_store_cube_sh[linfo->shadow_method], psl->shadow_cube_store_pass);
		DRW_shgroup_uniform_buffer(grp, "shadowTexture", &sldata->shadow_cube_target);
		DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
		DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL);
	}

	{
		psl->shadow_cascade_store_pass = DRW_pass_create("Shadow Cascade Storage Pass", DRW_STATE_WRITE_COLOR);

		DRWShadingGroup *grp = DRW_shgroup_create(e_data.shadow_store_cascade_sh[linfo->shadow_method], psl->shadow_cascade_store_pass);
		DRW_shgroup_uniform_buffer(grp, "shadowTexture", &sldata->shadow_cascade_target);
		DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
		DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL);
	}
	...
}
```
>
- shadow_cube_store_pass 是根据 linfo->shadow_method 来选择到底使用  shadow_store_cube_sh[SHADOW_ESM] 还是 shadow_store_cube_sh[SHADOW_VSM]

<br>

```
void EEVEE_lights_cache_finish(EEVEE_SceneLayerData *sldata)
{
	...
	switch (linfo->shadow_method) {
		case SHADOW_ESM: shadow_pool_format = DRW_TEX_R_32; break;
		case SHADOW_VSM: shadow_pool_format = DRW_TEX_RG_32; break;
		default:
			BLI_assert(!"Incorrect Shadow Method");
			break;
	}
	...
}
```
>
- SHADOW_ESM 的输出的RT的格式 是 DRW_TEX_R_32
- SHADOW_VSM 的输出的RT的格式 是 DRW_TEX_RG_32

<br>



```
static void eevee_shadow_cube_setup(Object *ob, EEVEE_LampsInfo *linfo, EEVEE_LampEngineData *led)
{
	float projmat[4][4];

	EEVEE_ShadowCubeData *evsmp = (EEVEE_ShadowCubeData *)led->storage;
	EEVEE_Light *evli = linfo->light_data + evsmp->light_id;
	EEVEE_ShadowCube *evsh = linfo->shadow_cube_data + evsmp->shadow_id;
	Lamp *la = (Lamp *)ob->data;

	perspective_m4(projmat, -la->clipsta, la->clipsta, -la->clipsta, la->clipsta, la->clipsta, la->clipend);

	for (int i = 0; i < 6; ++i) {
		float tmp[4][4];
		unit_m4(tmp);
		negate_v3_v3(tmp[3], ob->obmat[3]);
		mul_m4_m4m4(tmp, cubefacemat[i], tmp);
		mul_m4_m4m4(evsmp->viewprojmat[i], projmat, tmp);
	}

	evsh->bias = 0.05f * la->bias;
	evsh->near = la->clipsta;
	evsh->far = la->clipend;
	evsh->exp = (linfo->shadow_method == SHADOW_VSM) ? la->bleedbias : la->bleedexp;

	evli->shadowid = (float)(evsmp->shadow_id);
}

```
>
- evsh->exp 参数也是根据 shadow_method  来变化

<br>




### Shader

#### 1. shadow_store_frag
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
#if defined(ESM) || defined(VSM)
	vec3 T, B;
	make_orthonormal_basis(cubevec, wang_hash_noise(0u), T, B);

	T *= 0.01;
	B *= 0.01;

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
```
>
- 主要是  defined(ESM) || defined(VSM) 的部分，个人理解，最是都是不同的方式去计算 radial_distance ，然后进行柔和处理

<br><br>

#### 2. lit_surface_frag.glsl

lit_surface_frag.glsl 代码中会调用  light_visibility 函数

<br>

*lamps_lib.glsl*
```
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
	if (ld.l_shadowid >= MAX_SHADOW_CUBE) {
		vis *= shadow_cascade(ld.l_shadowid, W);
	}
	else if (ld.l_shadowid >= 0.0) {
		vis *= shadow_cubemap(ld.l_shadowid, l_vector);
	}
#endif

	return vis;
}



float shadow_cubemap(float shid, vec4 l_vector)
{
	ShadowCubeData scd = shadows_cube_data[int(shid)];

	vec3 cubevec = -l_vector.xyz / l_vector.w;
	float dist = l_vector.w;

#if defined(SHADOW_VSM)
	vec2 moments = texture_octahedron(shadowTexture, vec4(cubevec, shid)).rg;
	float p = 0.0;

	if (dist <= moments.x)
		p = 1.0;

	float variance = moments.y - (moments.x * moments.x);
	variance = max(variance, scd.sh_cube_bias / 10.0);

	float d = moments.x - dist;
	float p_max = variance / (variance + d * d);

	// Now reduce light-bleeding by removing the [0, x] tail and linearly rescaling (x, 1]
	p_max = clamp((p_max - scd.sh_cube_bleed) / (1.0 - scd.sh_cube_bleed), 0.0, 1.0);

	float vsm_test = max(p, p_max);
	return vsm_test;

#elif defined(SHADOW_ESM)
	float z = texture_octahedron(shadowTexture, vec4(cubevec, shid)).r;
	float esm_test = saturate(exp(scd.sh_cube_exp * (z - (dist - scd.sh_cube_bias))));
	return esm_test;

#else
	float z = texture_octahedron(shadowTexture, vec4(cubevec, shid)).r;
	float sh_test = step(0, z - (dist - scd.sh_cube_bias));
	return sh_test;
#endif
}


```
>
- light_visibility 函数调用 shadow_cubemap
- shadow_cubemap 的才是计算影子的核心代码


<br><br>

# Expose Shadow filter size

## 来源

- 主要看这个commit

> GIT : 2017/9/2  *   Eevee : Expose Shadow filter size. <br> 

> SVN : 2017/8/17  [windows] add numpy headers to python include dir, required for D2716


## 效果

![](/img/Eevee/VarianceShadowMapping/3.png)

## 作用
Shadow 添加 Smooth 参数可以调整


### Shader

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
	float shadowFilterSize;
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
```
>
- 留意 shadowFilterSize 参数基本就知道是怎么实现的了


<br>

### Shader 参数


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
			srd->filter_size = la->soft * 0.0005f;
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
- 最最最重要的是 这一行 srd->filter_size = la->soft * 0.0005f;

<br><br>


# Shadow Add high bitdepth option

## 来源

- 主要看这个commit

> GIT : 2017/9/2  *  Eevee : Shadow: Add high bitdepth option <br> 

> This option is here for reducing the memory usage of shadow maps.
- Also lower bitdepth are quicker to process.


> SVN : 2017/8/17  [windows] add numpy headers to python include dir, required for D2716


## 效果

![](/img/Eevee/VarianceShadowMapping/4.png)

## 作用
Shadow 添加 High Bitdepth 参数可以调整 rt 的格式


### 实现

*eevee_lights.c*
```
void EEVEE_lights_init(EEVEE_SceneLayerData *sldata)
{
	...
	int sh_method = BKE_collection_engine_property_value_get_int(props, "shadow_method");
	int sh_size = BKE_collection_engine_property_value_get_int(props, "shadow_size");
	int sh_high_bitdepth = BKE_collection_engine_property_value_get_int(props, "shadow_high_bitdepth");

	EEVEE_LampsInfo *linfo = sldata->lamps;
	if ((linfo->shadow_size != sh_size) ||
		(linfo->shadow_method != sh_method) ||
		(linfo->shadow_high_bitdepth != sh_high_bitdepth))
	{
		BLI_assert((sh_size > 0) && (sh_size <= 8192));
		DRW_TEXTURE_FREE_SAFE(sldata->shadow_pool);
		DRW_TEXTURE_FREE_SAFE(sldata->shadow_cube_target);
		DRW_TEXTURE_FREE_SAFE(sldata->shadow_cascade_target);

		linfo->shadow_high_bitdepth = sh_high_bitdepth;
		linfo->shadow_method = sh_method;
		linfo->shadow_size = sh_size;
		linfo->shadow_render_data.stored_texel_size = 1.0 / (float)linfo->shadow_size;

		/* Compute adequate size for the cubemap render target.
		 * The 3.0f factor is here to make sure there is no under sampling between
		 * the octahedron mapping and the cubemap. */
		int new_cube_target_size = (int)ceil(sqrt((float)(sh_size * sh_size) / 6.0f) * 3.0f);

		CLAMP(new_cube_target_size, 1, 4096);

		linfo->shadow_cube_target_size = new_cube_target_size;
		linfo->shadow_render_data.cube_texel_size = 1.0 / (float)linfo->shadow_cube_target_size;
	}
	...
}


void EEVEE_lights_cache_finish(EEVEE_SceneLayerData *sldata)
{
	...
	switch (linfo->shadow_method) {
		case SHADOW_ESM: shadow_pool_format = ((linfo->shadow_high_bitdepth) ? DRW_TEX_R_32 : DRW_TEX_R_16); break;
		case SHADOW_VSM: shadow_pool_format = ((linfo->shadow_high_bitdepth) ? DRW_TEX_RG_32 : DRW_TEX_RG_16); break;
		default:
			BLI_assert(!"Incorrect Shadow Method");
			break;
	}
	...
}
```
>
- 这里比较清楚了, 如果点击了 shadow_high_bitdepth 的话，RT 就用 32 位, 否则就用 16 位。