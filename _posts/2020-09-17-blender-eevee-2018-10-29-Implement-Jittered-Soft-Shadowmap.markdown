---
layout:     post
title:      "blender eevee Implement jittered soft shadowmap"
subtitle:   ""
date:       2021-4-23 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2018/10/29  *   Eevee : Implement jittered soft shadowmap. <br> 

> 
This new option is located in the shadows options in the render settings.
This approach is simple and just randomize the shadow map position (not
the lamp itself) and just let the temporal supersampling do the average of
all the shadowing. The downside is that is needs quite a large number of
samples to give smooth results and individual sample position can remain
visible.

>
Enabling this option will make the viewport refresh all shadow maps every
redraw so it has a serious performance impact.

>
This approach is not physicaly based at all and will not match cycles.


>
The sampling for point lamps (spheres) is not


> SVN : 2018/8/28  *  windows: missing alembic 1.7.8 files


<br><br>

## 作用
平滑阴影


<br><br>

## 效果

![](/img/Eevee/JitteredSoftShadowmap/01/1.png)

<br><br>


## 思路

### Cube Shadow Maps

*eevee_lights.c*

```
void EEVEE_lights_cache_finish(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	...
	/* Update Lamps UBOs. */
	EEVEE_lights_update(sldata, vedata);
	...
}
```

```
void EEVEE_lights_update(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	...
	eevee_shadow_cube_setup(ob, linfo, led, effects->taa_current_sample - 1);
	...
}
```

```

static void shadow_cube_random_position_set(
        EEVEE_Light *evli, Lamp *la,
        int sample_ofs,
        float ws_sample_pos[3])
{
	float jitter[3];

#ifndef DEBUG_SHADOW_DISTRIBUTION
	int i = sample_ofs;
#else
	for (int i = 0; i <= sample_ofs; ++i) {
#endif
		switch (la->type) {
			case LA_AREA:
				if (ELEM(la->area_shape, LA_AREA_RECT, LA_AREA_SQUARE)) {
					sample_rectangle(i, evli->rightvec, evli->upvec, evli->sizex, evli->sizey, jitter);
				}
				else {
					sample_ellipse(i, evli->rightvec, evli->upvec, evli->sizex, evli->sizey, jitter);
				}
				break;
			default:
				sample_ball(i, evli->radius, jitter);
		}
#ifdef DEBUG_SHADOW_DISTRIBUTION
		float p[3];
		add_v3_v3v3(p, jitter, ws_sample_pos);
		DRW_debug_sphere(p, 0.01f, (float[4]){1.0f, (sample_ofs == i) ? 1.0f : 0.0f, 0.0f, 1.0f});
	}
#endif
	add_v3_v3(ws_sample_pos, jitter);
}

static void eevee_shadow_cube_setup(Object *ob, EEVEE_LampsInfo *linfo, EEVEE_LampEngineData *led, int sample_ofs)
{
	...

	copy_v3_v3(cube_data->position, ob->obmat[3]);

	if (linfo->soft_shadows) {
		shadow_cube_random_position_set(evli, la, sample_ofs, cube_data->position);
	}

	...
}
```
>
- eevee_shadow_cube_setup 通过叠加 effects->taa_current_sample 来 修改了 cube_data->position，抖动位置

<br><br>

```
void EEVEE_draw_shadows(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	...

	copy_v3_v3(srd->position, cube_data->position);

	...
	for (int j = 0; j < 6; j++) {
		/* TODO optimize */
		float tmp[4][4];
		unit_m4(tmp);
		negate_v3_v3(tmp[3], srd->position);
		mul_m4_m4m4(viewmat, cubefacemat[j], tmp);
		mul_m4_m4m4(persmat, winmat, viewmat);
		invert_m4_m4(render_mats.mat[DRW_MAT_WININV], winmat);
		invert_m4_m4(render_mats.mat[DRW_MAT_VIEWINV], viewmat);
		invert_m4_m4(render_mats.mat[DRW_MAT_PERSINV], persmat);

		DRW_viewport_matrix_override_set_all(&render_mats);

		GPU_framebuffer_texture_cubeface_attach(sldata->shadow_cube_target_fb,
												sldata->shadow_cube_target, 0, j, 0);
		GPU_framebuffer_bind(sldata->shadow_cube_target_fb);
		GPU_framebuffer_clear_depth(sldata->shadow_cube_target_fb, 1.0f);
		DRW_draw_pass(psl->shadow_pass);
	}
	...
}
```
>
- 因为修改了位置，所以对应的灯光矩阵都修改了


<br><br>



### Cascaded Shadow Maps

*eevee_lights.c*

```
void EEVEE_draw_shadows(EEVEE_ViewLayerData *sldata, EEVEE_Data *vedata)
{
	...
	/* Cascaded Shadow Maps */
	...
	eevee_shadow_cascade_setup(ob, linfo, led, &saved_mats, near, far, effects->taa_current_sample - 1);
	...
}
```


```

static void shadow_cascade_random_matrix_set(float mat[4][4], float radius, int sample_ofs)
{
	float jitter[3];

#ifndef DEBUG_SHADOW_DISTRIBUTION
	int i = sample_ofs;
#else
	for (int i = 0; i <= sample_ofs; ++i) {
#endif
		sample_ellipse(i, mat[0], mat[1], radius, radius, jitter);
#ifdef DEBUG_SHADOW_DISTRIBUTION
		float p[3];
		add_v3_v3v3(p, jitter, mat[2]);
		DRW_debug_sphere(p, 0.01f, (float[4]){1.0f, (sample_ofs == i) ? 1.0f : 0.0f, 0.0f, 1.0f});
	}
#endif

	add_v3_v3(mat[2], jitter);
	orthogonalize_m4(mat, 2);
}

static void eevee_shadow_cascade_setup(
        Object *ob, EEVEE_LampsInfo *linfo, EEVEE_LampEngineData *led,
        DRWMatrixState *saved_mats, float view_near, float view_far, int sample_ofs)
{
	...
	if (linfo->soft_shadows) {
		shadow_cascade_random_matrix_set(viewmat, evli->radius, sample_ofs);
	}

	copy_m4_m4(sh_data->viewinv, viewmat);
	invert_m4(viewmat);
	...
	/* Project into lightspace */
	mul_m4_v3(viewmat, center);
	...
	mul_m4_m4m4(sh_data->viewprojmat[c], projmat, viewmat);

}
```
>
- 这里就是直接修改 viewmat 矩阵 来进行 平滑影子

