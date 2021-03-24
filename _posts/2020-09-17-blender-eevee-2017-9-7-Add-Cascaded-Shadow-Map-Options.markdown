---
layout:     post
title:      "blender eevee dd Cascaded Shadow Map options"
subtitle:   ""
date:       2021-3-24 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/9/7  *   Eevee :dd Cascaded Shadow Map options <br> 

> SVN : 2017/8/17  [windows] add numpy headers to python include dir, required for D2716


## 效果
![](/img/Eevee/CSM/03/1.png)

## 作用
添加 Cascaded Shadow Map 参数用来调整效果
- Max Distance
- Count
- Distribution
- Fade

这里主要是看看怎么把视锥体进行分段


## 准备
先阅读  [Cascaded Shadow Maps](http://shaderstore.cn/2020/07/02/blender-eevee-2017-4-21-eevee-Cascaded-Shadow-Maps/)
和 [Refactor Shadow System](http://shaderstore.cn/2021/03/21/blender-eevee-2017-9-1-Refactor-Shadow-System/)

<br>

### eevee_shadow_cascade_setup
*eevee_lights.c*
```

static void eevee_shadow_cascade_setup(Object *ob, EEVEE_LampsInfo *linfo, EEVEE_LampEngineData *led)
{
	Lamp *la = (Lamp *)ob->data;

	/* Camera Matrices */
	float persmat[4][4], persinv[4][4];
	float viewprojmat[4][4], projinv[4][4];
	float view_near, view_far;
	float near_v[4] = {0.0f, 0.0f, -1.0f, 1.0f};
	float far_v[4] = {0.0f, 0.0f,  1.0f, 1.0f};
	bool is_persp = DRW_viewport_is_persp_get();
	// 获得透视投影矩阵
	DRW_viewport_matrix_get(persmat, DRW_MAT_PERS);
	invert_m4_m4(persinv, persmat);
	/* FIXME : Get near / far from Draw manager? */
	// 获得 VP 矩阵
	DRW_viewport_matrix_get(viewprojmat, DRW_MAT_WIN);
	// projinv 保存的是 VP矩阵 的 逆
	invert_m4_m4(projinv, viewprojmat);

	// 这里就是变换 near_v 和 far_v ，从NDC 到 World 空间
	mul_m4_v4(projinv, near_v);
	mul_m4_v4(projinv, far_v);

	// 这里计算NDC 空间 -1和1 深度 对应到 World 空间的深度view_near 和 view_far
	// 如果是透视投影的话，就需要进行除法，这个下面会进行推导
	view_near = near_v[2];
	view_far = far_v[2]; /* TODO: Should be a shadow parameter */
	if (is_persp) {
		view_near /= near_v[3];
		view_far /= far_v[3];
	}

	/* Lamps Matrices */
	float viewmat[4][4], projmat[4][4];
	int sh_nbr = 1; /* TODO : MSM */

	// CSM 分多少段，la->cascade_count外部参数设置
	int cascade_nbr = la->cascade_count;

	EEVEE_ShadowCascadeData *sh_data = (EEVEE_ShadowCascadeData *)led->storage;
	EEVEE_Light *evli = linfo->light_data + sh_data->light_id;
	EEVEE_Shadow *ubo_data = linfo->shadow_data + sh_data->shadow_id;
	EEVEE_ShadowCascade *cascade_data = linfo->shadow_cascade_data + sh_data->cascade_id;

	/* The technique consists into splitting
	 * the view frustum into several sub-frustum
	 * that are individually receiving one shadow map */

	// 设置csm_start, csm_end，主要是由 view_near 和 view_far 决定，World空间的 near 和 far
	float csm_start, csm_end;

	if (is_persp) {
		csm_start = view_near;
		csm_end = max_ff(view_far, -la->cascade_max_dist);
		/* Avoid artifacts */
		csm_end = min_ff(view_near, csm_end);
	}
	else {
		csm_start = -view_far;
		csm_end = view_far;
	}

	/* init near/far */
	for (int c = 0; c < MAX_CASCADE_NUM; ++c) {
		cascade_data->split_start[c] = csm_end;
		cascade_data->split_end[c] = csm_end;
	}

	/* Compute split planes */
	float splits_start_ndc[MAX_CASCADE_NUM];
	float splits_end_ndc[MAX_CASCADE_NUM];

	// 计算 splits_start_ndc[0]，csm_start 的 ndc空间坐标
	{
		// csm_start 是 World 空间
		/* Nearest plane */
		float p[4] = {1.0f, 1.0f, csm_start, 1.0f};
		/* TODO: we don't need full m4 multiply here */

		// 把 p 从 World 空间 变换到 NDC 空间，透视除法
		mul_m4_v4(viewprojmat, p);			
		splits_start_ndc[0] = p[2];
		if (is_persp) {
			splits_start_ndc[0] /= p[3];
		}
	}

	// 计算 splits_start_ndc[cascade_nbr - 1], csm_end 的 ndc空间坐标
	{
		/* Farthest plane */
		float p[4] = {1.0f, 1.0f, csm_end, 1.0f};
		/* TODO: we don't need full m4 multiply here */
		mul_m4_v4(viewprojmat, p);
		splits_end_ndc[cascade_nbr - 1] = p[2];
		if (is_persp) {
			splits_end_ndc[cascade_nbr - 1] /= p[3];
		}
	}

	// 设置 cascade_data->split_start 和 cascade_data->split_end，csm_start 和 csm_end 是 world space的
	// 这里注意一下，第一段是 split_start[0] 和 split_end[0], 第二段 是 split_start[1]，split_end[1]，这里的 split_end[0] 和 split_start[1] 是相同的，
	// 其他的以此类推
	cascade_data->split_start[0] = csm_start;
	cascade_data->split_end[cascade_nbr - 1] = csm_end;

	for (int c = 1; c < cascade_nbr; ++c) {
		/* View Space */

		// 线性分割，按照 c/cascade_nbr 的比例 在 csm_start 和 csm_end 中进行 插值
		float linear_split = LERP(((float)(c) / (float)cascade_nbr), csm_start, csm_end);

		// 指数分割，
		float exp_split = csm_start * powf(csm_end / csm_start, (float)(c) / (float)cascade_nbr);

		// 设置当前开始, 
		// 如果是Camera 是透视Camera的情况
		if (is_persp) {
			cascade_data->split_start[c] = LERP(la->cascade_exponent, linear_split, exp_split);
		}
		else {
			cascade_data->split_start[c] = linear_split;
		}

		// 上一个的结束 等于 当前的开始
		cascade_data->split_end[c-1] = cascade_data->split_start[c];

		/* Add some overlap for smooth transition */
		cascade_data->split_start[c] = LERP(la->cascade_fade, cascade_data->split_end[c-1],
		                                    (c > 1) ? cascade_data->split_end[c-2] : cascade_data->split_start[0]);

		// 计算 splits_start_ndc[c] , NDC 空间
		/* NDC Space */
		{
			float p[4] = {1.0f, 1.0f, cascade_data->split_start[c], 1.0f};
			/* TODO: we don't need full m4 multiply here */
			mul_m4_v4(viewprojmat, p);
			splits_start_ndc[c] = p[2];

			if (is_persp) {
				splits_start_ndc[c] /= p[3];
			}
		}

		// 计算 splits_end_ndc[c-1] , NDC 空间
		{
			float p[4] = {1.0f, 1.0f, cascade_data->split_end[c-1], 1.0f};
			/* TODO: we don't need full m4 multiply here */
			mul_m4_v4(viewprojmat, p);
			splits_end_ndc[c-1] = p[2];

			if (is_persp) {
				splits_end_ndc[c-1] /= p[3];
			}
		}
	}

	/* Set last cascade split fade distance into the first split_start. */
	float prev_split = (cascade_nbr > 1) ? cascade_data->split_end[cascade_nbr-2] : cascade_data->split_start[0];
	cascade_data->split_start[0] = LERP(la->cascade_fade, cascade_data->split_end[cascade_nbr-1], prev_split);

	/* For each cascade */
	for (int c = 0; c < cascade_nbr; ++c) {
		/* Given 8 frustum corners */
		float corners[8][4] = {
			/* Near Cap */
			{-1.0f, -1.0f, splits_start_ndc[c], 1.0f},
			{ 1.0f, -1.0f, splits_start_ndc[c], 1.0f},
			{-1.0f,  1.0f, splits_start_ndc[c], 1.0f},
			{ 1.0f,  1.0f, splits_start_ndc[c], 1.0f},
			/* Far Cap */
			{-1.0f, -1.0f, splits_end_ndc[c], 1.0f},
			{ 1.0f, -1.0f, splits_end_ndc[c], 1.0f},
			{-1.0f,  1.0f, splits_end_ndc[c], 1.0f},
			{ 1.0f,  1.0f, splits_end_ndc[c], 1.0f}
		};

		// 1. 先把  corners 从 ndc 变换到 world space

		// 这里得透视除法会有证明
		/* Transform them into world space */
		for (int i = 0; i < 8; ++i)	{
			mul_m4_v4(persinv, corners[i]);
			mul_v3_fl(corners[i], 1.0f / corners[i][3]);
			corners[i][3] = 1.0f;
		}

		// 2. 再把  corners 从  world space 变换到 light space

		/* Project them into light space */
		invert_m4_m4(viewmat, ob->obmat);
		normalize_v3(viewmat[0]);
		normalize_v3(viewmat[1]);
		normalize_v3(viewmat[2]);

		for (int i = 0; i < 8; ++i)	{
			mul_m4_v4(viewmat, corners[i]);
		}

		// 3. 利用 corners 计算最小包围球，这个最小得包围盒是包围一小分块得视锥体，参数之后返回 包围球 得中心点和半径

		float center[3];
		frustum_min_bounding_sphere(corners, center, &(sh_data->radius[c]));

		// 这个相当于 shadow_origin = center * linfo->shadow_size / (2.0f * sh_data->radius[c])
		// shadow_origin / linfo->shadow_size = center / (2.0f * sh_data->radius[c])
		// 一个很比值关系
		/* Snap projection center to nearest texel to cancel shimmering. */
		float shadow_origin[2], shadow_texco[2];
		mul_v2_v2fl(shadow_origin, center, linfo->shadow_size / (2.0f * sh_data->radius[c])); /* Light to texture space. */

		/* Find the nearest texel. */
		shadow_texco[0] = round(shadow_origin[0]);
		shadow_texco[1] = round(shadow_origin[1]);

		// 计算 the nearest texel 和自身得 偏差，然后计算这个 偏差 映射到  light space 是多少距离
		/* Compute offset. */
		sub_v2_v2(shadow_texco, shadow_origin);
		mul_v2_fl(shadow_texco, (2.0f * sh_data->radius[c]) / linfo->shadow_size); /* Texture to light space. */

		
		/* Apply offset. */
		add_v2_v2(center, shadow_texco);

		/* Expand the projection to cover frustum range */
		orthographic_m4(projmat,
		                center[0] - sh_data->radius[c],
		                center[0] + sh_data->radius[c],
		                center[1] - sh_data->radius[c],
		                center[1] + sh_data->radius[c],
		                la->clipsta, la->clipend);

		mul_m4_m4m4(sh_data->viewprojmat[c], projmat, viewmat);
		mul_m4_m4m4(cascade_data->shadowmat[c], texcomat, sh_data->viewprojmat[c]);
	}

	ubo_data->bias = 0.05f * la->bias;
	ubo_data->near = la->clipsta;
	ubo_data->far = la->clipend;
	ubo_data->exp = (linfo->shadow_method == SHADOW_VSM) ? la->bleedbias : la->bleedexp;

	evli->shadowid = (float)(sh_data->shadow_id);
	ubo_data->shadow_start = (float)(sh_data->layer_id);
	ubo_data->data_start = (float)(sh_data->cascade_id);
	ubo_data->multi_shadow_count = (float)(sh_nbr);
}
```
>
- 实现细节都在注释里面

<br>

#### 1. NDC -> World

```
float near_v[4] = {0.0f, 0.0f, -1.0f, 1.0f};
float far_v[4] = {0.0f, 0.0f,  1.0f, 1.0f};
bool is_persp = DRW_viewport_is_persp_get();
// 获得 VP 矩阵
DRW_viewport_matrix_get(viewprojmat, DRW_MAT_WIN);
// projinv 保存的是 VP矩阵 的 逆
invert_m4_m4(projinv, viewprojmat);
// 这里就是变换 near_v 和 far_v ，从NDC 到 World 空间
mul_m4_v4(projinv, near_v);
mul_m4_v4(projinv, far_v);

// 这里计算NDC 空间 -1和1 深度 对应到 World 空间的深度view_near 和 view_far
// 如果是透视投影的话，就需要进行除法
view_near = near_v[2];
view_far = far_v[2]; /* TODO: Should be a shadow parameter */
if (is_persp) {
	view_near /= near_v[3];
	view_far /= far_v[3];
}
```
>
- projinc 存储得是 VP 矩阵逆 , 也就是 VP^-1
- 执行完 mul_m4_v4(projinv, near_v) 之后，view_near = near_v[2] / near_v[3]
- 也就是 NDC 变换到 World 都是进行透视除法，除以 w 分量

<br>

#### 2. 证明 NDC -> World 需要 除以 w 分量

- [matrix reshish](https://matrix.reshish.com/zh/) 很好的矩阵+图形网站
- [OpenGl Projection Matrix](http://www.songho.ca/opengl/gl_projectionmatrix.html)里面有OpenGL投影矩阵的推导

假设 P 矩阵 为 ![](/img/Eevee/CSM/03/2.png) <br>

那么 P 矩阵的 逆 就是 ![](/img/Eevee/CSM/03/3.png) 

<br>

View Space 中的 V(0, 0, X, 1) 要变成 NDC 中 (0, 0, 1, 1), 求 X	?			<br>

V * P = (0, 0, A * X + B, -X) => (0, 0, ( A * X + B ) / -X, 1)				<br>

( A * X + B ) / -X  =  1			<br>

X = -B / (1 + A)				<br>

那么假如变换之后 NDC 中 N(0, 0, 1, 1) 变换 到 View Space 是多少					<br>

N * P^-1 = (0, 0, -1, 1/B + A/B)										<br>

(0, 0,  -1 / (A+1)/B, 1)				<br>

(0, 0, -B / (1 + A), 1)				<br>


<br>

- 所以NDC -> World都要进行透视除法，除以 W 