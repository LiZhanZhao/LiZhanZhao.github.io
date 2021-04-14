---
layout:     post
title:      "blender eevee Irradiance Visibility: Initial Implementation"
subtitle:   ""
date:       2021-4-14 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/12/2  *   Eevee : Irradiance Visibility: Initial Implementation. <br> 

>
This augment the existing irradiance grid with a new visibility precomputation.
We store a small shadowmap for each grid sample so that light does not leak through walls and such.

>
The visibility parameter are similar to the one used by the Variance Shadow Map for point lights.

> Technical details:

> We store the visibility in the same texture (array) as the irradiance itself (in order to reduce the number of sampler).
But the irradiance and the visibility are not the same data so we must encode them in order to use the same texture format.
We use RGBA8 normalized texture and encode irradiance as RGBE (shared exponent).
Using RGBE encoding instead of R11_G11_B10 may lead to some lighting changes, but quality seems to be nearly the same in my test cases.
Using full RGBA16/32F maybe a future option but that will require much more memory and reduce the perf significantly.

> Visibility moments (VSM) are encoded as 16bits fixed point precision using a special range. This seems to retain enough precision for the needs.
Also interpolation does not seems to be big problem (even though it's incorrect).


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
irradiance grid 考虑可见性，例如，光线不能够穿墙

<br><br>

## 效果
*有墙挡住光源的效果*
![](/img/Eevee/IradianceGrid/Visibility/1.png)
<br><br>
*没有墙挡住光源的效果*
![](/img/Eevee/IradianceGrid/Visibility/2.png)

<br><br>

### 渲染
*eevee_lightprobes.c*


```
void EEVEE_lightprobes_cache_finish(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	...
	if (!sldata->irradiance_rt) {
		sldata->irradiance_rt = DRW_texture_create_2D_array(IRRADIANCE_POOL_SIZE, IRRADIANCE_POOL_SIZE, 2,
															irradiance_format, DRW_TEX_FILTER, NULL);
	}
	...
}


/* Diffuse filter probe_rt to irradiance_pool at index probe_idx */
static void diffuse_filter_probe(
	EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata, EEVEE_PassList *psl, int offset,
	float clipsta, float clipend, float vis_range, float vis_blur)
{
	...

	DRW_framebuffer_bind(sldata->probe_filter_fb);

	...
	/* Compute visibility */
	pinfo->samples_ct = 512.0f; /* TODO refine */
	pinfo->invsamples_ct = 1.0f / pinfo->samples_ct;
	pinfo->shres = pinfo->irradiance_vis_size;
	pinfo->visibility_range = vis_range;
	pinfo->visibility_blur = vis_blur;
	pinfo->near_clip = -clipsta;
	pinfo->far_clip = -clipend;
	pinfo->texel_size = 1.0f / (float)pinfo->irradiance_vis_size;

	cell_per_row = IRRADIANCE_POOL_SIZE / pinfo->irradiance_vis_size;
	x = pinfo->irradiance_vis_size * (offset % cell_per_row);
	y = pinfo->irradiance_vis_size * (offset / cell_per_row);

	DRW_framebuffer_texture_detach(sldata->irradiance_rt);
	DRW_framebuffer_texture_layer_attach(sldata->probe_filter_fb, sldata->irradiance_rt, 0, 1, 0);

	DRW_framebuffer_viewport_size(sldata->probe_filter_fb, x, y, pinfo->irradiance_vis_size, sldata->probes->irradiance_vis_size);
	DRW_draw_pass(psl->probe_visibility_compute);
	...
}
```
>
- sldata->irradiance_rt 声明为 一个texture array，数目为 2	.
<br><br>
- 使用 probe_visibility_compute 把cell的可见性信息渲染到  irradiance_rt 的 第二个texture上 .
<br><br>
- 然后把 irradiance 渲染到 irradiance_rt 的 第一个texture上

<br><br>



### probe_visibility_compute

#### 1.初始化
```
void EEVEE_lightprobes_init(EEVEE_ViewLayerData *sldata, EEVEE_Data *UNUSED(vedata))
{
	...
	ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, datatoc_bsdf_common_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_bsdf_sampling_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_lightprobe_filter_visibility_frag_glsl);
	shader_str = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);

	e_data.probe_filter_visibility_sh = DRW_shader_create_fullscreen(
			shader_str,
			"#define HAMMERSLEY_SIZE " STRINGIFY(HAMMERSLEY_SIZE) "\n"
			"#define NOISE_SIZE 64\n");
	...
}

void EEVEE_lightprobes_cache_init(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	...
	{
		psl->probe_visibility_compute = DRW_pass_create("LightProbe Visibility Compute", DRW_STATE_WRITE_COLOR);

		DRWShadingGroup *grp = DRW_shgroup_create(e_data.probe_filter_visibility_sh, psl->probe_visibility_compute);
		DRW_shgroup_uniform_int(grp, "outputSize", &sldata->probes->shres, 1);
		DRW_shgroup_uniform_float(grp, "visibilityRange", &sldata->probes->visibility_range, 1);
		DRW_shgroup_uniform_float(grp, "visibilityBlur", &sldata->probes->visibility_blur, 1);
		DRW_shgroup_uniform_float(grp, "sampleCount", &sldata->probes->samples_ct, 1);
		DRW_shgroup_uniform_float(grp, "invSampleCount", &sldata->probes->invsamples_ct, 1);
		DRW_shgroup_uniform_float(grp, "storedTexelSize", &sldata->probes->texel_size, 1);
		DRW_shgroup_uniform_float(grp, "nearClip", &sldata->probes->near_clip, 1);
		DRW_shgroup_uniform_float(grp, "farClip", &sldata->probes->far_clip, 1);
		DRW_shgroup_uniform_texture(grp, "texHammersley", e_data.hammersley);
		DRW_shgroup_uniform_texture(grp, "probeDepth", sldata->probe_depth_rt);

		struct Gwn_Batch *geom = DRW_cache_fullscreen_quad_get();
		DRW_shgroup_call_add(grp, geom, NULL);
	}
	...
}
```
>
- 这里可以看到 probe_visibility_compute 使用了 lightprobe_filter_visibility_frag.glsl + bsdf_sampling_lib.glsl + bsdf_common_lib.glsl
<br><br>
- probeDepth 是一个 cubamap，保存深度值，这个东西需要传进行Shader中

<br><br>

#### 2.Shader

*bsdf_common_lib.glsl*
```
/* ---- RGBM (shared multiplier) encoding ---- */
/* From http://iwasbeingirony.blogspot.fr/2010/06/difference-between-rgbm-and-rgbd.html */

/* Higher RGBM_MAX_RANGE gives imprecision issues in low intensity. */
#define RGBM_MAX_RANGE 512.0

vec4 rgbm_encode(vec3 rgb)
{
	float maxRGB = max_v3(rgb);
	float M = maxRGB / RGBM_MAX_RANGE;
	M = ceil(M * 255.0) / 255.0;
	return vec4(rgb / (M * RGBM_MAX_RANGE), M);
}

vec3 rgbm_decode(vec4 data)
{
	return data.rgb * (data.a * RGBM_MAX_RANGE);
}

/* ---- RGBE (shared exponent) encoding ---- */
vec4 rgbe_encode(vec3 rgb)
{
	float maxRGB = max_v3(rgb);
	float fexp = ceil(log2(maxRGB));
	return vec4(rgb / exp2(fexp), (fexp + 128.0) / 255.0);
}

vec3 rgbe_decode(vec4 data)
{
	float fexp = data.a * 255.0 - 128.0;
	return data.rgb * exp2(fexp);
}

#if 1
#define irradiance_encode rgbe_encode
#define irradiance_decode rgbe_decode
#else /* No ecoding (when using floating point format) */
#define irradiance_encode(X) (X).rgbb
#define irradiance_decode(X) (X).rgb
#endif

/* Irradiance Visibility Encoding */
#if 1
vec4 visibility_encode(vec2 accum, float range)
{
	accum /= range;

	vec4 data;
	data.x = fract(accum.x);
	data.y = floor(accum.x) / 255.0;
	data.z = fract(accum.y);
	data.w = floor(accum.y) / 255.0;

	return data;
}

vec2 visibility_decode(vec4 data, float range)
{
	return (data.xz + data.yw * 255.0) * range;
}
#else /* No ecoding (when using floating point format) */
vec4 visibility_encode(vec2 accum, float range)
{
	return accum.xyxy;
}

vec2 visibility_decode(vec4 data, float range)
{
	return data.xy;
}
#endif
```
<br>

*bsdf_sampling_lib.glsl*
```
vec3 sample_cone(float nsample, float angle, vec3 N, vec3 T, vec3 B)
{
	vec3 Xi = hammersley_3d(nsample);

	float z = cos(angle * Xi.x); /* cos theta */
	float r = sqrt( 1.0f - z*z ); /* sin theta */
	float x = r * Xi.y;
	float y = r * Xi.z;

	vec3 Ht = vec3(x, y, z);

	return tangent_to_world(Ht, N, T, B);
}
```
<br>

*lightprobe_filter_visibility_frag.glsl*
```

uniform samplerCube probeDepth;
uniform int outputSize;
uniform float lodFactor;
uniform float storedTexelSize;
uniform float lodMax;
uniform float nearClip;
uniform float farClip;
uniform float visibilityRange;
uniform float visibilityBlur;

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

float linear_depth(float z)
{
	return (nearClip  * farClip) / (z * (nearClip - farClip) + farClip);
}

float get_world_distance(float depth, vec3 cos)
{
	float is_background = step(1.0, depth);
	depth = linear_depth(depth);
	depth += 1e1 * is_background;
	cos = normalize(abs(cos));
	float cos_vec = max(cos.x, max(cos.y, cos.z));
	return depth / cos_vec;
}

void main()
{
	ivec2 texel = ivec2(gl_FragCoord.xy) % ivec2(outputSize);

	vec3 cos;

	cos.xy = (vec2(texel) + 0.5) * storedTexelSize;

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

	vec3 T, B;
	make_orthonormal_basis(cos, T, B); /* Generate tangent space */

	vec2 accum = vec2(0.0);

	for (float i = 0; i < sampleCount; i++) {
		vec3 sample = sample_cone(i, M_PI_2 * visibilityBlur, cos, T, B);
		float depth = texture(probeDepth, sample).r;
		depth = get_world_distance(depth, sample);
		accum += vec2(depth, depth * depth);
	}

	accum *= invSampleCount;
	accum = abs(accum);

	/* Encode to normalized RGBA 8 */
	FragColor = visibility_encode(accum, visibilityRange);
}
```
>
- 这里主要是rt 上保存的 world_distance，这个world_distance 是这个cell 大概可见距离
<br><br>
- 也就是之后渲染到RT上面的东西是对应每一个cell的可见性距离

<br><br>


### 应用
*irradiance_lib.glsl*
```
float load_visibility_cell(int cell, vec3 L, float dist, float bias, float bleed_bias, float range)
{
	/* Keep in sync with diffuse_filter_probe() */
	ivec2 cell_co = ivec2(irradianceVisibilitySize);
	int cell_per_row = textureSize(irradianceGrid, 0).x / irradianceVisibilitySize;
	cell_co.x *= (cell) % cell_per_row;
	cell_co.y *= (cell) / cell_per_row;

	vec2 texel_size = 1.0 / vec2(textureSize(irradianceGrid, 0).xy);
	vec2 co = vec2(cell_co) * texel_size;

	vec2 uv = mapping_octahedron(-L, vec2(1.0 / float(irradianceVisibilitySize)));
	uv *= vec2(irradianceVisibilitySize) * texel_size;

	// 这里就是采样iradianceGrid 的数组的第二张图，保存了可见性距离
	vec4 data = texture(irradianceGrid, vec3(co + uv, 1.0));

	/* Decoding compressed data */
	vec2 moments = visibility_decode(data, range);

	/* Doing chebishev test */
	float variance = abs(moments.x * moments.x - moments.y);
	variance = max(variance, bias / 10.0);

	float d = dist - moments.x;
	float p_max = variance / (variance + d * d);

	/* Increase contrast in the weight by squaring it */
	p_max *= p_max;

	/* Now reduce light-bleeding by removing the [0, x] tail and linearly rescaling (x, 1] */
	p_max = clamp((p_max - bleed_bias) / (1.0 - bleed_bias), 0.0, 1.0);

	return (dist <= moments.x) ? 1.0 : p_max;
}
```
>
- 采样iradianceGrid 的数组的第二张图，第二张图保存了可见性距离
<br>
- 这里 d = dist - moments.x; 是计算 物体到 cell 的距离


<br><br>

*lightprobe_lib.glsl*
```
vec3 probe_evaluate_grid(GridData gd, vec3 W, vec3 N, vec3 localpos)
{
	localpos = localpos * 0.5 + 0.5;
	localpos = localpos * vec3(gd.g_resolution) - 0.5;

	vec3 localpos_floored = floor(localpos);
	vec3 trilinear_weight = fract(localpos);

	float weight_accum = 0.0;
	vec3 irradiance_accum = vec3(0.0);

	/* For each neighboor cells */
	for (int i = 0; i < 8; ++i) {
		ivec3 offset = ivec3(i, i >> 1, i >> 2) & ivec3(1);
		vec3 cell_cos = clamp(localpos_floored + vec3(offset), vec3(0.0), vec3(gd.g_resolution) - 1.0);

		/* Keep in sync with update_irradiance_probe */
		ivec3 icell_cos = ivec3(gd.g_level_bias * floor(cell_cos / gd.g_level_bias));
		int cell = gd.g_offset + icell_cos.z
		                       + icell_cos.y * gd.g_resolution.z
		                       + icell_cos.x * gd.g_resolution.z * gd.g_resolution.y;

		vec3 color = irradiance_from_cell_get(cell, N);

		/* We need this because we render probes in world space (so we need light vector in WS).
		 * And rendering them in local probe space is too much problem. */
		vec3 ws_cell_location = gd.g_corner +
			(gd.g_increment_x * cell_cos.x +
			 gd.g_increment_y * cell_cos.y +
			 gd.g_increment_z * cell_cos.z);

		vec3 ws_point_to_cell = ws_cell_location - W;
		float ws_dist_point_to_cell = length(ws_point_to_cell);
		vec3 ws_light = ws_point_to_cell / ws_dist_point_to_cell;

		vec3 trilinear = mix(1 - trilinear_weight, trilinear_weight, offset);
		float weight = trilinear.x * trilinear.y * trilinear.z;

		/* Precomputed visibility */
		weight *= load_visibility_cell(cell, ws_light, ws_dist_point_to_cell, gd.g_vis_bias, gd.g_vis_bleed, gd.g_vis_range);

		/* Smooth backface test */
		weight *= sqrt(max(0.002, dot(ws_light, N)));

		/* Avoid zero weight */
		weight = max(0.00001, weight);

		weight_accum += weight;
		irradiance_accum += color * weight;
	}

	return irradiance_accum / weight_accum;
}
```
>
- probe_evaluate_grid 函数是在 计算 Irradiance Grids  的入口
<br><br>
- 最主要的是调用 load_visibility_cell ，load_visibility_cell 函数 进行 Precomputed visibility 的工作.
<br><br>


### 总结
- 先渲染到每一个cell的可视距离，保存到rt上
- 在最后计算 Irradiance Grids 的时候，会计算到物体的点到cell的距离，然后再对应rt上的可视距离，来判断这个物体是否被 cell 影响到