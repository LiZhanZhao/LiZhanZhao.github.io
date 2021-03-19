---
layout:     post
title:      "blender eevee Spherical Harmonic Diffuse"
subtitle:   ""
date:       2020-06-03 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## Unity3D 复现效果
*Spherical Harmonic Diffuse*
![](/img/Eevee/SphericalHarmonicDiffuse/1.png)
![](/img/Eevee/SphericalHarmonicDiffuse/2.png)
![](/img/Eevee/SphericalHarmonicDiffuse/3.png)

## 理论

理解可以参考  
<br>
[Diffuse irradiance](https://learnopengl.com/PBR/IBL/Diffuse-irradiance)



## 编译运行
想要运行起来，要修改点东西。 <br> 

更换下面函数就好了。

*eevee.c*
```c

static DRWShadingGroup *eevee_cube_shgroup(struct GPUShader *sh, DRWPass *pass, struct Batch *geom)
{
	DRWShadingGroup *grp = DRW_shgroup_instance_create(sh, pass, geom);

	for (int i = 0; i < 6; ++i)
		DRW_shgroup_dynamic_call_add(grp, NULL);

	return grp;
}



```




## 实践

### 渲染过程

跟 [上一篇](http://shaderstore.cn/2020/05/19/blender-eevee-2017-4-18-eevee-Introduction-of-world-preconvolved-envmap/) 一样，但是

 *EEVEE_refresh_probe((EEVEE_Data *)vedata)* , 会有一些改动


#### EEVEE_refresh_probe
*eevee_probes.c*
```c

void EEVEE_refresh_probe(EEVEE_Data *vedata)
{
	...

	/* 4 - Compute spherical harmonics */
	pinfo->shres = 64;
	DRW_framebuffer_bind(fbl->probe_sh_fb);
	DRW_draw_pass(psl->probe_sh_compute);
	DRW_framebuffer_read_data(0, 0, 9, 1, 3, 0, (float *)pinfo->shcoefs);
}
```

- DRW_draw_pass(psl->probe_sh_compute);; 使用 gpu_shader_fullscreen_vert.glsl  probe_sh_frag.glsl 进行渲染

*gpu_shader_fullscreen_vert.glsl*
```glsl


#if __VERSION__ == 120
	attribute vec2 pos;
	attribute vec2 uvs;
	varying vec4 uvcoordsvar;
#else
	in vec2 pos;
	in vec2 uvs;
	out vec4 uvcoordsvar;
#endif

void main()
{
	uvcoordsvar = vec4(uvs, 0.0, 0.0);
	gl_Position = vec4(pos, 0.0, 1.0);
}

```

*probe_sh_frag.glsl*
```glsl

uniform samplerCube probeHdr;
uniform int probeSize;

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

void main()
{
	float pixstep = 1.0 / probeSize;
	float halfpix = pixstep / 2.0;

	float weight_accum = 0.0;
	vec3 sh = vec3(0.0);

	int shnbr = int(floor(gl_FragCoord.x));

	for (int face = 0; face < 6; ++face) {
		for (float x = halfpix; x < 1.0; x += pixstep) {
			for (float y = halfpix; y < 1.0; y += pixstep) {
				float shcoef;

				vec2 facecoord = vec2(x,y);
				vec3 cubevec = get_cubemap_vector(facecoord, face);
				float weight = texel_solid_angle(facecoord, halfpix);

				if (shnbr == 0) {
					shcoef = 0.282095;
				}
				else if (shnbr == 1) {
					shcoef = -0.488603 * cubevec.z * 2.0 / 3.0;
				}
				else if (shnbr == 2) {
					shcoef = 0.488603 * cubevec.y * 2.0 / 3.0;
				}
				else if (shnbr == 3) {
					shcoef = -0.488603 * cubevec.x * 2.0 / 3.0;
				}
				else if (shnbr == 4) {
					shcoef = 1.092548 * cubevec.x * cubevec.z * 1.0 / 4.0;
				}
				else if (shnbr == 5) {
					shcoef = -1.092548 * cubevec.z * cubevec.y * 1.0 / 4.0;
				}
				else if (shnbr == 6) {
					shcoef = 0.315392 * (3.0 * cubevec.y * cubevec.y - 1.0) * 1.0 / 4.0;
				}
				else if (shnbr == 7) {
					shcoef = 1.092548 * cubevec.x * cubevec.y * 1.0 / 4.0;
				}
				else { /* (shnbr == 8) */
					shcoef = 0.546274 * (cubevec.x * cubevec.x - cubevec.z * cubevec.z) * 1.0 / 4.0;
				}

				vec4 sample = textureCube(probeHdr, cubevec);
				sh += sample.rgb * shcoef * weight;
				weight_accum += weight;
			}
		}
	}

	sh *= M_4PI / weight_accum;

	gl_FragColor = vec4(sh, 1.0);
}

```

#### 优化Spherical Harmonic计算时间
*probe_sh_frag.glsl*
```glsl
uniform float lodBias;
...
vec4 sample = textureCubeLod(probeHdr, cubevec, lodBias);

```

*eevee_probes.c*

```c
/* Tweaking parameters to balance perf. vs precision */
pinfo->shres = 16; /* Less texture fetches & reduce branches */
pinfo->lodfactor = 4.0f; /* Improve cache reuse */
```




#### 传入Shader的数据


使用 probe_filter_frag.glsl 进行渲染 需要传入Shader的数据有

```c++
{
		psl->probe_sh_compute = DRW_pass_create("Probe SH Compute", DRW_STATE_WRITE_COLOR);

		DRWShadingGroup *grp = DRW_shgroup_create(e_data.probe_spherical_harmonic_sh, psl->probe_sh_compute);
		DRW_shgroup_uniform_int(grp, "probeSize", &stl->probes->shres, 1);
		DRW_shgroup_uniform_texture(grp, "probeHdr", txl->probe_rt, 0);

		struct Batch *geom = DRW_cache_fullscreen_quad_get();
		DRW_shgroup_call_add(grp, geom, NULL);
	}
```
### probeSize 的计算
目前 probeSize 直接 64

#### probeHdr 的计算
目前的probeHdr 只是用了一个6个面不同颜色的Cubemap，可以参考 *eevee_probes.c*   EEVEE_probes_init 函数中计算  txl->probe_rt





### 物体渲染

*lit_surface_frag.glsl*
```glsl

...

uniform samplerCube probeFiltered;
uniform float lodMax;

#ifndef USE_LTC
uniform sampler2D brdfLut;
#endif

...

void main()
{
	...

	ShadingData sd;
	sd.N = normalize(worldNormal);
	sd.V = (ProjectionMatrix[3][3] == 0.0) /* if perspective */
	            ? normalize(cameraPos - worldPosition)
	            : normalize(eye);
	sd.W = worldPosition;
	sd.R = reflect(-sd.V, sd.N);

	/* hardcoded test vars */
	vec3 albedo = mix(vec3(0.0, 0.0, 0.0), vec3(0.8, 0.8, 0.8), saturate(worldPosition.y/2));
	vec3 f0 = mix(vec3(0.83, 0.5, 0.1), vec3(0.03, 0.03, 0.03), saturate(worldPosition.y/2));
	vec3 specular = mix(f0, vec3(1.0), pow(max(0.0, 1.0 - dot(sd.N, sd.V)), 5.0));
	float roughness = saturate(worldPosition.x/lodMax);

	sd.spec_dominant_dir = get_specular_dominant_dir(sd.N, sd.R, roughness);

	vec3 radiance = vec3(0.0);

	/* Analitic Lights */
	for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
		LightData ld = lights_data[i];

		sd.l_vector = ld.l_position - worldPosition;
		sd.l_distance = length(sd.l_vector);

		light_common(ld, sd);

		float vis = light_visibility(ld, sd);
		float spec = light_specular(ld, sd, roughness);
		float diff = light_diffuse(ld, sd);

		radiance += vis * (albedo * diff + specular * spec) * ld.l_color;
	}

	/* Envmaps */
	vec2 uv = ltc_coords(dot(sd.N, sd.V), sqrt(roughness));
	vec2 brdf_lut = texture(brdfLut, uv).rg;
	vec3 Li = textureLod(probeFiltered, sd.spec_dominant_dir, roughness * lodMax).rgb;
	radiance += Li * brdf_lut.y + f0 * Li * brdf_lut.x;

	radiance += spherical_harmonics(sd.N, shCoefs) * albedo;

	fragColor = vec4(radiance, 1.0);
}
```
主要是 多了  radiance += spherical_harmonics(sd.N, shCoefs) * albedo , spherical_harmonics 函数来处理Spherical Harmonic Diffuse.


*bsdf_common_lib.glsl*
```glsl

#define spherical_harmonics spherical_harmonics_L2

/* http://seblagarde.wordpress.com/2012/01/08/pi-or-not-to-pi-in-game-lighting-equation/ */
vec3 spherical_harmonics_L1(vec3 N, vec3 shcoefs[9])
{
	vec3 sh = vec3(0.0);

	sh += 0.282095 * shcoefs[0];

	sh += -0.488603 * N.z * shcoefs[1];
	sh += 0.488603 * N.y * shcoefs[2];
	sh += -0.488603 * N.x * shcoefs[3];

	return sh;
}

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


```

### 传入数据到 lit_surface_frag

```c
{
	DRW_shgroup_uniform_vec3(stl->g_data->default_lit_grp, "shCoefs[0]", (float *)stl->probes->shcoefs, 9);
}
```

### shCoefs[0] 

这个就是 上面 EEVEE_refresh_probe 中的 DRW_draw_pass(psl->probe_sh_compute) 计算出来的，DRW_draw_pass(psl->probe_sh_compute) 渲染出 9x1 大小的纹理，然后再将这9个pixel当前shCoefs的传入到 lit_surface_frag 中。

