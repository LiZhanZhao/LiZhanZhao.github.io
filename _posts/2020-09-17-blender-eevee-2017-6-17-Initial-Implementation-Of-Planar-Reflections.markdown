---
layout:     post
title:      "blender eevee Initial Implementation Of Planar Reflections"
subtitle:   ""
date:       2021-2-2 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/17  * Eevee : Initial implementation of planar reflections .<br> 

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/PlanarReflections/1.png)
![](/img/Eevee/PlanarReflections/2.png)


## 作用 
实时镜面反射

## 编译
- 重新生成SLN
- git 定位到  2017/6/17  * Eevee : Initial implementation of planar reflections  .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)


## 渲染前
*eevee_lightprobes.c*
```
static void EEVEE_planar_reflections_updates(EEVEE_SceneLayerData *sldata)
{
	EEVEE_LightProbesInfo *pinfo = sldata->probes;
	Object *ob;
	float mtx[4][4], normat[4][4], imat[4][4], rangemat[4][4];

	float viewmat[4][4], winmat[4][4];
	
	//float winmat[4][4];			/* GL_PROJECTION matrix */
	//float viewmat[4][4];		/* GL_MODELVIEW matrix */
	//float viewinv[4][4];		/* inverse of viewmat */
	//float persmat[4][4];		/* viewmat*winmat */
	//float persinv[4][4];		/* inverse of persmat */
	// 所以这里的 viewmat = GL_MODELVIEW
	// winmat = GL_PROJECTION
	DRW_viewport_matrix_get(viewmat, DRW_MAT_VIEW);
	DRW_viewport_matrix_get(winmat, DRW_MAT_WIN);

	zero_m4(rangemat);
	rangemat[0][0] = rangemat[1][1] = rangemat[2][2] = 0.5f;
	rangemat[3][0] = rangemat[3][1] = rangemat[3][2] = 0.5f;
	rangemat[3][3] = 1.0f;

	/* PLANAR REFLECTION */
	for (int i = 0; (ob = pinfo->probes_planar_ref[i]) && (i < MAX_PLANAR); i++) {
		LightProbe *probe = (LightProbe *)ob->data;
		EEVEE_PlanarReflection *eplanar = &pinfo->planar_data[i];
		EEVEE_LightProbeEngineData *ped = EEVEE_lightprobe_data_get(ob);

		/* Computing mtx : matrix that mirror position around object's XY plane. */
		normalize_m4_m4(normat, ob->obmat);  /* object > world */ // 这里只是把Scale变成1
		invert_m4_m4(imat, normat); /* world > object */

		float reflect[3] = {1.0f, 1.0f, -1.0f}; /* XY reflection plane */
		scale_m4_v3(imat, reflect); /* world > object > mirrored obj */ // 这里简单理解，就是mat[2] * -1 ,达到z轴翻转
		mul_m4_m4m4(mtx, normat, imat); /* world > object > mirrored obj > world */

		/* Reflect Camera Matrix. */
		mul_m4_m4m4(ped->viewmat, viewmat, mtx);	//这里的ped->viewmat就经过了Z Reflection 

		/* TODO FOV margin */
		float winmat_fov[4][4];
		copy_m4_m4(winmat_fov, winmat);		// winmat = GL_PROJECTION

		/* Apply Perspective Matrix. */
		mul_m4_m4m4(ped->persmat, winmat_fov, ped->viewmat);		// ped->persmat = viewmat*winmat 

		/* This is the matrix used to reconstruct texture coordinates.
		 * We use the original view matrix because it does not create
		 * visual artifacts if receiver is not perfectly aligned with
		 * the planar reflection probe. */
		mul_m4_m4m4(eplanar->reflectionmat, winmat_fov, viewmat); /* TODO FOV margin */	// 注意这里的viewmat是没有经过Reflect的
		/* Convert from [-1, 1] to [0, 1] (NDC to Texture coord). */
		mul_m4_m4m4(eplanar->reflectionmat, rangemat, eplanar->reflectionmat);

		/* TODO frustum check. */
		ped->need_update = true;

		/* Compute clip plane equation / normal. */
		float refpoint[3];
		copy_v3_v3(eplanar->plane_equation, ob->obmat[2]);
		normalize_v3(eplanar->plane_equation); /* plane normal */
		mul_v3_v3fl(refpoint, eplanar->plane_equation, -probe->clipsta);
		add_v3_v3(refpoint, ob->obmat[3]);
		eplanar->plane_equation[3] = -dot_v3v3(eplanar->plane_equation, refpoint);

		/* Compute XY clip planes. */
		normalize_v3_v3(eplanar->clip_vec_x, ob->obmat[0]);
		normalize_v3_v3(eplanar->clip_vec_y, ob->obmat[1]);

		float vec[3] = {0.0f, 0.0f, 0.0f};
		vec[0] = 1.0f; vec[1] = 0.0f; vec[2] = 0.0f;
		mul_m4_v3(ob->obmat, vec); /* Point on the edge */
		eplanar->clip_edge_x_pos = dot_v3v3(eplanar->clip_vec_x, vec);

		vec[0] = 0.0f; vec[1] = 1.0f; vec[2] = 0.0f;
		mul_m4_v3(ob->obmat, vec); /* Point on the edge */
		eplanar->clip_edge_y_pos = dot_v3v3(eplanar->clip_vec_y, vec);

		vec[0] = -1.0f; vec[1] = 0.0f; vec[2] = 0.0f;
		mul_m4_v3(ob->obmat, vec); /* Point on the edge */
		eplanar->clip_edge_x_neg = dot_v3v3(eplanar->clip_vec_x, vec);

		vec[0] = 0.0f; vec[1] = -1.0f; vec[2] = 0.0f;
		mul_m4_v3(ob->obmat, vec); /* Point on the edge */
		eplanar->clip_edge_y_neg = dot_v3v3(eplanar->clip_vec_y, vec);

		/* Facing factors */
		float max_angle = max_ff(1e-2f, probe->falloff) * M_PI * 0.5f;
		float min_angle = 0.0f;
		eplanar->facing_scale = 1.0f / max_ff(1e-8f, cosf(min_angle) - cosf(max_angle));
		eplanar->facing_bias = -min_ff(1.0f - 1e-8f, cosf(max_angle)) * eplanar->facing_scale;

		/* Distance factors */
		float max_dist = probe->distinf;
		float min_dist = min_ff(1.0f - 1e-8f, 1.0f - probe->falloff) * probe->distinf;
		eplanar->attenuation_scale = -1.0f / max_ff(1e-8f, max_dist - min_dist);
		eplanar->attenuation_bias = max_dist * -eplanar->attenuation_scale;
	}
}
```
>
- 由于渲染的时候需要用到一些数据，先看看一些重要的数据是怎么计算的, 留意代码的注释
- 看上面的代码可以发现，ped->viewmat 是经过Reflect, ped->persmat  = ped->viewmat * GL_PROJECTION，也是经过 Reflect的
- eplanar->reflectionmat 是没有经过 Relect的， 而是 original view matrix


## 渲染

*eevee_lightprobes.c*
```
void EEVEE_lightprobes_refresh(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata){
	...
	update_planar:

	for (int i = 0; (ob = pinfo->probes_planar_ref[i]) && (i < MAX_PLANAR); i++) {
		EEVEE_LightProbeEngineData *ped = EEVEE_lightprobe_data_get(ob);

		if (ped->need_update) {
			EEVEE_PlanarReflection *eplanar = &pinfo->planar_data[i];

			/* Temporary Remove all planar reflections (avoid lag effect). */
			int tmp_num_planar = pinfo->num_planar;
			pinfo->num_planar = 0;

			render_scene_to_planar(vedata, i, ped->viewmat, ped->persmat, eplanar->plane_equation);

			/* Restore */
			pinfo->num_planar = tmp_num_planar;

			ped->need_update = false;
		}
	}
	...
}


static void render_scene_to_planar(
        EEVEE_Data *vedata, int layer,
        float (*viewmat)[4], float (*persmat)[4],
        float clip_plane[4])
{
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_TextureList *txl = vedata->txl;
	EEVEE_PassList *psl = vedata->psl;

	float viewinv[4][4];
	float persinv[4][4];

	invert_m4_m4(viewinv, viewmat);
	invert_m4_m4(persinv, persmat);

	/* Attach depth here since it's a DRW_TEX_TEMP */
	DRW_framebuffer_texture_attach(fbl->planarref_fb, e_data.planar_depth, 0, 0);
	DRW_framebuffer_texture_layer_attach(fbl->planarref_fb, txl->planar_pool, 0, layer, 0);
	DRW_framebuffer_bind(fbl->planarref_fb);

	DRW_framebuffer_clear(false, true, false, NULL, 1.0);

	/* Avoid using the texture attached to framebuffer when rendering. */
	GPUTexture *tmp_planar_pool = txl->planar_pool;
	txl->planar_pool = NULL;
	DRW_viewport_matrix_override_set(persmat, DRW_MAT_PERS);
	DRW_viewport_matrix_override_set(persinv, DRW_MAT_PERSINV);
	DRW_viewport_matrix_override_set(viewmat, DRW_MAT_VIEW);
	DRW_viewport_matrix_override_set(viewinv, DRW_MAT_VIEWINV);

	/* Background */
	DRW_draw_pass(psl->background_pass);

	/* Since we are rendering with an inverted view matrix, we need
	 * to invert the facing for backface culling to be the same. */
	DRW_state_invert_facing();
	DRW_state_clip_planes_add(clip_plane);

	/* Depth prepass */
	DRW_draw_pass(psl->depth_pass_clip);
	DRW_draw_pass(psl->depth_pass_clip_cull);

	/* Shading pass */
	DRW_draw_pass(psl->default_pass);
	DRW_draw_pass(psl->default_flat_pass);
	DRW_draw_pass(psl->material_pass);

	DRW_state_invert_facing();
	DRW_state_clip_planes_reset();

	/* Restore */
	txl->planar_pool = tmp_planar_pool;
	DRW_viewport_matrix_override_unset(DRW_MAT_PERS);
	DRW_viewport_matrix_override_unset(DRW_MAT_PERSINV);
	DRW_viewport_matrix_override_unset(DRW_MAT_VIEW);
	DRW_viewport_matrix_override_unset(DRW_MAT_VIEWINV);

	DRW_framebuffer_texture_detach(txl->planar_pool);
	DRW_framebuffer_texture_detach(e_data.planar_depth);
}

```
>
- Planar Reflections 的核心就在 render_scene_to_planar 函数中，这个render_scene_to_planar负责渲染场景的物体到一张RT上
- 但是需要注意的是render_scene_to_planar传入的参数 ped->viewmat, ped->persmat, eplanar->plane_equation , 上一步说过
ped->viewmat和ped->persmat 是经过Reflect, 但是eplanar->reflectionmat没有经过Reflect
- 所以这里的render_scene_to_planar渲染的时候，传进去给物体的V，MP 矩阵都是带有 Z plane Reflect，就相当于物体沿着 Z Plane 进行 镜面了
- DRW_state_invert_facing(); 为什么需要这个，因为物体镜面了，在镜面需要看到物体的背面


## Shader
*lit_surface_frag.glsl*

```
float planar_attenuation(vec3 W, vec3 N, PlanarData pd)
{
	float fac;

	/* Normal Facing */
	fac = saturate(dot(pd.pl_normal, N) * pd.pl_facing_scale + pd.pl_facing_bias);

	/* Distance from plane */
	fac *= saturate(abs(dot(pd.pl_plane_eq, vec4(W, 1.0))) * pd.pl_fade_scale + pd.pl_fade_bias);

	/* Fancy fast clipping calculation */
	vec2 dist_to_clip;
	dist_to_clip.x = dot(pd.pl_clip_pos_x, W);
	dist_to_clip.y = dot(pd.pl_clip_pos_y, W);
	fac *= step(2.0, dot(step(pd.pl_clip_edges, dist_to_clip.xxyy), vec2(-1.0, 1.0).xyxy)); /* compare and add all tests */

	return fac;
}

vec3 eevee_surface_lit(vec3 world_normal, vec3 albedo, vec3 f0, float roughness, float ao)
{
	...
	/* Planar Reflections */
	for (int i = 0; i < MAX_PLANAR && i < planar_count && spec_accum.a < 0.999; ++i) {
		PlanarData pd = planars_data[i];

		float influence = planar_attenuation(sd.W, sd.N, pd);

		if (influence > 0.0) {
			float influ_spec = min(influence, (1.0 - spec_accum.a));

			vec4 refco = pd.reflectionmat * vec4(sd.W, 1.0);
			refco.xy /= refco.w;
			vec3 sample = textureLod(probePlanars, vec3(refco.xy, i), 0.0).rgb;

			spec_accum.rgb += sample * influ_spec;
			spec_accum.a += influ_spec;
		}
	}
	...
}
```
>
- planar_attenuation 简单理解就是在 镜面才进行反射，如果超过镜面就不进行反射了
- pd.reflectionmat 是没有经过镜面反射的，只是一个普通的 VP 矩阵 + rangemat矩阵 ([-1, 1] to [0, 1] (NDC to Texture coord)) 的变换
