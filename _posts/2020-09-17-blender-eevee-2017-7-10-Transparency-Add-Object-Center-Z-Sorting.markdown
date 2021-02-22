---
layout:     post
title:      "blender eevee Transparency Add object center Z sorting."
subtitle:   ""
date:       2021-2-22 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/7/10  *   Eevee: Transparency: Add object center Z sorting .<br> 

> Better algo should take bounding box center, but it's not referenced yet in the draw call and cannot be tweaked by user.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/TransparencySorting/1.png)
![](/img/Eevee/TransparencySorting/2.png)



## 作用
透明物体渲染排序

## 编译
- 重新生成SLN
- git 定位到  2017/7/10  * Eevee: Transparency: Add object center Z sorting .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 渲染

*eevee_engine.c*
```
static void EEVEE_draw_scene(void *vedata)
{
	...
	/* Transparent */
	DRW_pass_sort_shgroup_z(psl->transparent_pass);
	DRW_draw_pass(psl->transparent_pass);
	...
}
```
>
- 在 DRW_draw_pass(psl->transparent_pass); 之前执行 DRW_pass_sort_shgroup_z(psl->transparent_pass); 进行透明物体排序


*draw_manager.c*
```
typedef struct ZSortData {
	float *axis;
	float *origin;
} ZSortData;

static int pass_shgroup_dist_sort(void *thunk, const void *a, const void *b)
{
	const DRWCall *call_a = (DRWCall *)((const DRWShadingGroup *)a)->calls.first;
	const DRWCall *call_b = (DRWCall *)((const DRWShadingGroup *)b)->calls.first;
	const ZSortData *zsortdata = (ZSortData *)thunk;

	float tmp[3];
	sub_v3_v3v3(tmp, zsortdata->origin, call_a->obmat[3]);
	const float a_sq = dot_v3v3(zsortdata->axis, tmp);
	sub_v3_v3v3(tmp, zsortdata->origin, call_b->obmat[3]);
	const float b_sq = dot_v3v3(zsortdata->axis, tmp);

	if      (a_sq < b_sq) return  1;
	else if (a_sq > b_sq) return -1;
	else                  return  0;
}

/**
 * Sort Shading groups by decreasing Z between
 * the first call object center and a given world space point.
 **/
void DRW_pass_sort_shgroup_z(DRWPass *pass)
{
	RegionView3D *rv3d = DST.draw_ctx.rv3d;

	float (*viewinv)[4];
	viewinv = (viewport_matrix_override.override[DRW_MAT_VIEWINV])
	          ? viewport_matrix_override.mat[DRW_MAT_VIEWINV] : rv3d->viewinv;

	ZSortData zsortdata = {viewinv[2], viewinv[3]};
	BLI_listbase_sort_r(&pass->shgroups, pass_shgroup_dist_sort, &zsortdata);
}
```
>
- ZSortData zsortdata = {viewinv[2], viewinv[3]}; 对应的是 struct ZSortData 的 axis 和 origin
- pass_shgroup_dist_sort 的核心就是，先算出a,b 物体在Camera 的 Z 轴的 距离，然后进行排序，一般都是 距离远的先渲染