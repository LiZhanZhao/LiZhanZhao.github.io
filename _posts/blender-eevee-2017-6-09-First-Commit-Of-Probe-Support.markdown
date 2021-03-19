---
layout:     post
title:      "blender eevee First commit of Probe support"
subtitle:   ""
date:       2021-1-14 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/9  * Eevee: First commit of Probe support. <br>  

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		

		

## 效果
![](/img/Eevee/FirstCommitOfProbe/res.png)
![](/img/Eevee/FirstCommitOfProbe/res-1.png)
![](/img/Eevee/FirstCommitOfProbe/res-2.png)
![](/img/Eevee/FirstCommitOfProbe/res-3.png)

## 作用 
添加 Probe 功能，这个功能其实就是动态环境图效果，就是在Probe的位置，拍摄周围附近的场景物体作为环境图，传入给物体的Shader中
![](/img/Eevee/FirstCommitOfProbe/1.png)


## 编译
- 重新生成SLN
- git 定位到 2017/6/9 17:40:47   Merge branch 'master' into blender2.8  (eevee-25-2)    
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## Shader 
*lit_surface_frag.glsl*
```
uniform sampler2DArray probeCubes;

struct ProbeData {
	vec4 position;
	vec4 shcoefs[7];
	vec4 attenuation;
};



layout(std140) uniform probe_block {
	ProbeData probes_data[MAX_PROBE];
};

...


vec3 probe_parallax_correction(vec3 W, vec3 spec_dir, ProbeData pd, inout float roughness)
{
	/* TODO */
	return spec_dir;
}

vec3 eevee_surface_lit(vec3 world_normal, vec3 albedo, vec3 f0, float roughness, float ao)
{
	float roughnessSquared = roughness * roughness;

	ShadingData sd;
	sd.N = normalize(world_normal);
	sd.V = (ProjectionMatrix[3][3] == 0.0) /* if perspective */
	            ? normalize(cameraPos - worldPosition)
	            : cameraForward;
	sd.W = worldPosition;

	vec3 radiance = vec3(0.0);
	/* Analitic Lights */
	for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
		LightData ld = lights_data[i];
		vec3 diff, spec;
		float vis;

		sd.l_vector = ld.l_position - worldPosition;

		light_visibility(ld, sd, vis);
		light_shade(ld, sd, albedo, roughnessSquared, f0, diff, spec);

		radiance += vis * (diff + spec) * ld.l_color;
	}


	/* Envmaps */
	vec3 spec_dir = get_specular_dominant_dir(sd.N, reflect(-sd.V, sd.N), roughnessSquared);
	vec2 uv = lut_coords(dot(sd.N, sd.V), roughness);
	vec2 brdf_lut = texture(utilTex, vec3(uv, 1.0)).rg;

	vec4 spec_accum = vec4(0.0);
	vec4 diff_accum = vec4(0.0);

	/* Specular probes */
	/* Start at 1 because 0 is world probe */
	for (int i = 1; i < MAX_PROBE && i < probe_count; ++i) {
		ProbeData pd = probes_data[i];

		vec3 sample_vec = probe_parallax_correction(sd.W, spec_dir, pd, roughness);
		vec4 sample = textureLod_octahedron(probeCubes, vec4(sample_vec, i), roughness * lodMax).rgba;

		float dist_attenuation = saturate(pd.p_atten_bias - pd.p_atten_scale * distance(sd.W, pd.p_position));
		float influ_spec = min(dist_attenuation, (1.0 - spec_accum.a));

		spec_accum.rgb += sample.rgb * influ_spec;
		spec_accum.a += influ_spec;
	}

	/* World probe */
	if (spec_accum.a < 1.0 || diff_accum.a < 1.0) {
		ProbeData pd = probes_data[0];

		vec3 spec = textureLod_octahedron(probeCubes, vec4(spec_dir, 0), roughness * lodMax).rgb;
		vec3 diff = spherical_harmonics(sd.N, pd.shcoefs);

		diff_accum.rgb += diff * (1.0 - diff_accum.a);
		spec_accum.rgb += spec * (1.0 - spec_accum.a);
	}

	vec3 indirect_radiance =
	        spec_accum.rgb * F_ibl(f0, brdf_lut) +
	        diff_accum.rgb * albedo;

	return radiance + indirect_radiance * ao;
}
```
>
- 目前在Shader上的用法就是，Shader传入 sampler2DArray probeCubes, Probes 数组，这个probeCubes的[0]保存的是World Probe，世界的环境图，其他[1-x] 保存的就是 Object Probe 拍摄出来的环境图， ProbeData probes_data[MAX_PROBE] 对应是probe计算需要用到的参数。
- 在 Specular probes 计算中，在Probe拍摄出来的环境图，然后在它范围内的物体计算对应的环境图高光，这里涉及到的就是 probes_data 和 probeCubes 怎么计算传进来的。





## 渲染流程

*eevee_probes.c*
```

void EEVEE_probes_refresh(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	EEVEE_ProbesInfo *pinfo = sldata->probes;
	Object *ob;
	const DRWContextState *draw_ctx = DRW_context_state_get();
	RegionView3D *rv3d = draw_ctx->rv3d;
	struct wmWindowManager *wm = CTX_wm_manager(draw_ctx->evil_C);

	/* Render world in priority */
	if (e_data.update_world) {
		render_world_probe(sldata, psl);
		e_data.update_world = false;

		if (!e_data.world_ready_to_shade) {
			e_data.world_ready_to_shade = true;
			pinfo->num_render_probe = 1;
		}

		DRW_uniformbuffer_update(sldata->probe_ubo, &sldata->probes->probe_data);

		DRW_viewport_request_redraw();
	}
	else if (true) { /* TODO if at least one probe needs refresh */

		/* Only compute probes if not navigating or in playback */
		if (((rv3d->rflag & RV3D_NAVIGATING) != 0) || ED_screen_animation_no_scrub(wm) != NULL) {
			return;
		}

		for (int i = 1; (ob = pinfo->probes_ref[i]) && (i < MAX_PROBE); i++) {
			EEVEE_ProbeEngineData *ped = EEVEE_probe_data_get(ob);

			if (ped->need_update) {
				render_one_probe(sldata, psl, i);
				ped->need_update = false;

				if (!ped->ready_to_shade) {
					pinfo->num_render_probe++;
					ped->ready_to_shade = true;
				}

				DRW_uniformbuffer_update(sldata->probe_ubo, &sldata->probes->probe_data);

				DRW_viewport_request_redraw();

				/* Only do one probe per frame */
				break;
			}
		}
	}
}
```
>
- 上面的EEVEE_probes_refresh 渲染两部分，一部分是World Probe，主要是在函数 render_world_probe 上
![](/img/Eevee/FirstCommitOfProbe/2.png)
- 一部分是 Probe，只要的函数在 render_one_probe 上
![](/img/Eevee/FirstCommitOfProbe/3.png)
- 还有 DRW_uniformbuffer_update(sldata->probe_ubo, &sldata->probes->probe_data); 这里就是更新Shader中的 ProbeData probes_data[MAX_PROBE] 数据了


### Shader 中 ProbeData probes_data[MAX_PROBE]

*bsdf_common_lib.glsl*
```
struct ProbeData {
	vec4 position;
	vec4 shcoefs[7];
	vec4 attenuation;
};
```

*eevee_private.h*
```
typedef struct EEVEE_Probe {
	float position[3], pad1;
	float shcoefs[9][3], pad2;
	float attenuation_bias, attenuation_scale, pad3[2];
} EEVEE_Probe;
```


*eevee_probes.c*
```


static void EEVEE_probes_updates(EEVEE_SceneLayerData *sldata)
{
	EEVEE_ProbesInfo *pinfo = sldata->probes;
	Object *ob;

	for (int i = 1; (ob = pinfo->probes_ref[i]) && (i < MAX_PROBE); i++) {
		Probe *probe = (Probe *)ob->data;
		EEVEE_Probe *eprobe = &pinfo->probe_data[i];

		float dist_minus_falloff = probe->distinf - (1.0f - probe->falloff) * probe->distinf;
		eprobe->attenuation_bias = probe->distinf / max_ff(1e-8f, dist_minus_falloff);
		eprobe->attenuation_scale = 1.0f / max_ff(1e-8f, dist_minus_falloff);
	}
}
```
>
- 这里就是通过外部参数
![](/img/Eevee/FirstCommitOfProbe/4.png) 来计算Shader中用到的 ProbeData


### World Probe 世界Probe

*eevee_probes.c*
```

static void filter_probe(EEVEE_Probe *eprobe, EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, int probe_idx)
{
	EEVEE_ProbesInfo *pinfo = sldata->probes;

	/* 2 - Let gpu create Mipmaps for Filtered Importance Sampling. */
	/* Bind next framebuffer to be able to gen. mips for probe_rt. */
	DRW_framebuffer_bind(sldata->probe_filter_fb);
	DRW_texture_generate_mipmaps(sldata->probe_rt);

	/* 3 - Render to probe array to the specified layer, do prefiltering. */
	/* Detach to rebind the right mipmap. */
	DRW_framebuffer_texture_detach(sldata->probe_pool);
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
		pinfo->layer = probe_idx;
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

		DRW_framebuffer_texture_attach(sldata->probe_filter_fb, sldata->probe_pool, 0, i);
		DRW_framebuffer_viewport_size(sldata->probe_filter_fb, mipsize, mipsize);
		DRW_draw_pass(psl->probe_prefilter);
		DRW_framebuffer_texture_detach(sldata->probe_pool);

		mipsize /= 2;
		CLAMP_MIN(mipsize, 1);
	}
	/* For shading, save max level of the octahedron map */
	pinfo->lodmax = (float)(maxlevel - min_lod_level) - 1.0f;

	/* 4 - Compute spherical harmonics */
	/* Tweaking parameters to balance perf. vs precision */
	pinfo->shres = 16; /* Less texture fetches & reduce branches */
	pinfo->lodfactor = 4.0f; /* Improve cache reuse */
	DRW_framebuffer_bind(sldata->probe_sh_fb);
	DRW_draw_pass(psl->probe_sh_compute);
	DRW_framebuffer_read_data(0, 0, 9, 1, 3, 0, (float *)eprobe->shcoefs);

	/* reattach to have a valid framebuffer. */
	DRW_framebuffer_texture_attach(sldata->probe_filter_fb, sldata->probe_pool, 0, 0);
}

static void render_world_probe(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl)
{
	EEVEE_ProbesInfo *pinfo = sldata->probes;
	EEVEE_Probe *eprobe = &pinfo->probe_data[0];

	/* 1 - Render to cubemap target using geometry shader. */
	/* For world probe, we don't need to clear since we render the background directly. */
	pinfo->layer = 0;

	DRW_framebuffer_bind(sldata->probe_fb);
	DRW_draw_pass(psl->probe_background);

	filter_probe(eprobe, sldata, psl, 0);
}


```
>
- render_world_probe 这个函数就比较熟悉了，一开始 DRW_draw_pass(psl->probe_background); 渲染环境图到背景中
- filter_probe(eprobe, sldata, psl, 0); 利用环境图，生成环境图的SH 和 IBL 高光图
- 这些步骤在前面的
[blender eevee Introduction of world preconvolved envmap](http://shaderstore.cn/2020/05/19/blender-eevee-2017-4-18-eevee-Introduction-of-world-preconvolved-envmap/) 和
[blender eevee Spherical Harmonic Diffuse](http://shaderstore.cn/2020/06/03/blender-eevee-2017-4-19-eevee-Spherical-Harmonic-Diffuse/) 提及过


### Object Probe 物体Probe

```
for (int i = 1; (ob = pinfo->probes_ref[i]) && (i < MAX_PROBE); i++) {
    EEVEE_ProbeEngineData *ped = EEVEE_probe_data_get(ob);

    if (ped->need_update) {
        render_one_probe(sldata, psl, i);
        ped->need_update = false;

        if (!ped->ready_to_shade) {
            pinfo->num_render_probe++;
            ped->ready_to_shade = true;
        }

        DRW_uniformbuffer_update(sldata->probe_ubo, &sldata->probes->probe_data);

        DRW_viewport_request_redraw();

        /* Only do one probe per frame */
        break;
    }
}
```

>
- 遍历所有的Object Probe，调用 render_one_probe(sldata, psl, i);



```

/* Renders the probe with index probe_idx.
 * Renders the world probe if probe_idx = -1. */
static void render_one_probe(EEVEE_SceneLayerData *sldata, EEVEE_PassList *psl, int probe_idx)
{
	EEVEE_ProbesInfo *pinfo = sldata->probes;
	EEVEE_Probe *eprobe = &pinfo->probe_data[probe_idx];
	Object *ob = pinfo->probes_ref[probe_idx];
	Probe *prb = (Probe *)ob->data;

	float winmat[4][4], posmat[4][4];

	unit_m4(posmat);

	/* Update transforms */
	copy_v3_v3(eprobe->position, ob->obmat[3]);

	/* Move to capture position */
	negate_v3_v3(posmat[3], ob->obmat[3]);

	/* 1 - Render to each cubeface individually.
	 * We do this instead of using geometry shader because a) it's faster,
	 * b) it's easier than fixing the nodetree shaders (for view dependant effects). */
	pinfo->layer = 0;
	perspective_m4(winmat, -prb->clipsta, prb->clipsta, -prb->clipsta, prb->clipsta, prb->clipsta, prb->clipend);

	/* Detach to rebind the right cubeface. */
	DRW_framebuffer_bind(sldata->probe_fb);
	DRW_framebuffer_texture_detach(sldata->probe_rt);
	DRW_framebuffer_texture_detach(sldata->probe_depth_rt);
	for (int i = 0; i < 6; ++i) {
		float viewmat[4][4], persmat[4][4];
		float viewinv[4][4], persinv[4][4];

		DRW_framebuffer_cubeface_attach(sldata->probe_fb, sldata->probe_rt, 0, i, 0);
		DRW_framebuffer_cubeface_attach(sldata->probe_fb, sldata->probe_depth_rt, 0, i, 0);
		DRW_framebuffer_viewport_size(sldata->probe_fb, PROBE_CUBE_SIZE, PROBE_CUBE_SIZE);

		DRW_framebuffer_clear(false, true, false, NULL, 1.0);

		/* Setup custom matrices */
		mul_m4_m4m4(viewmat, cubefacemat[i], posmat);
		mul_m4_m4m4(persmat, winmat, viewmat);
		invert_m4_m4(persinv, persmat);
		invert_m4_m4(viewinv, viewmat);

		DRW_viewport_matrix_override_set(persmat, DRW_MAT_PERS);
		DRW_viewport_matrix_override_set(persinv, DRW_MAT_PERSINV);
		DRW_viewport_matrix_override_set(viewmat, DRW_MAT_VIEW);
		DRW_viewport_matrix_override_set(viewinv, DRW_MAT_VIEWINV);
		DRW_viewport_matrix_override_set(winmat, DRW_MAT_WIN);

		DRW_draw_pass(psl->background_pass);

		/* Depth prepass */
		DRW_draw_pass(psl->depth_pass);
		DRW_draw_pass(psl->depth_pass_cull);

		/* Shading pass */
		DRW_draw_pass(psl->default_pass);
		DRW_draw_pass(psl->default_flat_pass);
		DRW_draw_pass(psl->material_pass);

		DRW_framebuffer_texture_detach(sldata->probe_rt);
		DRW_framebuffer_texture_detach(sldata->probe_depth_rt);
	}
	DRW_framebuffer_texture_attach(sldata->probe_fb, sldata->probe_rt, 0, 0);
	DRW_framebuffer_texture_attach(sldata->probe_fb, sldata->probe_depth_rt, 0, 0);

	DRW_viewport_matrix_override_unset(DRW_MAT_PERS);
	DRW_viewport_matrix_override_unset(DRW_MAT_PERSINV);
	DRW_viewport_matrix_override_unset(DRW_MAT_VIEW);
	DRW_viewport_matrix_override_unset(DRW_MAT_VIEWINV);
	DRW_viewport_matrix_override_unset(DRW_MAT_WIN);

	filter_probe(eprobe, sldata, psl, probe_idx);
}
```
>
- 上面的，思路就是，在Object Probe 的位置上，渲染出一个Cubemap，for循环6个面，把场景的所有物体都渲染到每一个面上，其实就是相当于一个摄像机分别在cubemap的每一个面上进行拍摄，拍出一个cubemap的每一个面，然后这个cubemap就相当于一个环境图。
- 得到这样的环境图，就进行 filter_probe， 跟World Probe的思路类似，然后就传入到Shader中进行计算。
- winmat : perspective_m4(projmat, -la->clipsta, la->clipsta, -la->clipsta, la->clipsta, la->clipsta, la->clipend) <br>
参考[OpenGL Projection Matrix](http://www.songho.ca/opengl/gl_projectionmatrix.html) 主要是x 轴 ：[-la->clipsta, la->clipsta] -> [-1,1] <br>
y 轴 ：[-la->clipsta, la->clipsta] -> [-1,1], <br>
z 轴 ：[la->clipsta, la->clipend] -> [-1,1] <br>
- cubefacemat Cube的六个面的矩阵，相当于旋转矩阵
