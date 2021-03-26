---
layout:     post
title:      "blender eevee Shadows Filtering improvement"
subtitle:   ""
date:       2021-3-26 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/9/9  *   Eevee : Shadows: Filtering improvement.<br> 

>
- Replace poisson by concentric samples: Less variance. They are sorted by radius then by angle.
- Separate filtering into 2 blur. First blur is 3x3 box blur. Second is user dependant.
- Group fetches by group of 4.


> SVN : 2017/8/17  [windows] add numpy headers to python include dir, required for D2716


## 效果
![](/img/Eevee/Shadow-3/1.png)

## 作用
优化影子的过滤效果

<br>

### 渲染入口

*eevee_lights.c*

```
void EEVEE_lights_cache_finish(EEVEE_SceneLayerData *sldata)
{
	...
	if (!sldata->shadow_cube_target) {
		/* TODO render everything on the same 2d render target using clip planes and no Geom Shader. */
		/* Cubemaps */
		sldata->shadow_cube_target = DRW_texture_create_cube(linfo->shadow_cube_target_size, DRW_TEX_DEPTH_24, 0, NULL);
		sldata->shadow_cube_blur = DRW_texture_create_cube(linfo->shadow_cube_target_size, shadow_pool_format, DRW_TEX_FILTER, NULL);
	}

	if (!sldata->shadow_cascade_target) {
		/* CSM */
		sldata->shadow_cascade_target = DRW_texture_create_2D_array(
		        linfo->shadow_size, linfo->shadow_size, MAX_CASCADE_NUM, DRW_TEX_DEPTH_24, 0, NULL);
		sldata->shadow_cascade_blur = DRW_texture_create_2D_array(
		        linfo->shadow_size, linfo->shadow_size, MAX_CASCADE_NUM, shadow_pool_format, DRW_TEX_FILTER, NULL);
	}
	...
}
```

```
/* this refresh lamps shadow buffers */
void EEVEE_draw_shadows(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	EEVEE_LampsInfo *linfo = sldata->lamps;
	Object *ob;
	int i;
	float clear_col[4] = {FLT_MAX};

	/* Cube Shadow Maps */
	DRW_framebuffer_texture_attach(sldata->shadow_target_fb, sldata->shadow_cube_target, 0, 0);
	/* Render each shadow to one layer of the array */
	for (i = 0; (ob = linfo->shadow_cube_ref[i]) && (i < MAX_SHADOW_CUBE); i++) {
		EEVEE_LampEngineData *led = EEVEE_lamp_data_get(ob);
		Lamp *la = (Lamp *)ob->data;

		float cube_projmat[4][4];
		perspective_m4(cube_projmat, -la->clipsta, la->clipsta, -la->clipsta, la->clipsta, la->clipsta, la->clipend);

		if (led->need_update) {
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

				mul_m4_m4m4(srd->shadowmat[j], cube_projmat, srd->viewmat[j]);
			}
			DRW_uniformbuffer_update(sldata->shadow_render_ubo, srd);

			DRW_framebuffer_bind(sldata->shadow_target_fb);
			DRW_framebuffer_clear(true, true, false, clear_col, 1.0f);

			/* Render shadow cube */
			DRW_draw_pass(psl->shadow_cube_pass);

			for (linfo->current_shadow_face = 0;
			     linfo->current_shadow_face < 6;
			     linfo->current_shadow_face++)
			{
				/* Copy using a small 3x3 box filter */
				linfo->filter_size = (la->soft > 0.00001f) ? 1.0f : 0.0f;
				DRW_framebuffer_cubeface_attach(sldata->shadow_store_fb, sldata->shadow_cube_blur, 0, linfo->current_shadow_face, 0);
				DRW_framebuffer_bind(sldata->shadow_store_fb);
				DRW_draw_pass(psl->shadow_cube_copy_pass);
				DRW_framebuffer_texture_detach(sldata->shadow_cube_blur);
			}

			/* Push it to shadowmap array */
			linfo->filter_size = la->soft * 0.0005f;
			DRW_framebuffer_texture_layer_attach(sldata->shadow_store_fb, sldata->shadow_pool, 0, i, 0);
			DRW_framebuffer_bind(sldata->shadow_store_fb);
			DRW_draw_pass(psl->shadow_cube_store_pass);

			led->need_update = false;
		}
	}
	linfo->update_flag &= ~LIGHT_UPDATE_SHADOW_CUBE;

	DRW_framebuffer_texture_detach(sldata->shadow_cube_target);

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
		for (int j = 0; j < la->cascade_count; ++j) {
			copy_m4_m4(srd->shadowmat[j], evscd->viewprojmat[j]);
		}
		DRW_uniformbuffer_update(sldata->shadow_render_ubo, &linfo->shadow_render_data);

		DRW_framebuffer_bind(sldata->shadow_target_fb);
		DRW_framebuffer_clear(false, true, false, NULL, 1.0);

		/* Render shadow cascades */
		DRW_draw_pass(psl->shadow_cascade_pass);

		for (linfo->current_shadow_cascade = 0;
		     linfo->current_shadow_cascade < la->cascade_count;
		     ++linfo->current_shadow_cascade)
		{
			/* Copy using a small 3x3 box filter */
			linfo->filter_size = (la->soft > 0.00001f) ? 1.0f : 0.0f;
			DRW_framebuffer_texture_layer_attach(sldata->shadow_store_fb, sldata->shadow_cascade_blur, 0, linfo->current_shadow_cascade, 0);
			DRW_framebuffer_bind(sldata->shadow_store_fb);
			DRW_draw_pass(psl->shadow_cascade_copy_pass);
			DRW_framebuffer_texture_detach(sldata->shadow_cascade_blur);

			/* Push it to shadowmap array and blur more */
			linfo->filter_size = la->soft * 0.0005f / (evscd->radius[linfo->current_shadow_cascade] * 0.05f);
			int layer = evscd->layer_id + linfo->current_shadow_cascade;
			DRW_framebuffer_texture_layer_attach(sldata->shadow_store_fb, sldata->shadow_pool, 0, layer, 0);
			DRW_framebuffer_bind(sldata->shadow_store_fb);
			DRW_draw_pass(psl->shadow_cascade_store_pass);
		}
	}

	DRW_framebuffer_texture_detach(sldata->shadow_cascade_target);
}
```
>
- 在  Cube Shadow Maps 过程中，DRW_draw_pass(psl->shadow_cube_pass); 之后 把深度信息渲染到 shadow_cube_target Cube RT 上，接下来 shadow_cube_target Cube RT 会传入到 DRW_draw_pass(psl->shadow_cube_copy_pass); 输出 shadow_cube_blur RT, 再进行 DRW_draw_pass(psl->shadow_cube_store_pass);
<br><br>
- 在 Cascaded Shadow Maps 过程中，DRW_draw_pass(psl->shadow_cascade_pass); 之后把深度信息渲染到 shadow_cascade_target RT 上，接下来 shadow_cascade_target RT 会传入到 DRW_draw_pass(psl->shadow_cascade_copy_pass);，再进行 DRW_draw_pass(psl->shadow_cascade_store_pass);

<br><br>

### shadow_cube_copy_pass 和 shadow_cascade_copy_pass

#### 1.初始化

```
void EEVEE_lights_init(EEVEE_SceneLayerData *sldata)
{
	...
	if (!e_data.shadow_sh) {
		e_data.shadow_sh = DRW_shader_create(
		        datatoc_shadow_vert_glsl, da		        datatoc_shadow_vert_glsl, datatoc_shadow_geom_glsl, datatoc_shadow_frag_glsl, NULL);tatoc_shadow_geom_glsl, datatoc_shadow_frag_glsl, NULL);

		DynStr *ds_frag = BLI_dynstr_new();
		BLI_dynstr_append(ds_frag, datatoc_concentric_samples_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_shadow_store_frag_glsl);
		char *store_shadow_shader_str = BLI_dynstr_get_cstring(ds_frag);
		BLI_dynstr_free(ds_frag);

		e_data.shadow_store_cube_sh[SHADOW_ESM] = DRW_shader_create_fullscreen(store_shadow_shader_str, "#define ESM\n");
		e_data.shadow_store_cascade_sh[SHADOW_ESM] = DRW_shader_create_fullscreen(store_shadow_shader_str, "#define ESM\n"
		                                                                                                   "#define CSM\n");

		e_data.shadow_store_cube_sh[SHADOW_VSM] = DRW_shader_create_fullscreen(store_shadow_shader_str, "#define VSM\n");
		e_data.shadow_store_cascade_sh[SHADOW_VSM] = DRW_shader_create_fullscreen(store_shadow_shader_str, "#define VSM\n"
		                                                                                                   "#define CSM\n");

		MEM_freeN(store_shadow_shader_str);

		e_data.shadow_copy_cube_sh[SHADOW_ESM] = DRW_shader_create_fullscreen(datatoc_shadow_copy_frag_glsl, "#define ESM\n"
		                                                                                                     "#define COPY\n");
		e_data.shadow_copy_cascade_sh[SHADOW_ESM] = DRW_shader_create_fullscreen(datatoc_shadow_copy_frag_glsl, "#define ESM\n"
		                                                                                                        "#define COPY\n"
		                                                                                                        "#define CSM\n");

		e_data.shadow_copy_cube_sh[SHADOW_VSM] = DRW_shader_create_fullscreen(datatoc_shadow_copy_frag_glsl, "#define VSM\n"
		                                                                                                     "#define COPY\n");
		e_data.shadow_copy_cascade_sh[SHADOW_VSM] = DRW_shader_create_fullscreen(datatoc_shadow_copy_frag_glsl, "#define VSM\n"
		                                                                                                        "#define COPY\n"
		                                                                                                        "#define CSM\n");
	}
	...
}
```
<br>

```
void EEVEE_lights_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
		...
		{
		psl->shadow_cube_copy_pass = DRW_pass_create("Shadow Copy Pass", DRW_STATE_WRITE_COLOR);

		DRWShadingGroup *grp = DRW_shgroup_create(e_data.shadow_copy_cube_sh[linfo->shadow_method], psl->shadow_cube_copy_pass);
		DRW_shgroup_uniform_buffer(grp, "shadowTexture", &sldata->shadow_cube_target);
		DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
		DRW_shgroup_uniform_float(grp, "shadowFilterSize", &linfo->filter_size, 1);
		DRW_shgroup_uniform_int(grp, "faceId", &linfo->current_shadow_face, 1);
		DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL);
		}
		...

		{
			psl->shadow_cascade_copy_pass = DRW_pass_create("Shadow Cascade Copy Pass", DRW_STATE_WRITE_COLOR);

			DRWShadingGroup *grp = DRW_shgroup_create(e_data.shadow_copy_cascade_sh[linfo->shadow_method], psl->shadow_cascade_copy_pass);
			DRW_shgroup_uniform_buffer(grp, "shadowTexture", &sldata->shadow_cascade_target);
			DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
			DRW_shgroup_uniform_float(grp, "shadowFilterSize", &linfo->filter_size, 1);
			DRW_shgroup_uniform_int(grp, "cascadeId", &linfo->current_shadow_cascade, 1);
			DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL);
		}
		
		
}
```
>
- shadow_cube_copy_pass 使用 shadow_copy_frag.glsl, 但是跟根据 shadow_method 来定义 ESM 和 VSM
<br><br>
- shadow_cube_copy_pass Shader 中的 传入第一步 shadow_cube_pass 渲染完的结果 shadow_cube_target, 输出的 shadow_cube_blur RT
<br><br>
- shadow_cube_copy_pass 输出  shadow_cube_blur RT，正常来说是传入到 shadow_cube_store_pass 进行 Push it to shadowmap array

<br>

>
- shadow_cascade_copy_pass 使用 shadow_copy_frag.glsl, 定义了CSM, 但是跟根据 shadow_method 来定义 ESM 和 VSM
<br><br>
- shadow_cascade_copy_pass Shader 中的 传入第一步 shadow_cascade_pass 渲染完的结果 shadow_cascade_target, 输出的 shadow_cascade_blur RT
<br><br>
- shadow_cascade_copy_pass 输出shadow_cascade_blur RT 会传入到 shadow_cascade_store_pass 进行 Push it to shadowmap array


<br><br>

#### 2. Shader
*shadow_copy_frag*
```
/* Copy the depth only shadowmap into another texture while converting
 * to linear depth (or other storage method) and doing a 3x3 box filter. */

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
uniform vec3 cubeFaceVec[3];
#endif
uniform float shadowFilterSize;

out vec4 FragColor;

float linear_depth(float z)
{
	return (nearClip  * farClip) / (z * (nearClip - farClip) + farClip);
}

vec4 linear_depth(vec4 z)
{
	return (nearClip  * farClip) / (z * (nearClip - farClip) + farClip);
}

#ifdef CSM
vec4 get_world_distance(vec4 depths, vec3 cos[4])
{
	/* Background case */
	vec4 is_background = step(vec4(0.99999), depths);
	depths *= abs(farClip - nearClip); /* Same factor as in shadow_cascade(). */
	depths += 1e1 * is_background;
	return depths;
}

float get_world_distance(float depth, vec3 cos)
{
	/* Background case */
	float is_background = step(0.9999, depth);
	depth *= abs(farClip - nearClip); /* Same factor as in shadow_cascade(). */
	depth += 1e1 * is_background;
	return depth;
}
#else /* CUBEMAP */
vec4 get_world_distance(vec4 depths, vec3 cos[4])
{
	vec4 is_background = step(vec4(1.0), depths);
	depths = linear_depth(depths);
	depths += vec4(1e16) * is_background;
	cos[0] = normalize(abs(cos[0]));
	cos[1] = normalize(abs(cos[1]));
	cos[2] = normalize(abs(cos[2]));
	cos[3] = normalize(abs(cos[3]));
	vec4 cos_vec;
	cos_vec.x = max(cos[0].x, max(cos[0].y, cos[0].z));
	cos_vec.y = max(cos[1].x, max(cos[1].y, cos[1].z));
	cos_vec.z = max(cos[2].x, max(cos[2].y, cos[2].z));
	cos_vec.w = max(cos[3].x, max(cos[3].y, cos[3].z));
	return depths / cos_vec;
}

float get_world_distance(float depth, vec3 cos)
{
	float is_background = step(1.0, depth);
	depth = linear_depth(depth);
	depth += 1e16 * is_background;
	cos = normalize(abs(cos));
	float cos_vec = max(cos.x, max(cos.y, cos.z));
	return depth / cos_vec;
}
#endif

/* Marco Salvi's GDC 2008 presentation about shadow maps pre-filtering techniques slide 24 */
float ln_space_prefilter(float w0, float x, float w1, float y)
{
    return x + log(w0 + w1 * exp(y - x));
}

#define SAMPLE_WEIGHT 0.11111

#ifdef ESM
void filter(vec4 depths, inout float accum)
{
	accum = ln_space_prefilter(1.0, accum, SAMPLE_WEIGHT, depths.x);
	accum = ln_space_prefilter(1.0, accum, SAMPLE_WEIGHT, depths.y);
	accum = ln_space_prefilter(1.0, accum, SAMPLE_WEIGHT, depths.z);
	accum = ln_space_prefilter(1.0, accum, SAMPLE_WEIGHT, depths.w);
}
#else /* VSM */
void filter(vec4 depths, inout vec2 accum)
{
	vec4 depths_sqr = depths * depths;
	accum += vec2(dot(vec4(1.0), depths), dot(vec4(1.0), depths_sqr)) * SAMPLE_WEIGHT;
}
#endif

#ifdef CSM
vec3 get_texco(vec2 uvs, vec2 ofs)
{
	return vec3(uvs + ofs, float(cascadeId));
}
#else /* CUBEMAP */
const vec3 minorAxisX[6] = vec3[6](
	vec3(0.0f, 0.0f, -1.0f),
	vec3(0.0f, 0.0f, 1.0f),
	vec3(1.0f, 0.0f, 0.0f),
	vec3(1.0f, 0.0f, 0.0f),
	vec3(1.0f, 0.0f, 0.0f),
	vec3(-1.0f, 0.0f, 0.0f)
);

const vec3 minorAxisY[6] = vec3[6](
	vec3(0.0f, -1.0f, 0.0f),
	vec3(0.0f, -1.0f, 0.0f),
	vec3(0.0f, 0.0f, -1.0f),
	vec3(0.0f, 0.0f, 1.0f),
	vec3(0.0f, -1.0f, 0.0f),
	vec3(0.0f, -1.0f, 0.0f)
);

const vec3 majorAxis[6] = vec3[6](
	vec3(-1.0f, 0.0f, 0.0f),
	vec3(1.0f, 0.0f, 0.0f),
	vec3(0.0f, 1.0f, 0.0f),
	vec3(0.0f, -1.0f, 0.0f),
	vec3(0.0f, 0.0f, -1.0f),
	vec3(0.0f, 0.0f, 1.0f)
);

vec3 get_texco(vec2 uvs, vec2 ofs)
{
	uvs += ofs;
	return majorAxis[0] + uvs.x * minorAxisX[1] + uvs.y * minorAxisY[2];
}
#endif

void main() {
	/* Copy the depth only shadowmap into another texture while converting
	 * to linear depth and do a 3x3 box blur. */

#ifdef CSM
	vec2 uvs = gl_FragCoord.xy * storedTexelSize;
#else /* CUBEMAP */
	vec2 uvs = gl_FragCoord.xy * cubeTexelSize;
#endif

	/* Center texel */
	vec3 co = get_texco(uvs, vec2(0.0));
	float depth = texture(shadowTexture, co).r;
	depth = get_world_distance(depth, co);
#ifdef ESM
	float accum = ln_space_prefilter(0.0, 0.0, SAMPLE_WEIGHT, depth);
#else /* VSM */
	vec2 accum = vec2(depth, depth * depth) * SAMPLE_WEIGHT;
#endif

#ifdef CSM
	vec3 ofs = storedTexelSize * vec3(1.0, 0.0, -1.0) * shadowFilterSize;
#else /* CUBEMAP */
	vec3 ofs = cubeTexelSize * vec3(1.0, 0.0, -1.0) * shadowFilterSize;
#endif

	vec3 cos[4];
	cos[0] = get_texco(uvs, ofs.zz);
	cos[1] = get_texco(uvs, ofs.yz);
	cos[2] = get_texco(uvs, ofs.xz);
	cos[3] = get_texco(uvs, ofs.zy);

	vec4 depths;
	depths.x = texture(shadowTexture, cos[0]).r;
	depths.y = texture(shadowTexture, cos[1]).r;
	depths.z = texture(shadowTexture, cos[2]).r;
	depths.w = texture(shadowTexture, cos[3]).r;
	depths = get_world_distance(depths, cos);
	filter(depths, accum);

	cos[0] = get_texco(uvs, ofs.xy);
	cos[1] = get_texco(uvs, ofs.zx);
	cos[2] = get_texco(uvs, ofs.yx);
	cos[3] = get_texco(uvs, ofs.xx);
	depths.x = texture(shadowTexture, cos[0]).r;
	depths.y = texture(shadowTexture, cos[1]).r;
	depths.z = texture(shadowTexture, cos[2]).r;
	depths.w = texture(shadowTexture, cos[3]).r;
	depths = get_world_distance(depths, cos);
	filter(depths, accum);

	FragColor = vec2(accum).xyxy;
}
```

<br><br>

### shadow_cube_store_pass 和 shadow_cascade_store_pass

#### 1. 初始化
```
void EEVEE_lights_init(EEVEE_SceneLayerData *sldata)
{
	...
	e_data.shadow_sh = DRW_shader_create(
		        datatoc_shadow_vert_glsl, datatoc_shadow_geom_glsl, datatoc_shadow_frag_glsl, NULL);

	DynStr *ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, datatoc_concentric_samples_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_shadow_store_frag_glsl);
	char *store_shadow_shader_str = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	e_data.shadow_store_cube_sh[SHADOW_ESM] = DRW_shader_create_fullscreen(store_shadow_shader_str, "#define ESM\n");
	e_data.shadow_store_cascade_sh[SHADOW_ESM] = DRW_shader_create_fullscreen(store_shadow_shader_str, "#define ESM\n"
																										"#define CSM\n");

	e_data.shadow_store_cube_sh[SHADOW_VSM] = DRW_shader_create_fullscreen(store_shadow_shader_str, "#define VSM\n");
	e_data.shadow_store_cascade_sh[SHADOW_VSM] = DRW_shader_create_fullscreen(store_shadow_shader_str, "#define VSM\n"
																										"#define CSM\n");
	...
}

void EEVEE_lights_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	...
	{
		psl->shadow_cascade_store_pass = DRW_pass_create("Shadow Cascade Storage Pass", DRW_STATE_WRITE_COLOR);

		DRWShadingGroup *grp = DRW_shgroup_create(e_data.shadow_store_cascade_sh[linfo->shadow_method], psl->shadow_cascade_store_pass);
		DRW_shgroup_uniform_buffer(grp, "shadowTexture", &sldata->shadow_cascade_blur);
		DRW_shgroup_uniform_block(grp, "shadow_render_block", sldata->shadow_render_ubo);
		DRW_shgroup_uniform_int(grp, "cascadeId", &linfo->current_shadow_cascade, 1);
		DRW_shgroup_uniform_float(grp, "shadowFilterSize", &linfo->filter_size, 1);
		DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL);
	}
	...
}
```
>
- shadow_cube_store_pass 和 shadow_cascade_store_pass 现在变了 shadow_store_frag.glsl + concentric_samples_lib.glsl 组合
<br><br>
- shadow_cube_store_pass 和 shadow_cascade_store_pass 传入的Shader 都是第二步的渲染结果，例如 shadow_cube_blur RT 和 shadow_cascade_blur RT

<br><br>

#### 2. Shader
*concentric_samples_lib.glsl*
```
/* Precomputed table of concentric samples.
 * Generated using this algorithm http://l2program.co.uk/900/concentric-disk-sampling
 * Sorted by radius then by rotation angle.
 * This way it's better for cache usage and for
 * easily restricting to a certain number of
 * sample while still having a circular kernel. */

#define CONCENTRIC_SAMPLE_NUM 256
const vec2 concentric[CONCENTRIC_SAMPLE_NUM] =
vec2[CONCENTRIC_SAMPLE_NUM](
	vec2(0.0441941738242, 0.0441941738242),
	vec2(-0.0441941738242, -0.0441941738242),
	vec2(-0.0441941738242, 0.0441941738242),
	vec2(0.0441941738242, -0.0441941738242),
	vec2(0.181111092429, 0.0485285709567),
	vec2(0.132582521472, 0.132582521472),
	vec2(-0.181111092429, 0.0485285709567),
	vec2(0.0485285709567, 0.181111092429),
	vec2(-0.181111092429, -0.0485285709567),
	vec2(-0.0485285709567, 0.181111092429),
	vec2(-0.132582521472, -0.132582521472),
	vec2(-0.132582521472, 0.132582521472),
	vec2(-0.0485285709567, -0.181111092429),
	vec2(0.0485285709567, -0.181111092429),
	vec2(0.132582521472, -0.132582521472),
	vec2(0.181111092429, -0.0485285709567),
	vec2(0.308652606436, 0.0488857703251),
	vec2(0.278439538809, 0.141872031169),
	vec2(0.220970869121, 0.220970869121),
	vec2(-0.278439538809, 0.141872031169),
	vec2(0.141872031169, 0.278439538809),
	vec2(-0.308652606436, 0.0488857703251),
	vec2(0.0488857703251, 0.308652606436),
	vec2(-0.308652606436, -0.0488857703251),
	vec2(-0.0488857703251, 0.308652606436),
	vec2(-0.278439538809, -0.141872031169),
	vec2(-0.141872031169, 0.278439538809),
	vec2(-0.220970869121, -0.220970869121),
	vec2(-0.220970869121, 0.220970869121),
	vec2(-0.141872031169, -0.278439538809),
	vec2(-0.0488857703251, -0.308652606436),
	vec2(0.0488857703251, -0.308652606436),
	vec2(0.141872031169, -0.278439538809),
	vec2(0.220970869121, -0.220970869121),
	vec2(0.278439538809, -0.141872031169),
	vec2(0.308652606436, -0.0488857703251),
	vec2(0.434749091828, 0.0489844582952),
	vec2(0.41294895701, 0.144497089605),
	vec2(0.370441837162, 0.232764033475),
	vec2(0.309359216769, 0.309359216769),
	vec2(-0.370441837162, 0.232764033475),
	vec2(0.232764033475, 0.370441837162),
	vec2(-0.41294895701, 0.144497089605),
	vec2(0.144497089605, 0.41294895701),
	vec2(-0.434749091828, 0.0489844582952),
	vec2(0.0489844582952, 0.434749091828),
	vec2(-0.434749091828, -0.0489844582952),
	vec2(-0.0489844582952, 0.434749091828),
	vec2(-0.41294895701, -0.144497089605),
	vec2(-0.144497089605, 0.41294895701),
	vec2(-0.370441837162, -0.232764033475),
	vec2(-0.232764033475, 0.370441837162),
	vec2(-0.309359216769, -0.309359216769),
	vec2(-0.309359216769, 0.309359216769),
	vec2(-0.232764033475, -0.370441837162),
	vec2(-0.144497089605, -0.41294895701),
	vec2(-0.0489844582952, -0.434749091828),
	vec2(0.0489844582952, -0.434749091828),
	vec2(0.144497089605, -0.41294895701),
	vec2(0.232764033475, -0.370441837162),
	vec2(0.309359216769, -0.309359216769),
	vec2(0.370441837162, -0.232764033475),
	vec2(0.41294895701, -0.144497089605),
	vec2(0.434749091828, -0.0489844582952),
	vec2(0.560359517677, 0.0490251052956),
	vec2(0.543333277288, 0.14558571287),
	vec2(0.509798130208, 0.237722772229),
	vec2(0.460773024913, 0.322636745447),
	vec2(0.397747564417, 0.397747564417),
	vec2(-0.460773024913, 0.322636745447),
	vec2(0.322636745447, 0.460773024913),
	vec2(-0.509798130208, 0.237722772229),
	vec2(0.237722772229, 0.509798130208),
	vec2(-0.543333277288, 0.14558571287),
	vec2(0.14558571287, 0.543333277288),
	vec2(-0.560359517677, 0.0490251052956),
	vec2(0.0490251052956, 0.560359517677),
	vec2(-0.560359517677, -0.0490251052956),
	vec2(-0.0490251052956, 0.560359517677),
	vec2(-0.543333277288, -0.14558571287),
	vec2(-0.14558571287, 0.543333277288),
	vec2(-0.509798130208, -0.237722772229),
	vec2(-0.237722772229, 0.509798130208),
	vec2(-0.460773024913, -0.322636745447),
	vec2(-0.322636745447, 0.460773024913),
	vec2(-0.397747564417, -0.397747564417),
	vec2(-0.397747564417, 0.397747564417),
	vec2(-0.322636745447, -0.460773024913),
	vec2(-0.237722772229, -0.509798130208),
	vec2(-0.14558571287, -0.543333277288),
	vec2(-0.0490251052956, -0.560359517677),
	vec2(0.0490251052956, -0.560359517677),
	vec2(0.14558571287, -0.543333277288),
	vec2(0.237722772229, -0.509798130208),
	vec2(0.322636745447, -0.460773024913),
	vec2(0.397747564417, -0.397747564417),
	vec2(0.460773024913, -0.322636745447),
	vec2(0.509798130208, -0.237722772229),
	vec2(0.543333277288, -0.14558571287),
	vec2(0.560359517677, -0.0490251052956),
	vec2(0.685748328795, 0.0490456884495),
	vec2(0.671788470355, 0.146138636568),
	vec2(0.644152935937, 0.240256623474),
	vec2(0.603404305327, 0.32948367837),
	vec2(0.550372103135, 0.412003395727),
	vec2(0.486135912066, 0.486135912066),
	vec2(-0.550372103135, 0.412003395727),
	vec2(0.412003395727, 0.550372103135),
	vec2(-0.603404305327, 0.32948367837),
	vec2(0.32948367837, 0.603404305327),
	vec2(-0.644152935937, 0.240256623474),
	vec2(0.240256623474, 0.644152935937),
	vec2(-0.671788470355, 0.146138636568),
	vec2(0.146138636568, 0.671788470355),
	vec2(-0.685748328795, 0.0490456884495),
	vec2(0.0490456884495, 0.685748328795),
	vec2(-0.685748328795, -0.0490456884495),
	vec2(-0.0490456884495, 0.685748328795),
	vec2(-0.671788470355, -0.146138636568),
	vec2(-0.146138636568, 0.671788470355),
	vec2(-0.644152935937, -0.240256623474),
	vec2(-0.240256623474, 0.644152935937),
	vec2(-0.603404305327, -0.32948367837),
	vec2(-0.32948367837, 0.603404305327),
	vec2(-0.550372103135, -0.412003395727),
	vec2(-0.412003395727, 0.550372103135),
	vec2(-0.486135912066, -0.486135912066),
	vec2(-0.486135912066, 0.486135912066),
	vec2(-0.412003395727, -0.550372103135),
	vec2(-0.32948367837, -0.603404305327),
	vec2(-0.240256623474, -0.644152935937),
	vec2(-0.146138636568, -0.671788470355),
	vec2(-0.0490456884495, -0.685748328795),
	vec2(0.0490456884495, -0.685748328795),
	vec2(0.146138636568, -0.671788470355),
	vec2(0.240256623474, -0.644152935937),
	vec2(0.32948367837, -0.603404305327),
	vec2(0.412003395727, -0.550372103135),
	vec2(0.486135912066, -0.486135912066),
	vec2(0.550372103135, -0.412003395727),
	vec2(0.603404305327, -0.32948367837),
	vec2(0.644152935937, -0.240256623474),
	vec2(0.671788470355, -0.146138636568),
	vec2(0.685748328795, -0.0490456884495),
	vec2(0.811017637806, 0.0490575291556),
	vec2(0.799191174395, 0.146457218224),
	vec2(0.775710704038, 0.241721231257),
	vec2(0.740918624869, 0.33346040443),
	vec2(0.695322283745, 0.420336974019),
	vec2(0.639586577995, 0.501084084011),
	vec2(0.574524259714, 0.574524259714),
	vec2(-0.639586577995, 0.501084084011),
	vec2(0.501084084011, 0.639586577995),
	vec2(-0.695322283745, 0.420336974019),
	vec2(0.420336974019, 0.695322283745),
	vec2(-0.740918624869, 0.33346040443),
	vec2(0.33346040443, 0.740918624869),
	vec2(-0.775710704038, 0.241721231257),
	vec2(0.241721231257, 0.775710704038),
	vec2(-0.799191174395, 0.146457218224),
	vec2(0.146457218224, 0.799191174395),
	vec2(-0.811017637806, 0.0490575291556),
	vec2(0.0490575291556, 0.811017637806),
	vec2(-0.811017637806, -0.0490575291556),
	vec2(-0.0490575291556, 0.811017637806),
	vec2(-0.799191174395, -0.146457218224),
	vec2(-0.146457218224, 0.799191174395),
	vec2(-0.775710704038, -0.241721231257),
	vec2(-0.241721231257, 0.775710704038),
	vec2(-0.740918624869, -0.33346040443),
	vec2(-0.33346040443, 0.740918624869),
	vec2(-0.695322283745, -0.420336974019),
	vec2(-0.420336974019, 0.695322283745),
	vec2(-0.639586577995, -0.501084084011),
	vec2(-0.501084084011, 0.639586577995),
	vec2(-0.574524259714, -0.574524259714),
	vec2(-0.574524259714, 0.574524259714),
	vec2(-0.501084084011, -0.639586577995),
	vec2(-0.420336974019, -0.695322283745),
	vec2(-0.33346040443, -0.740918624869),
	vec2(-0.241721231257, -0.775710704038),
	vec2(-0.146457218224, -0.799191174395),
	vec2(-0.0490575291556, -0.811017637806),
	vec2(0.0490575291556, -0.811017637806),
	vec2(0.146457218224, -0.799191174395),
	vec2(0.241721231257, -0.775710704038),
	vec2(0.33346040443, -0.740918624869),
	vec2(0.420336974019, -0.695322283745),
	vec2(0.501084084011, -0.639586577995),
	vec2(0.574524259714, -0.574524259714),
	vec2(0.639586577995, -0.501084084011),
	vec2(0.695322283745, -0.420336974019),
	vec2(0.740918624869, -0.33346040443),
	vec2(0.775710704038, -0.241721231257),
	vec2(0.799191174395, -0.146457218224),
	vec2(0.811017637806, -0.0490575291556),
	vec2(0.936215188832, 0.0490649589778),
	vec2(0.925957819308, 0.146657310975),
	vec2(0.905555462146, 0.242642854784),
	vec2(0.875231649841, 0.335969952699),
	vec2(0.835318616427, 0.425616093506),
	vec2(0.786253657449, 0.510599095327),
	vec2(0.728574338866, 0.589987866609),
	vec2(0.662912607362, 0.662912607362),
	vec2(-0.728574338866, 0.589987866609),
	vec2(0.589987866609, 0.728574338866),
	vec2(-0.786253657449, 0.510599095327),
	vec2(0.510599095327, 0.786253657449),
	vec2(-0.835318616427, 0.425616093506),
	vec2(0.425616093506, 0.835318616427),
	vec2(-0.875231649841, 0.335969952699),
	vec2(0.335969952699, 0.875231649841),
	vec2(-0.905555462146, 0.242642854784),
	vec2(0.242642854784, 0.905555462146),
	vec2(-0.925957819308, 0.146657310975),
	vec2(0.146657310975, 0.925957819308),
	vec2(-0.936215188832, 0.0490649589778),
	vec2(0.0490649589778, 0.936215188832),
	vec2(-0.936215188832, -0.0490649589778),
	vec2(-0.0490649589778, 0.936215188832),
	vec2(-0.925957819308, -0.146657310975),
	vec2(-0.146657310975, 0.925957819308),
	vec2(-0.905555462146, -0.242642854784),
	vec2(-0.242642854784, 0.905555462146),
	vec2(-0.875231649841, -0.335969952699),
	vec2(-0.335969952699, 0.875231649841),
	vec2(-0.835318616427, -0.425616093506),
	vec2(-0.425616093506, 0.835318616427),
	vec2(-0.786253657449, -0.510599095327),
	vec2(-0.510599095327, 0.786253657449),
	vec2(-0.728574338866, -0.589987866609),
	vec2(-0.589987866609, 0.728574338866),
	vec2(-0.662912607362, -0.662912607362),
	vec2(-0.662912607362, 0.662912607362),
	vec2(-0.589987866609, -0.728574338866),
	vec2(-0.510599095327, -0.786253657449),
	vec2(-0.425616093506, -0.835318616427),
	vec2(-0.335969952699, -0.875231649841),
	vec2(-0.242642854784, -0.905555462146),
	vec2(-0.146657310975, -0.925957819308),
	vec2(-0.0490649589778, -0.936215188832),
	vec2(0.0490649589778, -0.936215188832),
	vec2(0.146657310975, -0.925957819308),
	vec2(0.242642854784, -0.905555462146),
	vec2(0.335969952699, -0.875231649841),
	vec2(0.425616093506, -0.835318616427),
	vec2(0.510599095327, -0.786253657449),
	vec2(0.589987866609, -0.728574338866),
	vec2(0.662912607362, -0.662912607362),
	vec2(0.728574338866, -0.589987866609),
	vec2(0.786253657449, -0.510599095327),
	vec2(0.835318616427, -0.425616093506),
	vec2(0.875231649841, -0.335969952699),
	vec2(0.905555462146, -0.242642854784),
	vec2(0.925957819308, -0.146657310975),
	vec2(0.936215188832, -0.0490649589778)
);
```


<br>

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

void make_orthonormal_basis(vec3 N, out vec3 T, out vec3 B)
{
	vec3 UpVector = (abs(N.z) < 0.999) ? vec3(0.0, 0.0, 1.0) : vec3(1.0, 0.0, 0.0);
	vec3 nT = normalize(cross(UpVector, N));
	vec3 nB = cross(N, nT);
}

/* Marco Salvi's GDC 2008 presentation about shadow maps pre-filtering techniques slide 24 */
float ln_space_prefilter(float w0, float x, float w1, float y)
{
    return x + log(w0 + w1 * exp(y - x));
}

vec4 ln_space_prefilter(float w0, vec4 x, float w1, vec4 y)
{
    return x + log(w0 + w1 * exp(y - x));
}

/* globals */
vec3 T, B;

#ifdef CSM
vec3 get_texco(vec3 cos, vec2 ofs)
{
	cos.xy += ofs * shadowFilterSize;
	return cos;
}
#else /* CUBEMAP */
vec3 get_texco(vec3 cos, vec2 ofs)
{
	return cos + ofs.x * T + ofs.y * B;
}
#endif

const float INV_SAMPLE_NUM = 1.0 / float(CONCENTRIC_SAMPLE_NUM);

void main() {
	vec3 cos;

	cos.xy = gl_FragCoord.xy * storedTexelSize;

#ifdef CSM
	cos.z = float(cascadeId);
#else /* CUBEMAP */
	/* add a 2 pixel border to ensure filtering is correct */
	cos.xy *= 1.0 + storedTexelSize * 2.0;
	cos.xy -= storedTexelSize;

	float pattern = 1.0;

	/* edge mirroring : only mirror if directly adjacent
	 * (not diagonally adjacent) */
	vec2 m = abs(cos.xy - 0.5) + 0.5;
	vec2 f = floor(m);
	if (f.x - f.y != 0.0) {
		cos.xy = 1.0 - cos.xy;
	}

	/* clamp to [0-1] */
	cos.xy = fract(cos.xy);

	/* get cubemap vector */
	cos = normalize(octahedral_to_cubemap_proj(cos.xy));
	make_orthonormal_basis(cos, T, B);

	T *= shadowFilterSize;
	B *= shadowFilterSize;
#endif

#ifdef ESM
	vec4 accum = vec4(0.0);

	/* disc blur in log space. */
	vec4 depths;
	depths.x = texture(shadowTexture, get_texco(cos, concentric[0])).r;
	depths.y = texture(shadowTexture, get_texco(cos, concentric[1])).r;
	depths.z = texture(shadowTexture, get_texco(cos, concentric[2])).r;
	depths.w = texture(shadowTexture, get_texco(cos, concentric[3])).r;
	accum = ln_space_prefilter(0.0, accum, INV_SAMPLE_NUM, depths);

	for (int i = 4; i < CONCENTRIC_SAMPLE_NUM; i += 4) {
		depths.x = texture(shadowTexture, get_texco(cos, concentric[i+0])).r;
		depths.y = texture(shadowTexture, get_texco(cos, concentric[i+1])).r;
		depths.z = texture(shadowTexture, get_texco(cos, concentric[i+2])).r;
		depths.w = texture(shadowTexture, get_texco(cos, concentric[i+3])).r;
		accum = ln_space_prefilter(1.0, accum, INV_SAMPLE_NUM, depths);
	}

	accum.x = ln_space_prefilter(1.0, accum.x, 1.0, accum.y);
	accum.x = ln_space_prefilter(1.0, accum.x, 1.0, accum.z);
	accum.x = ln_space_prefilter(1.0, accum.x, 1.0, accum.w);
	FragColor = accum.xxxx;

#else /* VSM */
	vec2 accum = vec2(0.0);

	/* disc blur. */
	vec4 depths1, depths2;
	for (int i = 0; i < CONCENTRIC_SAMPLE_NUM; i += 4) {
		depths1.xy = texture(shadowTexture, get_texco(cos, concentric[i+0])).rg;
		depths1.zw = texture(shadowTexture, get_texco(cos, concentric[i+1])).rg;
		depths2.xy = texture(shadowTexture, get_texco(cos, concentric[i+2])).rg;
		depths2.zw = texture(shadowTexture, get_texco(cos, concentric[i+3])).rg;
		accum += depths1.xy + depths1.zw + depths2.xy + depths2.zw;
	}

	FragColor = accum.xyxy * INV_SAMPLE_NUM;
#endif
}
```