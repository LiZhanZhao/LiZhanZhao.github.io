---
layout:     post
title:      "blender eevee Replace Cubemaps by octahedron maps for Env Probes"
subtitle:   ""
date:       2020-12-31 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/5/30  * Eevee: Replace Cubemaps by octahedron maps for env. probes.
This enables us to use 2D texture arrays for multiple probes.
There is a little artifact with very high roughness caused elongated pixel due to the projection (along every 90° meridian). <br>  

> SVN : 2017/4/5  MSVC 2015 windows x64 (vc140) Alembic 1.7.1

## 效果
![](/img/Eevee/OctahedronMapsForEnvProbes/1.png)


## 作用 
利用[“上一篇”](http://shaderstore.cn/2020/12/21/blender-eevee-2017-5-20-Move-Cube-Shadows-To-Octahedron-Shadowmaps/) 说的octahedron maps 技术来整理 IBL 高光 计算， 不使用Cubemap来计算了，直接用2D texture，IBL计算的时候，采样也是 2D Texture.

## 编译
- 直接编译。

## 流程

### Hdr 生成 Cubemap

![](/img/Eevee/OctahedronMapsForEnvProbes/2.png)

![](/img/Eevee/OctahedronMapsForEnvProbes/3.png)
>
- Cubemap 顺序 : X -X Y -Y Z -Z
- 这里可以参考 : [这里](http://shaderstore.cn/2020/07/31/blender-eevee-2017-4-28-eevee-World-nodetree-gpumaterial-compatibility/)

### Cubemap 生成 2D texture IBL
![](/img/Eevee/OctahedronMapsForEnvProbes/3.png)
![](/img/Eevee/OctahedronMapsForEnvProbes/4.png)
- 这里就是本文要说的事情



## 应用

### lit_surface_frag.glsl 中IBL和阴影

```


vec2 mapping_octahedron(vec3 cubevec, vec2 texel_size)
{
	/* projection onto octahedron */
	cubevec /= dot( vec3(1), abs(cubevec) );

	/* out-folding of the downward faces */
	if ( cubevec.z < 0.0 ) {
		cubevec.xy = (1.0 - abs(cubevec.yx)) * sign(cubevec.xy);
	}

	/* mapping to [0;1]ˆ2 texture space */
	vec2 uvs = cubevec.xy * (0.5) + 0.5;

	/* edge filtering fix */
	uvs *= 1.0 - 2.0 * texel_size;
	uvs += texel_size;

	return uvs;
}

vec4 textureLod_octahedron(sampler2D tex, vec3 cubevec, float lod)
{
	vec2 texelSize = 1.0 / vec2(textureSize(tex, int(lodMax)));

	vec2 uvs = mapping_octahedron(cubevec, texelSize);

	return textureLod(tex, uvs, lod);
}

vec4 texture_octahedron(sampler2DArray tex, vec4 cubevec)
{
	vec2 texelSize = 1.0 / vec2(textureSize(tex, 0));

	vec2 uvs = mapping_octahedron(cubevec.xyz, texelSize);

	return texture(tex, vec3(uvs, cubevec.w));
}


void light_visibility(LightData ld, ShadingData sd, out float vis)
{
	vis = 1.0;

	if (ld.l_type == SPOT) {
		...
	}
	else if (ld.l_type == AREA) {
		...
	}

	/* shadowing */
	if (ld.l_shadowid >= (MAX_SHADOW_MAP + MAX_SHADOW_CUBE)) {
		/* Shadow Cascade */
		...
	}
	else if (ld.l_shadowid >= 0.0) {
		/* Shadow Cube */
		float shid = ld.l_shadowid;
		ShadowCubeData scd = shadows_cube_data[int(shid)];

		vec3 cubevec = sd.W - ld.l_position;
		float dist = length(cubevec);

		float z = texture_octahedron(shadowCubes, vec4(cubevec, shid)).r;

		float esm_test = min(1.0, exp(-5.0 * dist) * z);
		float sh_test = step(0, z - dist);

		vis *= esm_test;
	}
}


vec3 eevee_surface_lit(vec3 world_normal, vec3 albedo, vec3 f0, float roughness, float ao)
{
	...

	/* Envmaps */
	vec2 uv = lut_coords(dot(sd.N, sd.V), roughness);
	vec3 brdf_lut = texture(brdfLut, uv).rgb;
	vec3 Li = textureLod_octahedron(probeFiltered, spec_dir, roughness * lodMax).rgb;
	indirect_radiance += Li * F_ibl(f0, brdf_lut.rg);
	indirect_radiance += spherical_harmonics(sd.N, shCoefs) * albedo;

	return radiance + indirect_radiance * ao;
}

```
>
- light_visibility 函数中  float z = texture_octahedron(shadowCubes, vec4(cubevec, shid)).r; 计算影子
- eevee_surface_lit 函数中 vec3 Li = textureLod_octahedron(probeFiltered, spec_dir, roughness * lodMax).rgb; 计算IBL
- 用到的知识点和  [“上一篇”](http://shaderstore.cn/2020/12/21/blender-eevee-2017-5-20-Move-Cube-Shadows-To-Octahedron-Shadowmaps/) 基本一致， texture_octahedron 和 textureLod_octahedron 都是为了把 vec3 -> uv，把以前采样一张cubemap的图，转换为采样2D图。


### probe_filter_frag.glsl 

```

uniform samplerCube probeHdr;
uniform float roughnessSquared;
uniform float texelSize;
uniform float lodFactor;
uniform float lodMax;
uniform float paddingSize;

in vec3 worldPosition;

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

void main() {
	vec2 uvs = gl_FragCoord.xy * texelSize;

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

	vec3 N, T, B, V;

    
	vec3 R = normalize(cubevec);
    // ------------------------------------------------------------------ 分割线

	/* Isotropic assumption */
	N = V = R;

	make_orthonormal_basis(N, T, B); /* Generate tangent space */

	/* Noise to dither the samples */
	/* Note : ghosting is better looking than noise. */
	// setup_noise();

	/* Integrating Envmap */
	float weight = 0.0;
	vec3 out_radiance = vec3(0.0);
	for (float i = 0; i < sampleCount; i++) {
		vec3 H = sample_ggx(i, roughnessSquared, N, T, B); /* Microfacet normal */
		vec3 L = -reflect(V, H);
		float NL = dot(N, L);

		if (NL > 0.0) {
			float NH = max(1e-8, dot(N, H)); /* cosTheta */

			/* Coarse Approximation of the mapping distortion
			 * Unit Sphere -> Cubemap Face */
			const float dist = 4.0 * M_PI / 6.0;
			float pdf = pdf_ggx_reflect(NH, roughnessSquared);
			/* http://http.developer.nvidia.com/GPUGems3/gpugems3_ch20.html : Equation 13 */
			float lod = clamp(lodFactor - 0.5 * log2(pdf * dist), 0.0, lodMax) ;

			out_radiance += textureLod(probeHdr, L, lod).rgb * NL;
			weight += NL;
		}
	}

	FragColor = vec4(out_radiance / weight, 1.0);
}
```
>
- probe_filter_frag 本来的作用就是生成 IBL Env Cubemap，现在就变成了生成 IBL Env 2D Texture。
- 分割线以上的，octahedral_to_cubemap_proj 就是为了把 uv 转为 vec3 , 这样就不用生成cubemap了，而是生成一张2D texture。


### 传入参数 给 probe_filter_frag.glsl 

*eevee_probes.c*
```

void EEVEE_refresh_probe(EEVEE_Data *vedata)
{
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_ProbesInfo *pinfo = stl->probes;

	float projmat[4][4];

	/* 1 - Render to cubemap target using geometry shader. */
	/* We don't need to clear since we render the background. */
	pinfo->layer = 0;
	perspective_m4(projmat, -0.1f, 0.1f, -0.1f, 0.1f, 0.1f, 100.0f);
	for (int i = 0; i < 6; ++i) {
		mul_m4_m4m4(pinfo->probemat[i], projmat, cubefacemat[i]);
	}

	DRW_framebuffer_bind(fbl->probe_fb);
	DRW_draw_pass(psl->probe_background);

	/* 2 - Let gpu create Mipmaps for Filtered Importance Sampling. */
	/* Bind next framebuffer to be able to gen. mips for probe_rt. */
	DRW_framebuffer_bind(fbl->probe_filter_fb);
	DRW_texture_generate_mipmaps(txl->probe_rt);

	/* 3 - Render to probe array to the specified layer, do prefiltering. */
	/* Detach to rebind the right mipmap. */
	DRW_framebuffer_texture_detach(txl->probe_pool);
	float mipsize = PROBE_SIZE;
	const int maxlevel = (int)floorf(log2f(PROBE_SIZE));
	const int min_lod_level = 3;
	for (int i = 0; i < maxlevel - min_lod_level; i++) {
		float bias = (i == 0) ? 0.0f : 1.0f;
		pinfo->texel_size = 1.0f / mipsize;
		pinfo->padding_size = powf(2.0f, (float)(maxlevel - min_lod_level - 1 - i));
		/* XXX : WHY THE HECK DO WE NEED THIS ??? */
		/* padding is incorrect without this! float precision issue? */
		if (pinfo->padding_size > 32) {
			pinfo->padding_size += 5;
		}
		if (pinfo->padding_size > 16) {
			pinfo->padding_size += 4;
		}
		else if (pinfo->padding_size > 8) {
			pinfo->padding_size += 2;
		}
		else if (pinfo->padding_size > 4) {
			pinfo->padding_size += 1;
		}
		pinfo->layer = 0;
		pinfo->roughness = (float)i / ((float)maxlevel - 4.0f);
		pinfo->roughness *= pinfo->roughness; /* Disney Roughness */
		pinfo->roughness *= pinfo->roughness; /* Distribute Roughness accros lod more evenly */
		CLAMP(pinfo->roughness, 1e-8f, 0.99999f); /* Avoid artifacts */

#if 1 /* Variable Sample count (fast) */
		switch (i) {
			case 0: pinfo->samples_ct = 1.0f; break;
			case 1: pinfo->samples_ct = 16.0f; break;
			case 2: pinfo->samples_ct = 32.0f; break;
			case 3: pinfo->samples_ct = 64.0f; break;
			default: pinfo->samples_ct = 128.0f; break;
		}
#else /* Constant Sample count (slow) */
		pinfo->samples_ct = 1024.0f;
#endif

		pinfo->invsamples_ct = 1.0f / pinfo->samples_ct;
		pinfo->lodfactor = bias + 0.5f * log((float)(PROBE_CUBE_SIZE * PROBE_CUBE_SIZE) * pinfo->invsamples_ct) / log(2);
		pinfo->lodmax = floorf(log2f(PROBE_CUBE_SIZE)) - 2.0f;

		DRW_framebuffer_texture_attach(fbl->probe_filter_fb, txl->probe_pool, 0, i);
		DRW_framebuffer_viewport_size(fbl->probe_filter_fb, mipsize, mipsize);
		DRW_draw_pass(psl->probe_prefilter);
		DRW_framebuffer_texture_detach(txl->probe_pool);

		mipsize /= 2;
		CLAMP_MIN(mipsize, 1);
	}
	/* For shading, save max level of the octahedron map */
	pinfo->lodmax = (float)(maxlevel - min_lod_level) - 1.0f;
	/* reattach to have a valid framebuffer. */
	DRW_framebuffer_texture_attach(fbl->probe_filter_fb, txl->probe_pool, 0, 0);

	/* 4 - Compute spherical harmonics */
	/* Tweaking parameters to balance perf. vs precision */
	pinfo->shres = 16; /* Less texture fetches & reduce branches */
	pinfo->lodfactor = 4.0f; /* Improve cache reuse */
	DRW_framebuffer_bind(fbl->probe_sh_fb);
	DRW_draw_pass(psl->probe_sh_compute);
	DRW_framebuffer_read_data(0, 0, 9, 1, 3, 0, (float *)pinfo->shcoefs);
}
```
>
- 主要是计算 Shader 中需要用到的参数，例如 texelSize, paddingSize 等