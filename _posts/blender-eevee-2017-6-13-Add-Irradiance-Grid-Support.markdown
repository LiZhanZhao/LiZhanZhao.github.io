---
layout:     post
title:      "blender eevee Add Irradiance Grid Support"
subtitle:   ""
date:       2021-1-28 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/13  * Eevee: Add Irradiance Grid support <br>
Early implementation. Slow and still has quality
3 ways of storing irradiance:
- Spherical Harmonics: Have problem with directionnal lighting.
- HL2 diffuse cube: Very low resolution but smooth transitions.
- Diffuse cube: High storage requirement.


> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/IradianceGrid/1.png)
![](/img/Eevee/IradianceGrid/1-1.png)

## 操作
![](/img/Eevee/IradianceGrid/2.png)

## 作用 
场景物体互相产生Irradiance,相当于环境图的Diffuse

## 编译
- 重新生成SLN
- git 定位到  2017/6/14  * Eevee: Add Irradiance Grid support
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## World Probe
为了了解Irradiance Grid 的机制，我们先看World Probe.(World Probe可以理解为全局HDR环境图的光照计算)

### 渲染
*eevee_lightprobes.c*
```
void EEVEE_lightprobes_refresh(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	...
	if (e_data.update_world) {
		render_world_to_probe(sldata, psl);
		glossy_filter_probe(sldata, psl, 0);
		diffuse_filter_probe(sldata, psl, 0);

		e_data.update_world = false;

		if (!e_data.world_ready_to_shade) {
			e_data.world_ready_to_shade = true;
			pinfo->num_render_cube = 1;
		}

		DRW_viewport_request_redraw();
	}
	...
}
```
>
- render_world_to_probe  的目的是为了渲染 "环境图" 到 probe_fb framebuffer 中，这个"环境图"是可以通过连Shader连出来的。
- glossy_filter_probe 的目的是利用 lightprobe_vert.glsl, lightprobe_geom.glsl, lightprobe_filter_glossy_frag.glsl 后处理，计算出来IBL需要用到的贴图
- diffuse_filter_probe 这个是重点，下面详细

### diffuse_filter_probe

*eevee_private.h*
```
/* Only define one of these. */
// #define IRRADIANCE_SH_L2
// #define IRRADIANCE_CUBEMAP
#define IRRADIANCE_HL2
```
>
- 这里定义了一些宏，现在先解析 IRRADIANCE_HL2 的情况

*eevee_lightprobes.c*
```
/* Diffuse filter probe_rt to irradiance_pool at index probe_idx */
static void diffuse_filter_probe(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, int cell_idx)
{
	EEVEE_LightProbesInfo *pinfo = sldata->probes;

	/* TODO do things properly */
	float lodmax = pinfo->lodmax;

	/* 4 - Compute spherical harmonics */
	/* Tweaking parameters to balance perf. vs precision */
	DRW_framebuffer_bind(sldata->probe_filter_fb);
	DRW_texture_generate_mipmaps(sldata->probe_rt);

	/* Bind the right texture layer (one layer per irradiance grid) */
	DRW_framebuffer_texture_detach(sldata->probe_pool);
	DRW_framebuffer_texture_attach(sldata->probe_filter_fb, sldata->irradiance_pool, 0, 0);

	/* find cell position on the virtual 3D texture */
	/* NOTE : Keep in sync with load_irradiance_cell() */
#if defined(IRRADIANCE_SH_L2)
	int size[2] = {3, 3};
#elif defined(IRRADIANCE_CUBEMAP)
	int size[2] = {8, 8};
	pinfo->samples_ct = 1024.0f;
#elif defined(IRRADIANCE_HL2)
	int size[2] = {3, 2};
	pinfo->samples_ct = 1024.0f;
#endif

	int cell_per_row = IRRADIANCE_POOL_SIZE / size[0];
	int x = size[0] * (cell_idx % cell_per_row);
	int y = size[1] * (cell_idx / cell_per_row);

#ifndef IRRADIANCE_SH_L2
	const float bias = 0.0f;
	pinfo->invsamples_ct = 1.0f / pinfo->samples_ct;
	pinfo->lodfactor = bias + 0.5f * log((float)(PROBE_RT_SIZE * PROBE_RT_SIZE) * pinfo->invsamples_ct) / log(2);
	pinfo->lodmax = floorf(log2f(PROBE_RT_SIZE)) - 2.0f;
#else
	pinfo->shres = 32; /* Less texture fetches & reduce branches */
	pinfo->lodmax = 2.0f; /* Improve cache reuse */
#endif

	DRW_framebuffer_viewport_size(sldata->probe_filter_fb, x, y, size[0], size[1]);
	DRW_draw_pass(psl->probe_diffuse_compute);

	/* reattach to have a valid framebuffer. */
	DRW_framebuffer_texture_detach(sldata->irradiance_pool);
	DRW_framebuffer_texture_attach(sldata->probe_filter_fb, sldata->probe_pool, 0, 0);

	/* restore */
	pinfo->lodmax = lodmax;
}

```
>
- DRW_framebuffer_bind(sldata->probe_filter_fb);<br>
	DRW_framebuffer_texture_attach(sldata->probe_filter_fb, sldata->irradiance_pool, 0, 0);<br>
	sldata->irradiance_pool = DRW_texture_create_2D(IRRADIANCE_POOL_SIZE, IRRADIANCE_POOL_SIZE, DRW_TEX_RGBA_16, DRW_TEX_FILTER, NULL);<br>
	这里可以看到 diffuse_filter_probe 最终是把结果渲染到 irradiance_pool 中<br>
	然而 irradiance_pool 是一张1024 * 1024 大小的2D texture
	<br><br>
- DRW_framebuffer_viewport_size(sldata->probe_filter_fb, x, y, size[0], size[1]);<br>
	在渲染之前执行了 viewport_size，这里就是只渲染viewport的 rect(x,y, size[0], size[1]) 中
	<br><br>
- DRW_draw_pass(psl->probe_diffuse_compute);<br>
	利用后处理 + lightprobe_filter_diffuse_frag.glsl 进行渲染Iradiance结果
	<br><br>
- 综上所述 irradianceGrid 一张图里面可以存储很多个 3x2 的数据块，多个数据块组合成一张 irradianceGrid


### lightprobe_filter_diffuse_frag.glsl
```

uniform samplerCube probeHdr;
uniform int probeSize;
uniform float lodFactor;
uniform float lodMax;

in vec3 worldPosition;

out vec4 FragColor;

#define M_4PI 12.5663706143591729

const mat3 CUBE_ROTATIONS[6] = mat3[](
	mat3(vec3( 0.0,  0.0, -1.0),
		 vec3( 0.0, -1.0,  0.0),
		 vec3(-1.0,  0.0,  0.0)),
	mat3(vec3( 0.0,  0.0,  1.0),
		 vec3( 0.0, -1.0,  0.0),
		 vec3( 1.0,  0.0,  0.0)),
	mat3(vec3( 1.0,  0.0,  0.0),
		 vec3( 0.0,  0.0,  1.0),
		 vec3( 0.0, -1.0,  0.0)),
	mat3(vec3( 1.0,  0.0,  0.0),
		 vec3( 0.0,  0.0, -1.0),
		 vec3( 0.0,  1.0,  0.0)),
	mat3(vec3( 1.0,  0.0,  0.0),
		 vec3( 0.0, -1.0,  0.0),
		 vec3( 0.0,  0.0, -1.0)),
	mat3(vec3(-1.0,  0.0,  0.0),
		 vec3( 0.0, -1.0,  0.0),
		 vec3( 0.0,  0.0,  1.0)));

vec3 get_cubemap_vector(vec2 co, int face)
{
	return normalize(CUBE_ROTATIONS[face] * vec3(co * 2.0 - 1.0, 1.0));
}

float area_element(float x, float y)
{
	return atan(x * y, sqrt(x * x + y * y + 1));
}

float texel_solid_angle(vec2 co, float halfpix)
{
	vec2 v1 = (co - vec2(halfpix)) * 2.0 - 1.0;
	vec2 v2 = (co + vec2(halfpix)) * 2.0 - 1.0;

	return area_element(v1.x, v1.y) - area_element(v1.x, v2.y) - area_element(v2.x, v1.y) + area_element(v2.x, v2.y);
}

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

void main()
{
#if defined(IRRADIANCE_SH_L2)
	float pixstep = 1.0 / probeSize;
	float halfpix = pixstep / 2.0;

	/* Downside: leaks negative values, very bandwidth consuming */
	int comp = int(gl_FragCoord.x) % 3 + (int(gl_FragCoord.y) % 3) * 3;

	float weight_accum = 0.0;
	vec3 sh = vec3(0.0);

	for (int face = 0; face < 6; ++face) {
		for (float x = halfpix; x < 1.0; x += pixstep) {
			for (float y = halfpix; y < 1.0; y += pixstep) {
				float weight, coef;
				vec2 facecoord = vec2(x,y);
				vec3 cubevec = get_cubemap_vector(facecoord, face);

				if (comp == 0) {
					coef = 0.282095;
				}
				else if (comp == 1) {
					coef = -0.488603 * cubevec.z * 2.0 / 3.0;
				}
				else if (comp == 2) {
					coef = 0.488603 * cubevec.y * 2.0 / 3.0;
				}
				else if (comp == 3) {
					coef = -0.488603 * cubevec.x * 2.0 / 3.0;
				}
				else if (comp == 4) {
					coef = 1.092548 * cubevec.x * cubevec.z * 1.0 / 4.0;
				}
				else if (comp == 5) {
					coef = -1.092548 * cubevec.z * cubevec.y * 1.0 / 4.0;
				}
				else if (comp == 6) {
					coef = 0.315392 * (3.0 * cubevec.y * cubevec.y - 1.0) * 1.0 / 4.0;
				}
				else if (comp == 7) {
					coef = 1.092548 * cubevec.x * cubevec.y * 1.0 / 4.0;
				}
				else { /* (comp == 8) */
					coef = 0.546274 * (cubevec.x * cubevec.x - cubevec.z * cubevec.z) * 1.0 / 4.0;
				}

				weight = texel_solid_angle(facecoord, halfpix);

				vec4 sample = textureLod(probeHdr, cubevec, lodMax);
				sh += sample.rgb * coef * weight;
				weight_accum += weight;
			}
		}
	}
	sh *= M_4PI / weight_accum;

	FragColor = vec4(sh, 1.0);
#else
#if defined(IRRADIANCE_CUBEMAP)
	/* Downside: Need lots of memory for storage, distortion due to octahedral mapping */
	const vec2 map_size = vec2(16.0);
	const vec2 texelSize = 1.0 / map_size;
	vec2 uvs = mod(gl_FragCoord.xy, map_size) * texelSize;
	const float paddingSize = 1.0;

	/* Add a N pixel border to ensure filtering is correct
	 * for N mipmap levels. */
	uvs += uvs * texelSize * paddingSize * 2.0;
	uvs -= texelSize * paddingSize;

	/* edge mirroring : only mirror if directly adjacent
	 * (not diagonally adjacent) */
	vec2 m = abs(uvs - 0.5) + 0.5;
	vec2 f = floor(m);
	if (f.x - f.y != 0.0) {
		uvs = 1.0 - uvs;
	}

	/* clamp to [0-1] */
	uvs = fract(uvs);

	/* get cubemap vector */
	vec3 cubevec = octahedral_to_cubemap_proj(uvs);

#elif defined(IRRADIANCE_HL2)
	/* Downside: very very low resolution (6 texels), bleed lighting because of interpolation */
	int x = int(gl_FragCoord.x) % 3;
	int y = int(gl_FragCoord.y) % 2;

	vec3 cubevec = vec3(1.0, 0.0, 0.0);

	if (x == 1) {
		cubevec = cubevec.yxy;
	}
	else if (x == 2) {
		cubevec = cubevec.yyx;
	}

	if (y == 1) {
		cubevec = -cubevec;
	}
#endif

	vec3 N, T, B, V;

	N = normalize(cubevec);

	make_orthonormal_basis(N, T, B); /* Generate tangent space */

	/* Integrating Envmap */
	float weight = 0.0;
	vec3 out_radiance = vec3(0.0);
	for (float i = 0; i < sampleCount; i++) {
		vec3 L = sample_hemisphere(i, N, T, B); /* Microfacet normal */
		float NL = dot(N, L);

		if (NL > 0.0) {
			/* Coarse Approximation of the mapping distortion
			 * Unit Sphere -> Cubemap Face */
			const float dist = 4.0 * M_PI / 6.0;
			float pdf = pdf_hemisphere();
			/* http://http.developer.nvidia.com/GPUGems3/gpugems3_ch20.html : Equation 13 */
			float lod = clamp(lodFactor - 0.5 * log2(pdf * dist), 0.0, lodMax) ;

			out_radiance += textureLod(probeHdr, L, lod).rgb * NL / pdf;
		}
		weight += 1.0;
	}

	FragColor = vec4(out_radiance / weight, 1.0);
#endif
}
```
>
- 下面只考虑 IRRADIANCE_HL2 的情况
- <br>
```  
int x = int(gl_FragCoord.x) % 3;
int y = int(gl_FragCoord.y) % 2;
vec3 cubevec = vec3(1.0, 0.0, 0.0);
if (x == 1) {
	cubevec = cubevec.yxy;
}
else if (x == 2) {
	cubevec = cubevec.yyx;
}
if (y == 1) {
	cubevec = -cubevec;
}
```
<br>
得到的结果就是 (0,0) -> (1, 0, 0) <br> 
				(1,0) -> (0, 1, 0) <br>
				 (2,0) -> (0, 0, 1) <br>
				 (0,1) -> (-1, 0, 0) <br> 
				(1,1) -> (0, -1, 0) <br>
				 (2,1) -> (0, 0, -1) <br>
				<br>
结论就是，3x2, 第一行 的3个元素都是存储cubevec为 (1, 0, 0), (0, 1, 0), (0, 0, 1) 计算出来的结果 <br>
第二行的3个元素都是存储cubevec为 (-1, 0, 0), (0, -1, 0), (0, 0, -1) 计算出来的结果


### 应用
*lit_surface_frag.glsl*


```

/* World probe */
if (spec_accum.a < 1.0 || diff_accum.a < 1.0) {
	ProbeData pd = probes_data[0];

	IrradianceData ir_data = load_irradiance_cell(0, sd.N);

	vec3 spec = textureLod_octahedron(probeCubes, vec4(spec_dir, 0), roughness * lodMax).rgb;
	vec3 diff = compute_irradiance(sd.N, ir_data);

	diff_accum.rgb += diff * (1.0 - diff_accum.a);
	spec_accum.rgb += spec * (1.0 - spec_accum.a);
}
```
>
- 最主要的下面两个函数
- IrradianceData ir_data = load_irradiance_cell(0, sd.N); 
- vec3 diff = compute_irradiance(sd.N, ir_data);

### load_irradiance_cell
```

IrradianceData load_irradiance_cell(int cell, vec3 N)
{
	/* Keep in sync with diffuse_filter_probe() */

#if defined(IRRADIANCE_CUBEMAP)

	#define AMBIANT_CUBESIZE 8
	ivec2 cell_co = ivec2(AMBIANT_CUBESIZE);
	int cell_per_row = textureSize(irradianceGrid, 0).x / cell_co.x;
	cell_co.x *= cell % cell_per_row;
	cell_co.y *= cell / cell_per_row;

	vec2 texelSize = 1.0 / vec2(AMBIANT_CUBESIZE);

	vec2 uvs = mapping_octahedron(N, texelSize);
	uvs *= vec2(AMBIANT_CUBESIZE) / vec2(textureSize(irradianceGrid, 0));
	uvs += vec2(cell_co) / vec2(textureSize(irradianceGrid, 0));

	IrradianceData ir;
	ir.color = texture(irradianceGrid, uvs).rgb;

#elif defined(IRRADIANCE_SH_L2)

	ivec2 cell_co = ivec2(3, 3);
	int cell_per_row = textureSize(irradianceGrid, 0).x / cell_co.x;
	cell_co.x *= cell % cell_per_row;
	cell_co.y *= cell / cell_per_row;

	ivec3 ofs = ivec3(0, 1, 2);

	IrradianceData ir;
	ir.shcoefs[0] = texelFetch(irradianceGrid, cell_co + ofs.xx, 0).rgb;
	ir.shcoefs[1] = texelFetch(irradianceGrid, cell_co + ofs.yx, 0).rgb;
	ir.shcoefs[2] = texelFetch(irradianceGrid, cell_co + ofs.zx, 0).rgb;
	ir.shcoefs[3] = texelFetch(irradianceGrid, cell_co + ofs.xy, 0).rgb;
	ir.shcoefs[4] = texelFetch(irradianceGrid, cell_co + ofs.yy, 0).rgb;
	ir.shcoefs[5] = texelFetch(irradianceGrid, cell_co + ofs.zy, 0).rgb;
	ir.shcoefs[6] = texelFetch(irradianceGrid, cell_co + ofs.xz, 0).rgb;
	ir.shcoefs[7] = texelFetch(irradianceGrid, cell_co + ofs.yz, 0).rgb;
	ir.shcoefs[8] = texelFetch(irradianceGrid, cell_co + ofs.zz, 0).rgb;

#else /* defined(IRRADIANCE_HL2) */

	ivec2 cell_co = ivec2(3, 2);
	int cell_per_row = textureSize(irradianceGrid, 0).x / cell_co.x;
	cell_co.x *= cell % cell_per_row;
	cell_co.y *= cell / cell_per_row;

	ivec3 is_negative = ivec3(step(0.0, -N));

	IrradianceData ir;
	ir.cubesides[0] = texelFetch(irradianceGrid, cell_co + ivec2(0, is_negative.x), 0).rgb;
	ir.cubesides[1] = texelFetch(irradianceGrid, cell_co + ivec2(1, is_negative.y), 0).rgb;
	ir.cubesides[2] = texelFetch(irradianceGrid, cell_co + ivec2(2, is_negative.z), 0).rgb;

#endif

	return ir;
}
```
> 
- <br>
```
ivec2 cell_co = ivec2(3, 2);
int cell_per_row = textureSize(irradianceGrid, 0).x / cell_co.x;
cell_co.x *= cell % cell_per_row;
cell_co.y *= cell / cell_per_row;
```
在IRRADIANCE_HL2的情况下，因为worldProbe默认存储在iradianceGrid的第0索引中，3x2个像素，这里就是计算第一个像素的uv坐标，为了之后正确采样。
<br><br>
- <br>
```
ivec3 is_negative = ivec3(step(0.0, -N));
IrradianceData ir;
ir.cubesides[0] = texelFetch(irradianceGrid, cell_co + ivec2(0, is_negative.x), 0).rgb;
ir.cubesides[1] = texelFetch(irradianceGrid, cell_co + ivec2(1, is_negative.y), 0).rgb;
ir.cubesides[2] = texelFetch(irradianceGrid, cell_co + ivec2(2, is_negative.z), 0).rgb;
```
is_negative = ivec3(step(0.0, -N)); <br>
这里就是判断顶点的法线的分量的正负, 因为上面diffuse_filter_probe说过，存储radiance的时候，第一行 的3个元素都是存储cubevec为 (1, 0, 0), (0, 1, 0), (0, 0, 1) 计算出来的结果 <br>
第二行的3个元素都是存储cubevec为 (-1, 0, 0), (0, -1, 0), (0, 0, -1) 计算出来的结果, 所以就需要判断N的分量是否是负数
<br><br>
- 所以load_irradiance_cell就是根据cell索引来采样irradianceGrid的数据


### compute_irradiance
```


vec3 spherical_harmonics_L2(vec3 N, vec3 shcoefs[9])
{
	vec3 sh = vec3(0.0);

	sh += 0.282095 * shcoefs[0];

	sh += -0.488603 * N.z * shcoefs[1];
	sh += 0.488603 * N.y * shcoefs[2];
	sh += -0.488603 * N.x * shcoefs[3];

	sh += 1.092548 * N.x * N.z * shcoefs[4];
	sh += -1.092548 * N.z * N.y * shcoefs[5];
	sh += 0.315392 * (3.0 * N.y * N.y - 1.0) * shcoefs[6];
	sh += -1.092548 * N.x * N.y * shcoefs[7];
	sh += 0.546274 * (N.x * N.x - N.z * N.z) * shcoefs[8];

	return sh;
}

vec3 hl2_basis(vec3 N, vec3 cubesides[3])
{
	vec3 irradiance = vec3(0.0);

	vec3 n_squared = N * N;

	irradiance += n_squared.x * cubesides[0];
	irradiance += n_squared.y * cubesides[1];
	irradiance += n_squared.z * cubesides[2];

	return irradiance;
}


vec3 compute_irradiance(vec3 N, IrradianceData ird)
{
#if defined(IRRADIANCE_CUBEMAP)
	return ird.color;
#elif defined(IRRADIANCE_SH_L2)
	return spherical_harmonics_L2(N, ird.shcoefs);
#else /* defined(IRRADIANCE_HL2) */
	return hl2_basis(N, ird.cubesides);
#endif
}
```
>
- 这里最最最简单的理解就是利用SH计算光照


## Grid Probe
### 渲染
*eevee_lightprobes.c*
```
for (int i = 0; (ob = pinfo->probes_grid_ref[i]) && (i < MAX_GRID); i++) {
	EEVEE_LightProbeEngineData *ped = EEVEE_lightprobe_data_get(ob);

	if (ped->need_update) {
		update_irradiance_probe(sldata, psl, i);

		ped->need_update = false;

		if (!ped->ready_to_shade) {
			pinfo->num_render_grid++;
			ped->ready_to_shade = true;
		}

		DRW_viewport_request_redraw();

		/* Only do one probe per frame */
		break;
	}
}
```
>
- 主要的逻辑都在 update_irradiance_probe 中


### update_irradiance_probe

```
static void update_irradiance_probe(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, int probe_idx)
{
	EEVEE_LightProbesInfo *pinfo = sldata->probes;
	EEVEE_LightGrid *egrid = &pinfo->grid_data[probe_idx];
	Object *ob = pinfo->probes_grid_ref[probe_idx];
	LightProbe *prb = (LightProbe *)ob->data;

	/* Temporary Remove all grids */
	int tmp_num_render_grid = pinfo->num_render_grid;
	int tmp_num_render_cube = pinfo->num_render_cube;
	pinfo->num_render_grid = 0;
	pinfo->num_render_cube = 0;

	/* Render a cubemap and compute irradiance for every point inside the irradiance grid */
	for (int i = 0; i < egrid->resolution[0]; ++i)	{
		for (int j = 0; j < egrid->resolution[1]; ++j)	{
			for (int k = 0; k < egrid->resolution[2]; ++k)	{
				float pos[3], tmp[3];
				int cell = egrid->offset + k + j * egrid->resolution[2] + i * egrid->resolution[2] * egrid->resolution[1];

				/* Compute world position of the sample */
				copy_v3_v3(pos, egrid->corner);
				mul_v3_v3fl(tmp, egrid->increment_x, (float)i);
				add_v3_v3(pos, tmp);
				mul_v3_v3fl(tmp, egrid->increment_y, (float)j);
				add_v3_v3(pos, tmp);
				mul_v3_v3fl(tmp, egrid->increment_z, (float)k);
				add_v3_v3(pos, tmp);

				/* TODO Remove specular */
				render_scene_to_probe(sldata, psl, pos, prb->clipsta, prb->clipend);
				/* TODO Do not update texture while rendering but write to CPU memory (or another buffer).
				 * This will allow "multiple bounces" computation. */
				diffuse_filter_probe(sldata, psl, cell);
			}
		}
	}

	/* Restore */
	pinfo->num_render_grid = tmp_num_render_grid;
	pinfo->num_render_cube = tmp_num_render_cube;

	/* TODO save in DNA / blendfile */
}
```
>
- 大体上去看update_irradiance_probe的话，会发现渲染流程比较简单，就是遍历 egrid->resolution[0] 和 egrid->resolution[1] 和 egrid->resolution[2]，3层循环，创建多个cell，计算cell的位置，然后在cell进行拍摄，拍cubemap的6个面，得到cubemap，然后就把这个cubemap直接给diffuse_filter_probe进行计算，但是这里的diffuse_filter_probe有一些不一样的地方就是，会传入的 cell_index 是由变化的，然后就按照上述的，计算一块一块数据，然后填充到 irradianceGrid 2D texture 中。



### 应用
*eevee_lightprobes.c*
```
int offset = 1; /* to account for the world probe */
for (int i = 0; (ob = pinfo->probes_grid_ref[i]) && (i < MAX_GRID); i++) {
	LightProbe *probe = (LightProbe *)ob->data;
	EEVEE_LightGrid *egrid = &pinfo->grid_data[i];

	egrid->offset = offset;

	/* Set offset for the next grid */
	offset += probe->grid_resolution_x * probe->grid_resolution_y * probe->grid_resolution_z;

	/* Update transforms */
	float tmp[4][4] = {
		{2.0f, 0.0f, 0.0f, 0.0f},
		{0.0f, 2.0f, 0.0f, 0.0f},
		{0.0f, 0.0f, 2.0f, 0.0f},
		{-1.0f, -1.0f, -1.0f, 1.0f}
	};
	float tmp_grid_mat[4][4] = {
		{1.0f / (float)(probe->grid_resolution_x + 1), 0.0f, 0.0f, 0.0f},
		{0.0f, 1.0f / (float)(probe->grid_resolution_y + 1), 0.0f, 0.0f},
		{0.0f, 0.0f, 1.0f / (float)(probe->grid_resolution_z + 1), 0.0f},
		{0.0f, 0.0f, 0.0f, 1.0f}
	};
	mul_m4_m4m4(tmp, tmp, tmp_grid_mat);
	mul_m4_m4m4(egrid->mat, ob->obmat, tmp);
	invert_m4(egrid->mat);

	float one_div_res[3];
	one_div_res[0] = 2.0f / (float)(probe->grid_resolution_x + 1);
	one_div_res[1] = 2.0f / (float)(probe->grid_resolution_y + 1);
	one_div_res[2] = 2.0f / (float)(probe->grid_resolution_z + 1);

	/* First cell. */
	copy_v3_v3(egrid->corner, one_div_res);
	add_v3_fl(egrid->corner, -1.0f);
	mul_m4_v3(ob->obmat, egrid->corner);

	/* Opposite neighbor cell. */
	copy_v3_fl3(egrid->increment_x, one_div_res[0], 0.0f, 0.0f);
	add_v3_v3(egrid->increment_x, one_div_res);
	add_v3_fl(egrid->increment_x, -1.0f);
	mul_m4_v3(ob->obmat, egrid->increment_x);
	sub_v3_v3(egrid->increment_x, egrid->corner);

	copy_v3_fl3(egrid->increment_y, 0.0f, one_div_res[1], 0.0f);
	add_v3_v3(egrid->increment_y, one_div_res);
	add_v3_fl(egrid->increment_y, -1.0f);
	mul_m4_v3(ob->obmat, egrid->increment_y);
	sub_v3_v3(egrid->increment_y, egrid->corner);

	copy_v3_fl3(egrid->increment_z, 0.0f, 0.0f, one_div_res[2]);
	add_v3_v3(egrid->increment_z, one_div_res);
	add_v3_fl(egrid->increment_z, -1.0f);
	mul_m4_v3(ob->obmat, egrid->increment_z);
	sub_v3_v3(egrid->increment_z, egrid->corner);

	copy_v3_v3_int(egrid->resolution, &probe->grid_resolution_x);
}

```
>
- 这里是Update Shader Unform
- <br>
```
/* Update transforms */
float tmp[4][4] = {
	{2.0f, 0.0f, 0.0f, 0.0f},
	{0.0f, 2.0f, 0.0f, 0.0f},
	{0.0f, 0.0f, 2.0f, 0.0f},
	{-1.0f, -1.0f, -1.0f, 1.0f}
};
float tmp_grid_mat[4][4] = {
	{1.0f / (float)(probe->grid_resolution_x + 1), 0.0f, 0.0f, 0.0f},
	{0.0f, 1.0f / (float)(probe->grid_resolution_y + 1), 0.0f, 0.0f},
	{0.0f, 0.0f, 1.0f / (float)(probe->grid_resolution_z + 1), 0.0f},
	{0.0f, 0.0f, 0.0f, 1.0f}
};
mul_m4_m4m4(tmp, tmp, tmp_grid_mat);
mul_m4_m4m4(egrid->mat, ob->obmat, tmp);
invert_m4(egrid->mat);
```
<br>
个人理解就是为了把物体变成到grid的局部空间上，然后得到相对于grid的坐标，然后再把坐标进行归一化，也就是  除以 (probe->grid_resolution_x + 1), 得到了 [0,1] 值，然后再变成 [-1,1]

*lit_surface_frag.glsl*
```
for (int i = 0; i < MAX_GRID && i < grid_count; ++i) {
	GridData gd = grids_data[i];

	vec3 localpos = (gd.localmat * vec4(sd.W, 1.0)).xyz;

	vec3 localpos_max = vec3(gd.g_resolution + ivec3(1)) - localpos;
	float fade = min(1.0, min_v3(min(localpos_max, localpos)));

	if (fade > 0.0) {
		localpos -= 1.0;
		vec3 localpos_floored = floor(localpos);
		vec3 trilinear_weight = fract(localpos); /* fract(-localpos) */

		float weight_accum = 0.0;
		vec3 irradiance_accum = vec3(0.0);

		/* For each neighboor cells */
		for (int i = 0; i < 8; ++i) {
			ivec3 offset = ivec3(i, i >> 1, i >> 2) & ivec3(1);
			vec3 cell_cos = clamp(localpos_floored + vec3(offset), vec3(0.0), vec3(gd.g_resolution) - 1.0);

			/* We need this because we render probes in world space (so we need light vector in WS).
				* And rendering them in local probe space is too much problem. */
			vec3 ws_cell_location = gd.g_corner +
				(gd.g_increment_x * cell_cos.x +
					gd.g_increment_y * cell_cos.y +
					gd.g_increment_z * cell_cos.z);
			vec3 ws_point_to_cell = ws_cell_location - sd.W;
			vec3 ws_light = normalize(ws_point_to_cell);

			vec3 trilinear = mix(1 - trilinear_weight, trilinear_weight, offset);
			float weight = trilinear.x * trilinear.y * trilinear.z;

			/* Smooth backface test */
			// weight *= max(0.005, dot(ws_light, sd.N));

			/* Avoid zero weight */
			weight = max(0.00001, weight);

			vec3 color = get_cell_color(ivec3(cell_cos), gd.g_resolution, gd.g_offset, sd.N);

			weight_accum += weight;
			irradiance_accum += color * weight;
		}

		vec3 indirect_diffuse = irradiance_accum / weight_accum;

		// float influ_diff = min(fade, (1.0 - spec_accum.a));
		float influ_diff = min(1.0, (1.0 - spec_accum.a));

		diff_accum.rgb += indirect_diffuse * influ_diff;
		diff_accum.a += influ_diff;

		// return texture(irradianceGrid, sd.W.xy).rgb;
	}
}
```
>
- 这里思路就是，就是判断物体是否在某一个grid的范围内，如果在的话，就计算grid哪一些cell对他有影响，加权平均，这里都会考虑多个grid对同一个物体的影响
- get_cell_color 和上面的很相似的，都是某一个cell怎么去采样 irradianceGrid

```
vec3 get_cell_color(ivec3 localpos, ivec3 gridres, int offset, vec3 ir_dir)
{
	/* Keep in sync with update_irradiance_probe */

	int cell = offset + localpos.z + localpos.y * gridres.z + localpos.x * gridres.z * gridres.y;
	IrradianceData ir_data = load_irradiance_cell(cell, ir_dir);
	return compute_irradiance(ir_dir, ir_data);
}

```