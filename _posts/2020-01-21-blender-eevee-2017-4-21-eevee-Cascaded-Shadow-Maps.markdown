---
layout:     post
title:      "blender eevee Cascaded Shadow Maps"
subtitle:   ""
date:       2020-07-02 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这两个commit

> 2017/4/20  Eevee: Start Implementation of Cascaded Shadow Maps

> 2017/4/21 * Eevee: Cascaded Shadow Maps, follow up.  
Compute coarse bounding box of split frustum. Can be improved  
Make use of 4 cascade.  
View dependant glitches are fixed.  
Optimized shader code.  


- 定位到 2017/4/21 * Eevee: Cascaded Shadow Maps, follow up. 

## 编译

- 修改 blender\build_files\cmake\platform 下的 platform_win32_msvc.cmake 

> 39行添加下面一段  
macro(find_package_wrapper)  
	if(WITH_WINDOWS_FIND_MODULES)  
		find_package(${ARGV})  
	endif()  
endmacro()  
<br> 
441 行添加下面一行  
set(ALEMBIC_FOUND 1)  
例如 :  
if(WITH_ALEMBIC)  
	set(ALEMBIC ${LIBDIR}/alembic)  
	set(ALEMBIC_INCLUDE_DIR ${ALEMBIC}/include)  
	set(ALEMBIC_INCLUDE_DIRS ${ALEMBIC_INCLUDE_DIR})  
	set(ALEMBIC_LIBPATH ${ALEMBIC}/lib)  
	set(ALEMBIC_LIBRARIES optimized alembic debug alembic_d)  
	set(ALEMBIC_FOUND 1)  -- 新加  
endif(  
  


- blender需要重新编译和生成, 命令 : *make.bat full nobuild 2015*

- 执行INSTALL

- 执行blender


## 效果展示
*CSM*
![](/img/Eevee/CSM/CSM.png)


## 理论
[Cascaded Shadow Mapping](https://developer.download.nvidia.cn/SDK/10.5/opengl/src/cascaded_shadow_maps/doc/cascaded_shadow_maps.pdf)
<br>  
[Cascaded Shadow Mapping 中文](https://zhuanlan.zhihu.com/p/53689987)




## 理解

### Shader

*lit_surface_frag.glsl*
```glsl

float light_visibility(LightData ld, ShadingData sd)
{
	float vis = 1.0;

	...

	/* shadowing */
	if (ld.l_shadowid >= (MAX_SHADOW_MAP + MAX_SHADOW_CUBE)) {
		/* Shadow Cascade */
		float shid = ld.l_shadowid - (MAX_SHADOW_CUBE + MAX_SHADOW_MAP);
		ShadowCascadeData smd = shadows_cascade_data[int(shid)];

		/* Finding Cascade index */
		vec4 z = vec4(-dot(cameraPos - worldPosition, normalize(eye)));
		vec4 comp = step(z, smd.split_distances);
		float cascade = dot(comp, comp);
		mat4 shadowmat;
		float bias;

		/* Manual Unrolling of a loop for better performance.
		 * Doing fetch directly with cascade index leads to
		 * major performance impact. (0.27ms -> 10.0ms for 1 light) */
		if (cascade == 0.0) {
			shadowmat = smd.shadowmat[0];
			bias = smd.bias[0];
		}
		else if (cascade == 1.0) {
			shadowmat = smd.shadowmat[1];
			bias = smd.bias[1];
		}
		else if (cascade == 2.0) {
			shadowmat = smd.shadowmat[2];
			bias = smd.bias[2];
		}
		else {
			shadowmat = smd.shadowmat[3];
			bias = smd.bias[3];
		}

		vec4 shpos = shadowmat * vec4(sd.W, 1.0);
		shpos.z -= bias * shpos.w;
		shpos.xyz /= shpos.w;

		vis *= texture(shadowCascades, vec4(shpos.xy, shid * float(MAX_CASCADE_NUM) + cascade, shpos.z));
	}
	else if (ld.l_shadowid >= MAX_SHADOW_CUBE) {
		...
	}
	else {
		/* Shadow Cube */
		...
	}

	return vis;
}
```
> 
- 利用 smd.split_distances 把相机的视锥体被分成了4个区域， 每一个区域形成新的空间，相当于每一个区域都被一个方向光刚刚好包围住，形成4个灯光空间，每一个灯光空间都有自己的 shadowmat 和 深度图。  
- shader中计算顶点的深度来判断这个顶点处于哪一个区域，然后就可以判断使用哪一个灯光空间,然后就把顶点变化到这个灯光空间，得到UV值，采样这个灯光空间的深度图，类似于ShadowMap的思路。


### Shader 数据

#### smd.shadowmat 和 smd.bias

```c

static void frustum_min_bounding_sphere(const float corners[8][4], float r_center[3], float *r_radius)
{
#if 0 /* Simple solution but waist too much space. */
	float minvec[3], maxvec[3];

	/* compute the bounding box */
	INIT_MINMAX(minvec, maxvec);
	for (int i = 0; i < 8; ++i)	{
		minmax_v3v3_v3(minvec, maxvec, corners[i]);
	}

	/* compute the bounding sphere of this box */
	r_radius = len_v3v3(minvec, maxvec) * 0.5f;
	add_v3_v3v3(r_center, minvec, maxvec);
	mul_v3_fl(r_center, 0.5f);
#else
	/* Make the bouding sphere always centered on the front diagonal */
	add_v3_v3v3(r_center, corners[4], corners[7]);
	mul_v3_fl(r_center, 0.5f);
	*r_radius = len_v3v3(corners[0], r_center);

	/* Search the largest distance between the sphere center
	 * and the front plane corners. */
	for (int i = 0; i < 4; ++i) {
		float rad = len_v3v3(corners[4+i], r_center);
		if (rad > *r_radius) {
			*r_radius = rad;
		}
	}
#endif
}

static void eevee_shadow_cascade_setup(Object *ob, EEVEE_LampsInfo *linfo, EEVEE_LampEngineData *led)
{
	/* Camera Matrices */
	float persmat[4][4], persinv[4][4];
	float viewprojmat[4][4], projinv[4][4];
	float near, far;
	float near_v[4] = {0.0f, 0.0f, -1.0f, 1.0f};
	float far_v[4] = {0.0f, 0.0f,  1.0f, 1.0f};
	bool is_persp = DRW_viewport_is_persp_get();
	DRW_viewport_matrix_get(persmat, DRW_MAT_PERS);
	invert_m4_m4(persinv, persmat);
	/* FIXME : Get near / far from Draw manager? */
	DRW_viewport_matrix_get(viewprojmat, DRW_MAT_WIN);
	invert_m4_m4(projinv, viewprojmat);
	mul_m4_v4(projinv, near_v);
	mul_m4_v4(projinv, far_v);
	near = near_v[2];
	far = far_v[2]; /* TODO: Should be a shadow parameter */
	if (is_persp) {
		near /= near_v[3];
		far /= far_v[3];
	}

	/* Lamps Matrices */
	float viewmat[4][4], projmat[4][4];
	int cascade_ct = MAX_CASCADE_NUM;
	float shadow_res = 512.0f; /* TODO parameter */

	EEVEE_ShadowCascadeData *evscp = (EEVEE_ShadowCascadeData *)led->sto;
	EEVEE_Light *evli = linfo->light_data + evscp->light_id;
	EEVEE_ShadowCascade *evsh = linfo->shadow_cascade_data + evscp->shadow_id;
	Lamp *la = (Lamp *)ob->data;

	/* The technique consists into splitting
	 * the view frustum into several sub-frustum
	 * that are individually receiving one shadow map */

	/* init near/far */
	for (int c = 0; c < MAX_CASCADE_NUM; ++c) {
		evsh->split[c] = far;
	}

	/* Compute split planes */
	float splits_ndc[MAX_CASCADE_NUM + 1];
	splits_ndc[0] = -1.0f;
	splits_ndc[cascade_ct] = 1.0f;
	for (int c = 1; c < cascade_ct; ++c) {
		const float lambda = 0.8f; /* TODO : Parameter */

		/* View Space */
		float linear_split = LERP(((float)(c) / (float)cascade_ct), near, far);
		float exp_split = near * powf(far / near, (float)(c) / (float)cascade_ct);

		if (is_persp) {
			evsh->split[c-1] = LERP(lambda, linear_split, exp_split);
		}
		else {
			evsh->split[c-1] = linear_split;
		}

		/* NDC Space */
		float p[4] = {1.0f, 1.0f, evsh->split[c-1], 1.0f};
		mul_m4_v4(viewprojmat, p);
		splits_ndc[c] = p[2];

		if (is_persp) {
			splits_ndc[c] /= p[3];
		}
	}

	/* For each cascade */
	for (int c = 0; c < cascade_ct; ++c) {
		/* Given 8 frustrum corners */
		float corners[8][4] = {
			/* Near Cap */
			{-1.0f, -1.0f, splits_ndc[c], 1.0f},
			{ 1.0f, -1.0f, splits_ndc[c], 1.0f},
			{-1.0f,  1.0f, splits_ndc[c], 1.0f},
			{ 1.0f,  1.0f, splits_ndc[c], 1.0f},
			/* Far Cap */
			{-1.0f, -1.0f, splits_ndc[c+1], 1.0f},
			{ 1.0f, -1.0f, splits_ndc[c+1], 1.0f},
			{-1.0f,  1.0f, splits_ndc[c+1], 1.0f},
			{ 1.0f,  1.0f, splits_ndc[c+1], 1.0f}
		};

		/* Transform them into world space */
		for (int i = 0; i < 8; ++i)	{
			mul_m4_v4(persinv, corners[i]);
			mul_v3_fl(corners[i], 1.0f / corners[i][3]);
			corners[i][3] = 1.0f;
		}

		/* Project them into light space */
		invert_m4_m4(viewmat, ob->obmat);
		normalize_v3(viewmat[0]);
		normalize_v3(viewmat[1]);
		normalize_v3(viewmat[2]);

		for (int i = 0; i < 8; ++i)	{
			mul_m4_v4(viewmat, corners[i]);
		}

		float center[3], radius;
		frustum_min_bounding_sphere(corners, center, &radius);

		/* Snap projection center to nearest texel to cancel shimering. */
		float shadow_origin[2], shadow_texco[2];
		mul_v2_v2fl(shadow_origin, center, shadow_res / (2.0f * radius)); /* Light to texture space. */

		/* Find the nearest texel. */
		shadow_texco[0] = round(shadow_origin[0]);
		shadow_texco[1] = round(shadow_origin[1]);

		/* Compute offset. */
		sub_v2_v2(shadow_texco, shadow_origin);
		mul_v2_fl(shadow_texco, (2.0f * radius) / shadow_res); /* Texture to light space. */

		/* Apply offset. */
		add_v2_v2(center, shadow_texco);

		/* Expand the projection to cover frustum range */
		orthographic_m4(projmat,
		                center[0] - radius,
		                center[0] + radius,
		                center[1] - radius,
		                center[1] + radius,
		                la->clipsta, la->clipend);

		mul_m4_m4m4(evscp->viewprojmat[c], projmat, viewmat);
		mul_m4_m4m4(evsh->shadowmat[c], texcomat, evscp->viewprojmat[c]);

		/* TODO modify bias depending on the cascade radius */
		evsh->bias[c] = 0.005f * la->bias;
	}

	evli->shadowid = (float)(MAX_SHADOW_CUBE + MAX_SHADOW_MAP + evscp->shadow_id);
}
```
>
- 计算世界空间的分割平面(Z值)，把视锥体分成MAX_CASCADE_NUM份，之后把分割平面(Z值)变化到NDC空间
- 遍历每一个分块，获得每一个分块的NDC空间8个顶点，把8个顶点变换到世界空间，然后再从世界空间变换到灯光空间
- 计算这8个顶点最小包围球，获得这个球的中心和半径，注意这些顶点已经在灯光空间
- 获得这个中心点(x, y)对应在 512x512 纹理的像素坐标(四舍五入), 反过来再求 这个像素坐标 对应的灯光空间 点，(Snap projection center to nearest texel to cancel shimering.)。
- 利用 center 和 radius 来计算 正交矩阵，把x,y 从 [center - radius, center + radius] 变换到 [-1, 1]

#### shadowCascades


*eevee_cascade_shadow_shgroup*
```c
static DRWShadingGroup *eevee_cascade_shadow_shgroup(EEVEE_PassList *psl, EEVEE_StorageList *stl, struct Batch *geom, float (*obmat)[4])
{
	DRWShadingGroup *grp = DRW_shgroup_instance_create(e_data.shadow_sh, psl->shadow_cascade_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", stl->shadow_render_ubo, 0);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	for (int i = 0; i < MAX_CASCADE_NUM; ++i)
		DRW_shgroup_dynamic_call_add_empty(grp);

	return grp;
}
```

*shadow_vert.glsl*
```glsl

uniform mat4 ShadowModelMatrix;

in vec3 pos;

out vec4 vPos;
out int face;

void main() {
	vPos = ShadowModelMatrix * vec4(pos, 1.0);
	face = gl_InstanceID;
}
```

*shadow_geom.glsl*
```glsl

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

layout(std140) uniform shadow_render_block {
	mat4 ShadowMatrix[6];
	int Layer;
};

in vec4 vPos[];
in int face[];

void main() {
	int f = face[0];
	gl_Layer = Layer + f;

	for (int v = 0; v < 3; ++v) {
		gl_Position = ShadowMatrix[f] * vPos[v];
		EmitVertex();
	}

	EndPrimitive();
}
```

*shadow_frag.glsl*
```glsl


void main() {
}

```


>
- 几何着色器可以根据不同的ShadowMatrix来把顶点变换到不同的灯光空间中。
- gpu instance 渲染4次