---
layout:     post
title:      "blender eevee Introduction of world preconvolved envmap"
subtitle:   ""
date:       2020-05-19 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## Unity3D 复现效果
*World Preconvolved Envmap*
![](/img/Eevee/WorldPreconvolvedEnvmap/1.png)

## 理论

理解可以参考  
<br> 
[Specular IBL](https://learnopengl.com/PBR/IBL/Specular-IBL)


## 编译运行
想要运行起来，要修改点东西。 <br> 

更换这几个函数就好了。

*eevee.c*
```c

static DRWShadingGroup *eevee_cube_shgroup(struct GPUShader *sh, DRWPass *pass, struct Batch *geom)
{
	DRWShadingGroup *grp = DRW_shgroup_instance_create(sh, pass, geom);

	for (int i = 0; i < 6; ++i)
		DRW_shgroup_dynamic_call_add(grp, NULL);

	return grp;
}

static void EEVEE_engine_init(void *ved)
{
	...

	if (!e_data.probe_filter_sh) {
		/*
		char *lib_str = NULL;

		DynStr *ds_vert = BLI_dynstr_new();
		BLI_dynstr_append(ds_vert, datatoc_bsdf_common_lib_glsl);
		BLI_dynstr_append(ds_vert, datatoc_bsdf_sampling_lib_glsl);
		lib_str = BLI_dynstr_get_cstring(ds_vert);
		BLI_dynstr_free(ds_vert);

		e_data.probe_filter_sh = DRW_shader_create_with_lib(datatoc_probe_vert_glsl, datatoc_probe_geom_glsl, datatoc_probe_filter_frag_glsl, lib_str,
		                                                    "#define HAMMERSLEY_SIZE 8192\n"
		                                                    "#define NOISE_SIZE 64\n");

		MEM_freeN(lib_str);
		*/

		char *shader_str = NULL;

		DynStr *ds_frag = BLI_dynstr_new();
		BLI_dynstr_append(ds_frag, datatoc_bsdf_common_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_bsdf_sampling_lib_glsl);
		BLI_dynstr_append(ds_frag, datatoc_probe_filter_frag_glsl);
		shader_str = BLI_dynstr_get_cstring(ds_frag);
		BLI_dynstr_free(ds_frag);

		e_data.probe_filter_sh = DRW_shader_create(datatoc_probe_vert_glsl, datatoc_probe_geom_glsl, shader_str,
			"#define HAMMERSLEY_SIZE 8192\n"
			"#define NOISE_SIZE 64\n");

		MEM_freeN(shader_str);
	}

	if (!e_data.tonemap) {
		e_data.tonemap = DRW_shader_create_fullscreen(datatoc_tonemap_frag_glsl, NULL);
	}

	...
}


```




## 实践

### 渲染过程
```c

static void EEVEE_draw_scene(void *vedata)
{
	EEVEE_PassList *psl = ((EEVEE_Data *)vedata)->psl;
	EEVEE_FramebufferList *fbl = ((EEVEE_Data *)vedata)->fbl;

	/* Default framebuffer and texture */
	DefaultFramebufferList *dfbl = DRW_viewport_framebuffer_list_get();
	DefaultTextureList *dtxl = DRW_viewport_texture_list_get();

	/* Refresh Probes */
	EEVEE_refresh_probe((EEVEE_Data *)vedata);

	/* Refresh shadows */
	EEVEE_draw_shadows((EEVEE_Data *)vedata);

	/* Attach depth to the hdr buffer and bind it */	
	DRW_framebuffer_texture_detach(dtxl->depth);
	DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0, 0);
	DRW_framebuffer_bind(fbl->main);

	/* Clear Depth */
	/* TODO do background */
	float clearcol[4] = {0.0f, 0.0f, 0.0f, 1.0f};
	DRW_framebuffer_clear(true, true, false, clearcol, 1.0f);

	DRW_draw_pass(psl->depth_pass);
	DRW_draw_pass(psl->depth_pass_cull);
	DRW_draw_pass(psl->pass);

	/* Restore default framebuffer */
	DRW_framebuffer_texture_detach(dtxl->depth);
	DRW_framebuffer_texture_attach(dfbl->default_fb, dtxl->depth, 0, 0);
	DRW_framebuffer_bind(dfbl->default_fb);

	DRW_draw_pass(psl->tonemap);
}
```
这里主要关注 *EEVEE_refresh_probe((EEVEE_Data *)vedata)* , 在这个函数中最重要的就是计算preconvolved envmap cubemap, shader 会用到。


#### EEVEE_refresh_probe
*eevee_probes.c*
```c

void EEVEE_refresh_probe(EEVEE_Data *vedata)
{
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_PassList *psl = vedata->psl;
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_ProbesInfo *pinfo = stl->probes;

	const bContext *C = DRW_get_context();
	Scene *scene = CTX_data_scene(C);
	World *world = scene->world;

	float projmat[4][4];

	/* 1 - Render to cubemap target using geometry shader. */
	/* We don't need to clear since we render the background. */
	pinfo->layer = 0;
	perspective_m4(projmat, -0.1f, 0.1f, -0.1f, 0.1f, 0.1f, 100.0f);
	for (int i = 0; i < 6; ++i) {
		mul_m4_m4m4(pinfo->probemat[i], projmat, cubefacemat[i]);
	}

	// 这里不跑进去
	/* Debug Tex : Use World 1st Tex Slot */
	if (world && world->mtex[0]) {
		MTex *mtex = world->mtex[0];
		if (mtex && mtex->tex) {
			Tex *tex = mtex->tex;
			if (tex->ima) {
				pinfo->backgroundtex = GPU_texture_from_blender(tex->ima, &tex->iuser, GL_TEXTURE_2D, true, 0.0, 0);

				DRW_framebuffer_bind(fbl->probe_fb);
				DRW_draw_pass(psl->probe_background);
			}
		}
	}

	/* 2 - Let gpu create Mipmaps for Filtered Importance Sampling. */
	/* Bind next framebuffer to be able to write to probe_rt. */
	DRW_framebuffer_bind(fbl->probe_filter_fb);
	DRW_texture_generate_mipmaps(txl->probe_rt);

	/* 3 - Render to probe array to the specified layer, do prefiltering. */
	/* Detach to rebind the right mipmap. */
	DRW_framebuffer_texture_detach(txl->probe_pool);
	float mipsize = PROBE_SIZE * 2;
	int miplevels = 1 + (int)floorf(log2f(PROBE_SIZE));
	for (int i = 0; i < miplevels - 2; i++) {
		float bias = (i == 0) ? 0.0f : 1.0f;

		mipsize /= 2;
		CLAMP_MIN(mipsize, 1);

		pinfo->layer = 0;
		pinfo->roughness = (float)i / ((float)miplevels - 3.0f);
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
		pinfo->lodfactor = bias + 0.5f * log((float)(PROBE_SIZE * PROBE_SIZE) * pinfo->invsamples_ct) / log(2);
		pinfo->lodmax = (float)miplevels - 3.0f;

		DRW_framebuffer_texture_attach(fbl->probe_filter_fb, txl->probe_pool, 0, i);
		DRW_framebuffer_viewport_size(fbl->probe_filter_fb, mipsize, mipsize);
		DRW_draw_pass(psl->probe_prefilter);
		DRW_framebuffer_texture_detach(txl->probe_pool);
	}
	/* reattach to have a valid framebuffer. */
	DRW_framebuffer_texture_attach(fbl->probe_filter_fb, txl->probe_pool, 0, 0);

	/* 4 - Compute spherical harmonics */
	/* TODO */
	// DRW_framebuffer_bind(fbl->probe_sh_fb);
	// DRW_draw_pass(psl->probe_sh);
}
```

- DRW_draw_pass(psl->probe_prefilter); 使用 probe_vert.glsl  probe_geom.glsl  probe_filter_frag.glsl 进行渲染

*probe_vert.glsl*
```glsl
in vec3 pos;

out vec4 vPos;
out int face;

void main() {
	vPos = vec4(pos, 1.0);
	face = gl_InstanceID;
}

```

*probe_geom.glsl*
```glsl

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

uniform int Layer;

in vec4 vPos[];
in int face[];

out vec3 worldPosition;
out vec3 worldNormal;

const vec3 maj_axes[6] = vec3[6](vec3(1.0,  0.0,  0.0), vec3(-1.0,  0.0, 0.0), vec3(0.0, 1.0, 0.0), vec3(0.0, -1.0,  0.0), vec3( 0.0,  0.0, 1.0), vec3( 0.0,  0.0, -1.0));
const vec3 x_axis[6]   = vec3[6](vec3(0.0,  0.0, -1.0), vec3( 0.0,  0.0, 1.0), vec3(1.0, 0.0, 0.0), vec3(1.0,  0.0,  0.0), vec3( 1.0,  0.0, 0.0), vec3(-1.0,  0.0,  0.0));
const vec3 y_axis[6]   = vec3[6](vec3(0.0, -1.0,  0.0), vec3( 0.0, -1.0, 0.0), vec3(0.0, 0.0, 1.0), vec3(0.0,  0.0, -1.0), vec3( 0.0, -1.0, 0.0), vec3( 0.0, -1.0,  0.0));

void main() {
	int f = face[0];
	gl_Layer = Layer + f;

	for (int v = 0; v < 3; ++v) {
		gl_Position = vPos[v];
		worldPosition = x_axis[f] * vPos[v].x + y_axis[f] * vPos[v].y + maj_axes[f];
		EmitVertex();
	}

	EndPrimitive();
}

```



*probe_filter_frag.glsl*
```glsl

uniform samplerCube probeHdr;
uniform float roughnessSquared;
uniform float lodFactor;
uniform float lodMax;

in vec3 worldPosition;

out vec4 FragColor;

void main() {
	vec3 N, T, B, V;

	vec3 R = normalize(worldPosition);

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

			out_radiance += textureCubeLod(probeHdr, L, lod).rgb * NL;
			weight += NL;
		}
	}

	FragColor = vec4(out_radiance / weight, 1.0);
}

```

#### 传入Shader的数据


使用 probe_filter_frag.glsl 进行渲染 需要传入Shader的数据有

```glsl
{
		psl->probe_prefilter = DRW_pass_create("Probe Filtering", DRW_STATE_WRITE_COLOR);

		struct Batch *geom = DRW_cache_fullscreen_quad_get();
		DRWShadingGroup *grp = eevee_cube_shgroup(e_data.probe_filter_sh, psl->probe_prefilter, geom);
		DRW_shgroup_uniform_float(grp, "sampleCount", &stl->probes->samples_ct, 1);
		DRW_shgroup_uniform_float(grp, "invSampleCount", &stl->probes->invsamples_ct, 1);
		DRW_shgroup_uniform_float(grp, "roughnessSquared", &stl->probes->roughness, 1);
		DRW_shgroup_uniform_float(grp, "lodFactor", &stl->probes->lodfactor, 1);
		DRW_shgroup_uniform_float(grp, "lodMax", &stl->probes->lodmax, 1);
		DRW_shgroup_uniform_int(grp, "Layer", &stl->probes->layer, 1);
		DRW_shgroup_uniform_texture(grp, "texHammersley", e_data.hammersley, 0);
		// DRW_shgroup_uniform_texture(grp, "texJitter", e_data.jitter, 1);
		DRW_shgroup_uniform_texture(grp, "probeHdr", txl->probe_rt, 3);
	}
```

#### texHammersley 的计算 (e_data.hammersley)
```c
if (!e_data.hammersley) {
	e_data.hammersley = create_hammersley_sample_texture(8192);
}

static struct GPUTexture *create_hammersley_sample_texture(int samples)
{
	struct GPUTexture *tex;
	float (*texels)[2] = MEM_mallocN(sizeof(float[2]) * samples, "hammersley_tex");
	int i;

	for (i = 0; i < samples; i++) {
		float phi = radical_inverse(i) * 2.0f * M_PI;
		texels[i][0] = cos(phi);
		texels[i][1] = sinf(phi);
	}

	tex = DRW_texture_create_1D(samples, DRW_TEX_RG_16, DRW_TEX_WRAP, (float *)texels);
	MEM_freeN(texels);
	return tex;
}

/* Van der Corput sequence */
 /* From http://holger.dammertz.org/stuff/notes_HammersleyOnHemisphere.html */
static float radical_inverse(int i) {
	unsigned int bits = (unsigned int)i;
	bits = (bits << 16u) | (bits >> 16u);
	bits = ((bits & 0x55555555u) << 1u) | ((bits & 0xAAAAAAAAu) >> 1u);
	bits = ((bits & 0x33333333u) << 2u) | ((bits & 0xCCCCCCCCu) >> 2u);
	bits = ((bits & 0x0F0F0F0Fu) << 4u) | ((bits & 0xF0F0F0F0u) >> 4u);
	bits = ((bits & 0x00FF00FFu) << 8u) | ((bits & 0xFF00FF00u) >> 8u);
	return (float)bits * 2.3283064365386963e-10f;
}

```

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

	/* hardcoded test vars */
	vec3 albedo = vec3(0.0);
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

	fragColor = vec4(radiance, 1.0);
}
```

*bsdf_sampling_lib.glsl*
```glsl

uniform sampler1D texHammersley;
uniform sampler2D texJitter;
uniform float sampleCount;
uniform float invSampleCount;

vec2 jitternoise = vec2(0.0);

void setup_noise(void)
{
	jitternoise = texture(texJitter, gl_FragCoord.xy / NOISE_SIZE, 0).rg; /* Global variable */
}

vec3 hammersley_3d(float i, float invsamplenbr)
{
	vec3 Xi; /* Theta, cos(Phi), sin(Phi) */

	Xi.x = i * invsamplenbr; /* i/samples */
	Xi.x = fract(Xi.x + jitternoise.x);

	int u = int(mod(i + jitternoise.y * HAMMERSLEY_SIZE, HAMMERSLEY_SIZE));

	Xi.yz = texelFetch(texHammersley, u, 0).rg;

	return Xi;
}

vec3 hammersley_3d(float i)
{
	return hammersley_3d(i, invSampleCount);
}

/* -------------- BSDFS -------------- */

float pdf_ggx_reflect(float NH, float a2)
{
	return NH * a2 / D_ggx_opti(NH, a2);
}

vec3 sample_ggx(float nsample, float a2, vec3 N, vec3 T, vec3 B)
{
	vec3 Xi = hammersley_3d(nsample);

	/* Theta is the aperture angle of the cone */
	float z = sqrt( (1.0 - Xi.x) / ( 1.0 + a2 * Xi.x - Xi.x ) ); /* cos theta */
	float r = sqrt( 1.0 - z * z ); /* sin theta */
	float x = r * Xi.y;
	float y = r * Xi.z;

	/* Microfacet Normal */
	vec3 Ht = vec3(x, y, z);

	return tangent_to_world(Ht, N, T, B);
}

/* -- Tangent Space conversion -- */
vec3 tangent_to_world(vec3 vector, vec3 N, vec3 T, vec3 B)
{
	return T * vector.x + B * vector.y + N * vector.z;
}

vec3 world_to_tangent(vec3 vector, vec3 N, vec3 T, vec3 B)
{
	return vec3( dot(T, vector), dot(B, vector), dot(N, vector));
}

void make_orthonormal_basis(vec3 N, out vec3 T, out vec3 B)
{
	vec3 UpVector = abs(N.z) < 0.99999 ? vec3(0.0,0.0,1.0) : vec3(1.0,0.0,0.0);
	T = normalize( cross(UpVector, N) );
	B = cross(N, T);
}


```

### 传入数据到 lit_surface_frag

```c
{
	DRWState state = DRW_STATE_WRITE_COLOR | DRW_STATE_WRITE_DEPTH | DRW_STATE_DEPTH_EQUAL;
	psl->pass = DRW_pass_create("Default Light Pass", state);

	stl->g_data->default_lit_grp = DRW_shgroup_create(e_data.default_lit, psl->pass);
	DRW_shgroup_uniform_block(stl->g_data->default_lit_grp, "light_block", stl->light_ubo, 0);
	DRW_shgroup_uniform_block(stl->g_data->default_lit_grp, "shadow_block", stl->shadow_ubo, 1);
	DRW_shgroup_uniform_int(stl->g_data->default_lit_grp, "light_count", &stl->lamps->num_light, 1);
	DRW_shgroup_uniform_float(stl->g_data->default_lit_grp, "lodMax", &stl->probes->lodmax, 1);
	DRW_shgroup_uniform_vec3(stl->g_data->default_lit_grp, "cameraPos", e_data.camera_pos, 1);
	DRW_shgroup_uniform_texture(stl->g_data->default_lit_grp, "ltcMat", e_data.ltc_mat, 0);
	DRW_shgroup_uniform_texture(stl->g_data->default_lit_grp, "brdfLut", e_data.brdf_lut, 1);
	DRW_shgroup_uniform_texture(stl->g_data->default_lit_grp, "probeFiltered", txl->probe_pool, 2);
	/* NOTE : Adding Shadow Map textures uniform in EEVEE_cache_finish */
}
```
### probeFiltered 

probeFiltered 就是上面的 EEVEE_refresh_probe 计算出来的cubemap

### lodMax

lodMax 也是 EEVEE_refresh_probe 计算出来再传入来的


### 如何计算 brdfLut 
```c
if (!e_data.brdf_lut) {
	e_data.brdf_lut = create_ggx_lut_texture(64, 64);
}


static struct GPUTexture *create_ggx_lut_texture(int UNUSED(w), int UNUSED(h))
{
	struct GPUTexture *tex;
#if 0 /* Used only to generate the LUT values */
	struct GPUFrameBuffer *fb = NULL;
	static float samples_ct = 8192.0f;
	static float inv_samples_ct = 1.0f / 8192.0f;

	char *lib_str = NULL;

	DynStr *ds_vert = BLI_dynstr_new();
	BLI_dynstr_append(ds_vert, datatoc_bsdf_common_lib_glsl);
	BLI_dynstr_append(ds_vert, datatoc_bsdf_sampling_lib_glsl);
	lib_str = BLI_dynstr_get_cstring(ds_vert);
	BLI_dynstr_free(ds_vert);

	struct GPUShader *sh = DRW_shader_create_with_lib(datatoc_probe_vert_glsl, datatoc_probe_geom_glsl, datatoc_bsdf_lut_frag_glsl, lib_str,
	                                                    "#define HAMMERSLEY_SIZE 8192\n"
	                                                    "#define BRDF_LUT_SIZE 64\n"
	                                                    "#define NOISE_SIZE 64\n");

	DRWPass *pass = DRW_pass_create("Probe Filtering", DRW_STATE_WRITE_COLOR);
	DRWShadingGroup *grp = DRW_shgroup_create(sh, pass);
	DRW_shgroup_uniform_float(grp, "sampleCount", &samples_ct, 1);
	DRW_shgroup_uniform_float(grp, "invSampleCount", &inv_samples_ct, 1);
	DRW_shgroup_uniform_texture(grp, "texHammersley", e_data.hammersley, 0);
	DRW_shgroup_uniform_texture(grp, "texJitter", e_data.jitter, 1);

	struct Batch *geom = DRW_cache_fullscreen_quad_get();
	DRW_shgroup_call_add(grp, geom, NULL);

	float *texels = MEM_mallocN(sizeof(float[2]) * w * h, "lut");

	tex = DRW_texture_create_2D(w, h, DRW_TEX_RG_16, DRW_TEX_FILTER, (float *)texels);

	DRWFboTexture tex_filter = {&tex, DRW_BUF_RG_16, DRW_TEX_FILTER};
	DRW_framebuffer_init(&fb, w, h, &tex_filter, 1);

	DRW_framebuffer_bind(fb);
	DRW_draw_pass(pass);

	float *data = MEM_mallocN(sizeof(float[3]) * w * h, "lut");
	glReadBuffer(GL_COLOR_ATTACHMENT0);
	glReadPixels(0, 0, w, h, GL_RGB, GL_FLOAT, data);

	printf("{");
	for (int i = 0; i < w*h * 3; i+=3) {
		printf("%ff, %ff, ", data[i],  data[i+1]); i+=3;
		printf("%ff, %ff, ", data[i],  data[i+1]); i+=3;
		printf("%ff, %ff, ", data[i],  data[i+1]); i+=3;
		printf("%ff, %ff, \n", data[i],  data[i+1]);
	}
	printf("}");

	MEM_freeN(texels);
	MEM_freeN(data);
#else
	float (*texels)[3] = MEM_mallocN(sizeof(float[3]) * 64 * 64, "bsdf lut texels");

	for (int i = 0; i < 64 * 64; i++) {
		texels[i][0] = bsdf_split_sum_ggx[i*2 + 0];
		texels[i][1] = bsdf_split_sum_ggx[i*2 + 1];
		texels[i][2] = ltc_mag_ggx[i];
	}

	tex = DRW_texture_create_2D(64, 64, DRW_TEX_RGB_16, DRW_TEX_FILTER, (float *)texels);
	MEM_freeN(texels);
#endif

	return tex;
}

```


*probe_vert.glsl*
```glsl
in vec3 pos;

out vec4 vPos;
out int face;

void main() {
	vPos = vec4(pos, 1.0);
	face = gl_InstanceID;
}

```

*probe_geom.glsl*
```glsl

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

uniform int Layer;

in vec4 vPos[];
in int face[];

out vec3 worldPosition;
out vec3 worldNormal;

const vec3 maj_axes[6] = vec3[6](vec3(1.0,  0.0,  0.0), vec3(-1.0,  0.0, 0.0), vec3(0.0, 1.0, 0.0), vec3(0.0, -1.0,  0.0), vec3( 0.0,  0.0, 1.0), vec3( 0.0,  0.0, -1.0));
const vec3 x_axis[6]   = vec3[6](vec3(0.0,  0.0, -1.0), vec3( 0.0,  0.0, 1.0), vec3(1.0, 0.0, 0.0), vec3(1.0,  0.0,  0.0), vec3( 1.0,  0.0, 0.0), vec3(-1.0,  0.0,  0.0));
const vec3 y_axis[6]   = vec3[6](vec3(0.0, -1.0,  0.0), vec3( 0.0, -1.0, 0.0), vec3(0.0, 0.0, 1.0), vec3(0.0,  0.0, -1.0), vec3( 0.0, -1.0, 0.0), vec3( 0.0, -1.0,  0.0));

void main() {
	int f = face[0];
	gl_Layer = Layer + f;

	for (int v = 0; v < 3; ++v) {
		gl_Position = vPos[v];
		worldPosition = x_axis[f] * vPos[v].x + y_axis[f] * vPos[v].y + maj_axes[f];
		EmitVertex();
	}

	EndPrimitive();
}

```


*bsdf_lut_frag.glsl*
```glsl

out vec4 FragColor;

void main() {
	vec3 N, T, B, V;

	float NV = ( 1.0 - (clamp(gl_FragCoord.y / BRDF_LUT_SIZE, 1e-4, 0.9999)));
	float sqrtRoughness = clamp(gl_FragCoord.x / BRDF_LUT_SIZE, 1e-4, 0.9999);
	float a = sqrtRoughness * sqrtRoughness;
	float a2 = a * a;

	N = vec3(0.0, 0.0, 1.0);
	T = vec3(1.0, 0.0, 0.0);
	B = vec3(0.0, 1.0, 0.0);
	V = vec3(sqrt(1.0 - NV * NV), 0.0, NV);

	setup_noise();

	/* Integrating BRDF */
	float brdf_accum = 0.0;
	float fresnel_accum = 0.0;
	for (float i = 0; i < sampleCount; i++) {
		vec3 H = sample_ggx(i, a2, N, T, B); /* Microfacet normal */
		vec3 L = -reflect(V, H);
		float NL = L.z;

		if (NL > 0.0) {
			float NH = max(H.z, 0.0);
			float VH = max(dot(V, H), 0.0);

			float G1_v = G1_Smith_GGX(NV, a2);
			float G1_l = G1_Smith_GGX(NL, a2);
			float G_smith = 4.0 * NV * NL / (G1_v * G1_l); /* See G1_Smith_GGX for explanations. */

			float brdf = (G_smith * VH) / (NH * NV);
			float Fc = pow(1.0 - VH, 5.0);

			brdf_accum += (1.0 - Fc) * brdf;
			fresnel_accum += Fc * brdf;
		}
	}
	brdf_accum /= sampleCount;
	fresnel_accum /= sampleCount;

	FragColor = vec4(brdf_accum, fresnel_accum, 0.0, 1.0);
}
```

目前 这个texture是预计算的,传入给shader只是一张图片而已， 目前只是了解下他是思路而已。


