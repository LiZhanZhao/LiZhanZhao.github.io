---
layout:     post
title:      "blender eevee 灯光 1"
subtitle:   ""
date:       2019-12-25 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 阅读背景
阅读前请完成 [blender-编译源码](http://shaderstore.cn/2019/12/11/blender-code-source/) 。这次主要是记录自己 blender eevee 的学习过程。
<br>
这次的研究基于git checkout 到 blender的 ***2017/3/28 Diffuse Light(1 / 2) ...*** 的提交上。

## eevee 渲染过程

渲染过程跟 [上一篇](http://shaderstore.cn/2019/12/11/blender-eevee-2017-3-18-eevee-depth/) 一样 

DRW_framebuffer_clear(true, true, true, clearcol, 1.0f); 

DRW_draw_pass(psl->depth_pass);  

DRW_draw_pass(psl->depth_pass_cull);  

DRW_draw_pass(psl->pass);  

DRW_draw_pass(psl->tonemap);  



## DRW_draw_pass(psl->pass) 处理光照计算
这里主要是处理灯光，处理了SUN，POINT，SPOT 三种灯光。
看代码 lit_surface_vert.glsl, lit_surface_frag.glsl, bsdf_common_lib.glsl, bsdf_direct_lib.glsl

*bsdf_common_lib.glsl*
```glsl
#define M_PI        3.14159265358979323846  /* pi */
#define M_1_PI      0.318309886183790671538  /* 1/pi */
```

*bsdf_direct_lib.glsl*
```glsl
/* Bsdf direct light function */
/* in other word, how materials react to scene lamps */

/* Naming convention
 * N       World Normal (normalized)
 * L       Outgoing Light Vector (Surface to Light in World Space) (normalized)
 * Ldist   Distance from surface to the light
 * W       World Pos
 */

/* ------------ Diffuse ------------- */

float direct_diffuse_point(vec3 N, vec3 L, float Ldist)
{
	float bsdf = max(0.0, dot(N, L));
	bsdf /= Ldist * Ldist;
	bsdf *= M_1_PI; /* Normalize */
	return bsdf;
}
#if 0
float direct_diffuse_sphere(vec3 N, vec3 L)
{

}

float direct_diffuse_rectangle(vec3 N, vec3 L)
{

}
#endif

/* infinitly far away point source, no decay */
float direct_diffuse_sun(vec3 N, vec3 L)
{
	float bsdf = max(0.0, dot(N, L));
	bsdf *= M_1_PI; /* Normalize */
	return bsdf;
}

#if 0
float direct_diffuse_unit_disc(vec3 N, vec3 L)
{

}

/* ----------- GGx ------------ */
float direct_ggx_point(vec3 N, vec3 L)
{
	float bsdf = max(0.0, dot(N, L));
	bsdf *= M_1_PI; /* Normalize */
	return bsdf;
}

float direct_ggx_sphere(vec3 N, vec3 L)
{

}

float direct_ggx_rectangle(vec3 N, vec3 L)
{

}

float direct_ggx_disc(vec3 N, vec3 L)
{

}
#endif
```


*lit_surface_vert.glsl*
```glsl

uniform mat4 ModelViewProjectionMatrix;
uniform mat4 ModelMatrix;
uniform mat3 WorldNormalMatrix;

in vec3 pos;
in vec3 nor;

out vec3 worldPosition;
out vec3 worldNormal;

void main() {
	gl_Position = ModelViewProjectionMatrix * vec4(pos, 1.0);
	worldPosition = (ModelMatrix * vec4(pos, 1.0)).xyz;
	worldNormal = WorldNormalMatrix * nor;
}

```


*lit_surface_frag.glsl*
```glsl


uniform int light_count;

struct LightData {
	vec4 positionAndInfluence;     /* w : InfluenceRadius */
	vec4 colorAndSpec;          /* w : Spec Intensity */
	vec4 spotDataRadiusShadow;  /* x : spot size, y : spot blend */
	vec4 rightVecAndSizex;         /* xyz: Normalized up vector, w: Lamp Type */
	vec4 upVecAndSizey;      /* xyz: Normalized right vector, w: Lamp Type */
	vec4 forwardVecAndType;     /* xyz: Normalized forward vector, w: Lamp Type */
};

/* convenience aliases */
#define lampColor     colorAndSpec.rgb
#define lampSpec      colorAndSpec.a
#define lampPosition  positionAndInfluence.xyz
#define lampInfluence positionAndInfluence.w
#define lampSizeX     rightVecAndSizex.w
#define lampSizeY     upVecAndSizey.w
#define lampRight     rightVecAndSizex.xyz
#define lampUp        upVecAndSizey.xyz
#define lampForward   forwardVecAndType.xyz
#define lampType      forwardVecAndType.w
#define lampSpotSize  spotDataRadiusShadow.x
#define lampSpotBlend spotDataRadiusShadow.y

layout(std140) uniform light_block {
	LightData   lights_data[MAX_LIGHT];
};

in vec3 worldPosition;
in vec3 worldNormal;

out vec4 fragColor;

/* type */
#define POINT    0.0
#define SUN      1.0
#define SPOT     2.0
#define HEMI     3.0
#define AREA     4.0

vec3 light_diffuse(LightData ld, vec3 N, vec3 W, vec3 color) {
	vec3 light, wL, L;

	if (ld.lampType == SUN) {
		L = -ld.lampForward;
		light = color * direct_diffuse_sun(N, L) * ld.lampColor;
	}
	else {
		wL = ld.lampPosition - W;
		float dist = length(wL);
		light = color * direct_diffuse_point(N, wL / dist, dist) * ld.lampColor;
	}

	if (ld.lampType == SPOT) {
		float z = dot(ld.lampForward, wL);
		vec3 lL = wL / z;
		float x = dot(ld.lampRight, lL) / ld.lampSizeX;
		float y = dot(ld.lampUp, lL) / ld.lampSizeY;

		float ellipse = 1.0 / sqrt(1.0 + x * x + y * y);

		float spotmask = smoothstep(0.0, 1.0, (ellipse - ld.lampSpotSize) / ld.lampSpotBlend);

		light *= spotmask;
	}

	return light;
}

vec3 light_specular(LightData ld, vec3 V, vec3 N, vec3 T, vec3 B, vec3 spec, float roughness) {
	vec3 L = normalize(ld.lampPosition - worldPosition);
	vec3 light = L;

	return light;
}

void main() {
	vec3 n = normalize(worldNormal);
	vec3 diffuse = vec3(0.0);

	vec3 albedo = vec3(1.0, 1.0, 1.0);

	for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
		diffuse += light_diffuse(lights_data[i], n, worldPosition, albedo);
	}

	fragColor = vec4(diffuse,1.0);
}

```

看代码不难发现，主要是在Shader处理不同的光源进行光照计算， 主要的光照计算在 *lit_surface_frag.glsl 的 light_diffuse函数*中。

## 传入光源参数参数
看看Blender中光源的参数截图:

![](/img/Eevee/Light-1/point.png)
![](/img/Eevee/Light-1/sun.png)
![](/img/Eevee/Light-1/spot.png)
![](/img/Eevee/Light-1/spot-shape.png)

在 *eevee_lights.c* 的 *EEVEE_lights_update* 中,计算光源的传入shader的参数
```c
void EEVEE_lights_update(EEVEE_StorageList *stl)
{
	int light_ct = stl->lights_info->light_count;

	/* TODO only update if data changes */
	/* Update buffer with lamp data */
	for (int i = 0; i < light_ct; ++i) {
		EEVEE_Light *evli = stl->lights_data + i;
		Object *ob = stl->lights_ref[i];
		Lamp *la = (Lamp *)ob->data;
		float mat[4][4], scale[3];

		/* Position */
		copy_v3_v3(evli->position, ob->obmat[3]);

		/* Color */
		evli->color[0] = la->r * la->energy;
		evli->color[1] = la->g * la->energy;
		evli->color[2] = la->b * la->energy;

		/* Influence Radius */
		evli->dist = la->dist;

		/* Vectors */
		normalize_m4_m4_ex(mat, ob->obmat, scale);
		copy_v3_v3(evli->forwardvec, mat[2]);
		normalize_v3(evli->forwardvec);
		negate_v3(evli->forwardvec);

		copy_v3_v3(evli->rightvec, mat[0]);
		normalize_v3(evli->rightvec);

		copy_v3_v3(evli->upvec, mat[1]);
		normalize_v3(evli->upvec);

		/* Spot size & blend */
		if (la->type == LA_SPOT) {
			evli->sizex = scale[0] / scale[2];
			evli->sizey = scale[1] / scale[2];
			evli->spotsize = cosf(la->spotsize * 0.5f);
			evli->spotblend = (1.0f - evli->spotsize) * la->spotblend;
		}
		// else if (la->type == LA_SPOT) {

		// }

		/* Lamp Type */
		evli->lamptype = (float)la->type;
	}

	/* Upload buffer to GPU */
	DRW_uniformbuffer_update(stl->lights_ubo, stl->lights_data);
}
```
总结下  

*Point* 需要参数
- Color
- Pos  (light.transform.position)

*Sun* 需要参数
- Color
- lampForward (light.transform.forward)

*Spot* 需要
- Color
- lampForward  	(light.transform.forward)
- lampRight  	(light.transform.right)
- lampUp     	(light.transform.up)
- lampSizeX (light.transform.localScale.x / light.transform.localScale.z)
- lampSizeY (light.transform.localScale.y / light.transform.localScale.z)
- lampSpotSize  (1 - Mathf.Clamp01(light.spotSize))
- lampSpotBlend ((1.0f - lampSpotSize) * light.spotBlend)

## ToneMap
Tonemap 也优化过，*tonemap_frag.glsl*

```glsl

uniform sampler2D hdrColorBuf;

in vec4 uvcoordsvar;

out vec4 fragColor;

float linearrgb_to_srgb(float c)
{
	if (c < 0.0031308)
		return (c < 0.0) ? 0.0 : c * 12.92;
	else
		return 1.055 * pow(c, 1.0 / 2.4) - 0.055;
}

void linearrgb_to_srgb(vec4 col_from, out vec4 col_to)
{
	col_to.r = linearrgb_to_srgb(col_from.r);
	col_to.g = linearrgb_to_srgb(col_from.g);
	col_to.b = linearrgb_to_srgb(col_from.b);
	col_to.a = col_from.a;
}

void main() {
	fragColor = texture(hdrColorBuf, uvcoordsvar.st);

	linearrgb_to_srgb(fragColor, fragColor);
}
```

## 总结
这篇主要是了解到Eevee是怎么进行光照的，光源参数从外部传入，在Shader中进行计算。