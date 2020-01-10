---
layout:     post
title:      "blender eevee 灯光 2"
subtitle:   ""
date:       2020-01-11 12:00:00
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
这次的研究基于git checkout 到 blender的 ***2017/3/30 Diffuse Light(2 / 2) ...*** 的提交上。

## 渲染过程

渲染过程跟 [上一篇](http://shaderstore.cn/2019/12/25/blender-eevee-2017-3-28-eevee-diffuse-light-1/) 一样。


## 光照计算
这一篇更加完善地处理了灯光计算，光源包括了 : Point ,Sun, Spot, Area, Hemi  
<br>
看代码 lit_surface_vert.glsl, lit_surface_frag.glsl, bsdf_common_lib.glsl, bsdf_direct_lib.glsl

*bsdf_common_lib.glsl*
```glsl

#define M_PI        3.14159265358979323846  /* pi */
#define M_1_PI      0.318309886183790671538  /* 1/pi */

/* ------- Convenience functions --------- */

vec3 mul(mat3 m, vec3 v) { return m * v; }
mat3 mul(mat3 m1, mat3 m2) { return m1 * m2; }

float saturate(float a) { return clamp(a, 0.0, 1.0); }
vec2 saturate(vec2 a) { return vec2(saturate(a.x), saturate(a.y)); }
vec3 saturate(vec3 a) { return vec3(saturate(a.x), saturate(a.y), saturate(a.z)); }
vec4 saturate(vec4 a) { return vec4(saturate(a.x), saturate(a.y), saturate(a.z), saturate(a.w)); }

float distance_squared(vec2 a, vec2 b) { a -= b; return dot(a, a); }
float distance_squared(vec3 a, vec3 b) { a -= b; return dot(a, a); }

float hypot(float x, float y) { return sqrt(x*x + y*y); }

float inverse_distance(vec3 V) { return max( 1 / length(V), 1e-8); }

float line_plane_intersect_dist(vec3 lineorigin, vec3 linedirection, vec3 planeorigin, vec3 planenormal)
{
    return dot(planenormal, planeorigin - lineorigin) / dot(planenormal, linedirection);
}

vec3 line_plane_intersect(vec3 lineorigin, vec3 linedirection, vec3 planeorigin, vec3 planenormal)
{
	float dist = line_plane_intersect_dist(lineorigin, linedirection, planeorigin, planenormal);
	return lineorigin + linedirection * dist;
}

float rectangle_solid_angle(vec3 p0, vec3 p1, vec3 p2, vec3 p3)
{
	vec3 n0 = normalize(cross(p0, p1));
	vec3 n1 = normalize(cross(p1, p2));
	vec3 n2 = normalize(cross(p2, p3));
	vec3 n3 = normalize(cross(p3, p0));

	float g0 = acos(dot(-n0, n1));
	float g1 = acos(dot(-n1, n2));
	float g2 = acos(dot(-n2, n3));
	float g3 = acos(dot(-n3, n0));

	return max(0.0, (g0 + g1 + g2 + g3 - 2.0 * M_PI));
}


/* ------- Energy Conversion for lights ------- */
/* from Sebastien Lagarde
 * course_notes_moving_frostbite_to_pbr.pdf */

float sphere_energy(float radius)
{
	radius = max(radius, 1e-8);
	return 0.25 / (radius*radius * M_PI * M_PI) /* 1/(4*r²*Pi²) */
		* M_PI * M_PI * 10.0;  /* XXX : Empirical, Fit cycles power */
}

float rectangle_energy(float width, float height)
{
	return M_1_PI / (width*height) /* 1/(w*h*Pi) */
		* 20.0;  /* XXX : Empirical, Fit cycles power */
}

/* From UE4 paper */
void mrp_sphere(
        float radius, float dist, vec3 R, inout vec3 L,
        inout float roughness, inout float energy_conservation)
{
	L = dist * L;

	/* Sphere Light */
	roughness = max(3e-3, roughness); /* Artifacts appear with roughness below this threshold */

	/* energy preservation */
	float sphere_angle = saturate(radius / dist);
	energy_conservation *= pow(roughness / saturate(roughness + 0.5 * sphere_angle), 2.0);

	/* sphere light */
	float inter_dist = dot(L, R);
	vec3 closest_point_on_ray = inter_dist * R;
	vec3 center_to_ray = closest_point_on_ray - L;

	/* closest point on sphere */
	L = L + center_to_ray * saturate(radius * inverse_distance(center_to_ray));

	L = normalize(L);
}

void mrp_area(vec3 R, vec3 N, vec3 W, vec3 Lpos, vec3 Lx, vec3 Ly, vec3 Lz, float sizeX, float sizeY, float dist, inout vec3 L)
{
	vec3 refproj = line_plane_intersect(W, R, Lpos, Lz);
	vec3 norproj = line_plane_intersect(W, N, Lpos, Lz);

	vec2 area_half_size = vec2(sizeX, sizeY);

	/* Find the closest point to the rectangular light shape */
	vec3 refdir = refproj - Lpos;
	vec2 mrp = vec2(dot(refdir, Lx), dot(refdir, Ly));

	/* clamp to corners */
	mrp = clamp(mrp, -area_half_size, area_half_size);

	L = dist * L;
	L = L + mrp.x * Lx + mrp.y * Ly ;

	L = normalize(L);
}

/* GGX */
float D_ggx_opti(float NH, float a2)
{
	float tmp = (NH * a2 - NH) * NH + 1.0;
	return M_PI * tmp*tmp; /* Doing RCP and mul a2 at the end */
}

float G1_Smith_GGX(float NX, float a2)
{
	/* Using Brian Karis approach and refactoring by NX/NX
	 * this way the (2*NL)*(2*NV) in G = G1(V) * G1(L) gets canceled by the brdf denominator 4*NL*NV
	 * Rcp is done on the whole G later
	 * Note that this is not convenient for the transmition formula */
	return NX + sqrt( NX * (NX - NX * a2) + a2 );
	/* return 2 / (1 + sqrt(1 + a2 * (1 - NX*NX) / (NX*NX) ) ); /* Reference function */
}

float bsdf_ggx(vec3 N, vec3 L, vec3 V, float roughness)
{
	float a = roughness;
	float a2 = a * a;

	vec3 H = normalize(L + V);
	float NH = max(dot(N, H), 1e-8);
	float NL = max(dot(N, L), 1e-8);
	float NV = max(dot(N, V), 1e-8);

	float G = G1_Smith_GGX(NV, a2) * G1_Smith_GGX(NL, a2); /* Doing RCP at the end */
	float D = D_ggx_opti(NH, a2);

	/* Denominator is canceled by G1_Smith */
	/* bsdf = D * G / (4.0 * NL * NV); /* Reference function */
	return NL * a2 / (D * G); /* NL to Fit cycles Equation : line. 345 in bsdf_microfacet.h */
}

```

*bsdf_direct_lib.glsl*
```glsl
/* Bsdf direct light function */
/* in other word, how materials react to scene lamps */

/* Naming convention
 * V       View vector (normalized)
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
	bsdf *= M_PI / 2.0; /* Normalize */
	return bsdf;
}

/* From Frostbite PBR Course
 * Analitical irradiance from a sphere with correct horizon handling
 * http://www.frostbite.com/wp-content/uploads/2014/11/course_notes_moving_frostbite_to_pbr.pdf */
float direct_diffuse_sphere(vec3 N, vec3 L, float Ldist, float radius)
{
	radius = max(radius, 0.0001);
	float costheta = clamp(dot(N, L), -0.999, 0.999);
	float h = min(radius / Ldist , 0.9999);
	float h2 = h*h;
	float costheta2 = costheta * costheta;
	float bsdf;

	if (costheta2 > h2) {
		bsdf = M_PI * h2 * clamp(costheta, 0.0, 1.0);
	}
	else {
		float sintheta = sqrt(1.0 - costheta2);
		float x = sqrt(1.0 / h2 - 1.0);
		float y = -x * (costheta / sintheta);
		float sinthetasqrty = sintheta * sqrt(1.0 - y * y);
		bsdf = (costheta * acos(y) - x * sinthetasqrty) * h2 + atan(sinthetasqrty / x);
	}

	/* Energy conservation + cycle matching */
	bsdf = max(bsdf, 0.0);
	bsdf *= M_1_PI;
	bsdf *= sphere_energy(radius);

	return bsdf;
}

/* From Frostbite PBR Course
 * http://www.frostbite.com/wp-content/uploads/2014/11/course_notes_moving_frostbite_to_pbr.pdf */
float direct_diffuse_rectangle(
        vec3 W, vec3 N, vec3 L,
        float Ldist, vec3 Lx, vec3 Ly, vec3 Lz, float Lsizex, float Lsizey)
{
	vec3 lco = L * Ldist;

	/* Surface to corner vectors */
	vec3 p0 = lco + Lx * -Lsizex + Ly *  Lsizey;
	vec3 p1 = lco + Lx * -Lsizex + Ly * -Lsizey;
	vec3 p2 = lco + Lx *  Lsizex + Ly * -Lsizey;
	vec3 p3 = lco + Lx *  Lsizex + Ly *  Lsizey;

	float solidAngle = rectangle_solid_angle(p0, p1, p2, p3);

	float bsdf = solidAngle * 0.2 * (
		max(0.0, dot(normalize(p0), N)) +
		max(0.0, dot(normalize(p1), N)) +
		max(0.0, dot(normalize(p2), N)) +
		max(0.0, dot(normalize(p3), N)) +
		max(0.0, dot(L, N))
	);

	bsdf *= rectangle_energy(Lsizex * 2.0, Lsizey * 2.0);

	return bsdf;
}

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
#endif

/* ----------- GGx ------------ */
float direct_ggx_point(vec3 N, vec3 L, vec3 V, float roughness)
{
	return bsdf_ggx(N, L, V, roughness);
}

float direct_ggx_sphere(vec3 N, vec3 L, vec3 V, float Ldist, float radius, float roughness)
{
	vec3 R = reflect(V, N);

	float energy_conservation = 1.0;
	mrp_sphere(radius, Ldist, R, L, roughness, energy_conservation);
	float bsdf = bsdf_ggx(N, L, V, roughness);

	bsdf *= energy_conservation / (Ldist * Ldist);
	bsdf *= sphere_energy(radius) * max(radius * radius, 1e-16); /* radius is already inside energy_conservation */
	bsdf *= M_PI;

	return bsdf;
}

float direct_ggx_rectangle(
        vec3 W, vec3 N, vec3 L, vec3 V,
        float Ldist, vec3 Lx, vec3 Ly, vec3 Lz, float Lsizex, float Lsizey, float roughness)
{
	vec3 lco = L * Ldist;

	/* Surface to corner vectors */
	vec3 p0 = lco + Lx * -Lsizex + Ly *  Lsizey;
	vec3 p1 = lco + Lx * -Lsizex + Ly * -Lsizey;
	vec3 p2 = lco + Lx *  Lsizex + Ly * -Lsizey;
	vec3 p3 = lco + Lx *  Lsizex + Ly *  Lsizey;

	float solidAngle = rectangle_solid_angle(p0, p1, p2, p3);

	vec3 R = reflect(V, N);
	mrp_area(R, N, W, W + lco, Lx, Ly, Lz, Lsizex, Lsizey, Ldist, L);

	float bsdf = bsdf_ggx(N, L, V, roughness) * solidAngle;

	bsdf *= pow(max(0.0, dot(R, Lz)), 0.5); /* fade mrp artifacts */
	bsdf *= rectangle_energy(Lsizex * 2.0, Lsizey * 2.0);

	return bsdf;
}

#if 0
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
uniform vec3 cameraPos;
uniform vec3 eye;
uniform mat4 ProjectionMatrix;

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

vec3 light_diffuse(LightData ld, vec3 N, vec3 W, vec3 wL, vec3 L, float Ldist, vec3 color)
{
	vec3 light;

	if (ld.lampType == SUN) {
		L = -ld.lampForward;
		light = color * direct_diffuse_sun(N, L) * ld.lampColor;
	}
	else if (ld.lampType == AREA) {
		light = color * direct_diffuse_rectangle(W, N, L, Ldist,
		                                         ld.lampRight, ld.lampUp, ld.lampForward,
		                                         ld.lampSizeX, ld.lampSizeY) * ld.lampColor;
	}
	else {
		// light = color * direct_diffuse_point(N, L, Ldist) * ld.lampColor;
		light = color * direct_diffuse_sphere(N, L, Ldist, ld.lampSizeX) * ld.lampColor;
	}

	return light;
}

vec3 light_specular(
        LightData ld, vec3 V, vec3 N, vec3 W, vec3 wL,
        vec3 L, float Ldist, vec3 spec, float roughness)
{
	vec3 light;

	if (ld.lampType == SUN) {
		L = -ld.lampForward;
		light = spec * direct_ggx_point(N, L, V, roughness) * ld.lampColor;
	}
	else if (ld.lampType == AREA) {
		light = spec * direct_ggx_rectangle(W, N, L, V, Ldist, ld.lampRight, ld.lampUp, ld.lampForward,
		                                    ld.lampSizeX, ld.lampSizeY, roughness) * ld.lampColor;
	}
	else {
		light = spec * direct_ggx_sphere(N, L, V, Ldist, ld.lampSizeX, roughness) * ld.lampColor;
	}

	return light;
}

float light_visibility(
        LightData ld, vec3 V, vec3 N, vec3 W, vec3 wL, vec3 L, float Ldist)
{
	float vis = 1.0;

	if (ld.lampType == SPOT) {
		float z = dot(ld.lampForward, wL);
		vec3 lL = wL / z;
		float x = dot(ld.lampRight, lL) / ld.lampSizeX;
		float y = dot(ld.lampUp, lL) / ld.lampSizeY;

		float ellipse = 1.0 / sqrt(1.0 + x * x + y * y);

		float spotmask = smoothstep(0.0, 1.0, (ellipse - ld.lampSpotSize) / ld.lampSpotBlend);

		vis *= spotmask;
	}
	else if (ld.lampType == AREA) {
		vis *= step(0.0, -dot(L, ld.lampForward));
	}

	return vis;
}

void main()
{
	vec3 N = normalize(worldNormal);

	vec3 V;
	if (ProjectionMatrix[3][3] == 0.0) {
		V = normalize(cameraPos - worldPosition);
	}
	else {
		V = normalize(eye);
	}
	vec3 radiance = vec3(0.0);

	vec3 albedo = vec3(1.0, 1.0, 1.0);
	vec3 specular = mix(vec3(0.03), vec3(1.0), pow(max(0.0, 1.0 - dot(N,V)), 5.0));

	for (int i = 0; i < MAX_LIGHT && i < light_count; ++i) {
		vec3 wL = lights_data[i].lampPosition - worldPosition;
		float dist = length(wL);
		vec3 L = wL / dist;

		float vis = light_visibility(lights_data[i], V, N, worldPosition, wL, L, dist);
		vec3 spec = light_specular(lights_data[i], V, N, worldPosition, wL, L, dist, vec3(1.0), .2);
		vec3 diff = light_diffuse(lights_data[i], N, worldPosition, wL, L, dist, albedo);

		radiance += vis * (diff + spec);
	}

	fragColor = vec4(radiance, 1.0);
}

```

## 传入光源参数参数
看看Blender中光源的参数截图:

![](/img/Eevee/Light-1/point.png)
![](/img/Eevee/Light-1/sun.png)
![](/img/Eevee/Light-1/spot.png)
![](/img/Eevee/Light-1/spot-shape.png)
![](/img/Eevee/Light-2/hemi.png)
![](/img/Eevee/Light-2/area.png)


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
		else if (la->type == LA_AREA) {
			evli->sizex = la->area_size * scale[0] * 0.5f;
			evli->sizey = la->area_sizey * scale[1] * 0.5f;
		}
		else {
			evli->sizex = la->area_size * scale[0] * 0.5f;
		}

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

*Area* 需要
- Color
- lampSizeX (light.area_sizex * light.transform.localScale.x * 0.5f)
- lampSizeY (light.area_sizey * light.transform.localScale.y * 0.5f)


## Unity3D 实现结果
*Point*  
![](/img/Eevee/Light-2/Point-Res.png)
*Spot*  
![](/img/Eevee/Light-2/Spot-Res.png)
*Hemi*  
![](/img/Eevee/Light-2/Hemi-Res.png)
*Area*  
![](/img/Eevee/Light-2/Area-Res.png)
