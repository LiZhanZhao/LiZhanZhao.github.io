---
layout:     post
title:      "blender eevee LTC Area Light"
subtitle:   ""
date:       2020-01-15 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## Unity3D 复现效果
*Area LTC*
![](/img/Eevee/Light-LTC/LTC-1.gif)
![](/img/Eevee/Light-LTC/LTC-2.gif)

## 理论
LTC的理论基于[这里](https://eheitzresearch.wordpress.com/415-2/)。
todo...

## 实践

<br>

看代码 ltc_lib.glsl, lit_surface_vert.glsl, lit_surface_frag.glsl, bsdf_common_lib.glsl, bsdf_direct_lib.glsl

*ltc_lib.glsl*
```
/* Mainly From https://eheitzresearch.wordpress.com/415-2/ */

#define USE_LTC
#define LTC_LUT_SIZE 64

uniform sampler2D ltcMat;
uniform sampler2D ltcMag;

/* from Real-Time Area Lighting: a Journey from Research to Production
 * Stephen Hill and Eric Heitz */
float edge_integral(vec3 p1, vec3 p2)
{
#if 0
	/* more accurate replacement of acos */
	float x = dot(p1, p2);
	float y = abs(x);

	float a = 5.42031 + (3.12829 + 0.0902326 * y) * y;
	float b = 3.45068 + (4.18814 + y) * y;
	float theta_sintheta = a / b;

	if (x < 0.0) {
		theta_sintheta = (M_PI / sqrt(1.0 - x * x)) - theta_sintheta;
	}
	vec3 u = cross(p1, p2);
	return theta_sintheta * dot(u, N);
#endif
	float cos_theta = dot(p1, p2);
	cos_theta = clamp(cos_theta, -0.9999, 0.9999);

	float theta = acos(cos_theta);
	vec3 u = normalize(cross(p1, p2));
	return theta * cross(p1, p2).z / sin(theta);
}

int clip_quad_to_horizon(inout vec3 L[5])
{
	/* detect clipping config */
	int config = 0;
	if (L[0].z > 0.0) config += 1;
	if (L[1].z > 0.0) config += 2;
	if (L[2].z > 0.0) config += 4;
	if (L[3].z > 0.0) config += 8;

	/* clip */
	int n = 0;

	if (config == 0)
	{
		/* clip all */
	}
	else if (config == 1) /* V1 clip V2 V3 V4 */
	{
		n = 3;
		L[1] = -L[1].z * L[0] + L[0].z * L[1];
		L[2] = -L[3].z * L[0] + L[0].z * L[3];
	}
	else if (config == 2) /* V2 clip V1 V3 V4 */
	{
		n = 3;
		L[0] = -L[0].z * L[1] + L[1].z * L[0];
		L[2] = -L[2].z * L[1] + L[1].z * L[2];
	}
	else if (config == 3) /* V1 V2 clip V3 V4 */
	{
		n = 4;
		L[2] = -L[2].z * L[1] + L[1].z * L[2];
		L[3] = -L[3].z * L[0] + L[0].z * L[3];
	}
	else if (config == 4) /* V3 clip V1 V2 V4 */
	{
		n = 3;
		L[0] = -L[3].z * L[2] + L[2].z * L[3];
		L[1] = -L[1].z * L[2] + L[2].z * L[1];
	}
	else if (config == 5) /* V1 V3 clip V2 V4) impossible */
	{
		n = 0;
	}
	else if (config == 6) /* V2 V3 clip V1 V4 */
	{
		n = 4;
		L[0] = -L[0].z * L[1] + L[1].z * L[0];
		L[3] = -L[3].z * L[2] + L[2].z * L[3];
	}
	else if (config == 7) /* V1 V2 V3 clip V4 */
	{
		n = 5;
		L[4] = -L[3].z * L[0] + L[0].z * L[3];
		L[3] = -L[3].z * L[2] + L[2].z * L[3];
	}
	else if (config == 8) /* V4 clip V1 V2 V3 */
	{
		n = 3;
		L[0] = -L[0].z * L[3] + L[3].z * L[0];
		L[1] = -L[2].z * L[3] + L[3].z * L[2];
		L[2] =  L[3];
	}
	else if (config == 9) /* V1 V4 clip V2 V3 */
	{
		n = 4;
		L[1] = -L[1].z * L[0] + L[0].z * L[1];
		L[2] = -L[2].z * L[3] + L[3].z * L[2];
	}
	else if (config == 10) /* V2 V4 clip V1 V3) impossible */
	{
		n = 0;
	}
	else if (config == 11) /* V1 V2 V4 clip V3 */
	{
		n = 5;
		L[4] = L[3];
		L[3] = -L[2].z * L[3] + L[3].z * L[2];
		L[2] = -L[2].z * L[1] + L[1].z * L[2];
	}
	else if (config == 12) /* V3 V4 clip V1 V2 */
	{
		n = 4;
		L[1] = -L[1].z * L[2] + L[2].z * L[1];
		L[0] = -L[0].z * L[3] + L[3].z * L[0];
	}
	else if (config == 13) /* V1 V3 V4 clip V2 */
	{
		n = 5;
		L[4] = L[3];
		L[3] = L[2];
		L[2] = -L[1].z * L[2] + L[2].z * L[1];
		L[1] = -L[1].z * L[0] + L[0].z * L[1];
	}
	else if (config == 14) /* V2 V3 V4 clip V1 */
	{
		n = 5;
		L[4] = -L[0].z * L[3] + L[3].z * L[0];
		L[0] = -L[0].z * L[1] + L[1].z * L[0];
	}
	else if (config == 15) /* V1 V2 V3 V4 */
	{
		n = 4;
	}

	if (n == 3)
		L[3] = L[0];
	if (n == 4)
		L[4] = L[0];

	return n;
}

vec2 ltc_coords(float cosTheta, float roughness)
{
	float theta = acos(cosTheta);
	vec2 coords = vec2(roughness, theta/(0.5*3.14159));

	/* scale and bias coordinates, for correct filtered lookup */
	return coords * (LTC_LUT_SIZE - 1.0) / LTC_LUT_SIZE + 0.5 / LTC_LUT_SIZE;
}

mat3 ltc_matrix(vec2 coord)
{
	/* load inverse matrix */
	vec4 t = texture(ltcMat, coord);
	mat3 Minv = mat3(
		vec3(  1,   0, t.y),
		vec3(  0, t.z,   0),
		vec3(t.w,   0, t.x)
	);

	return Minv;
}

float ltc_evaluate(vec3 N, vec3 V, mat3 Minv, vec3 corners[4])
{
	/* construct orthonormal basis around N */
	vec3 T1, T2;
	T1 = normalize(V - N*dot(V, N));
	T2 = cross(N, T1);

	/* rotate area light in (T1, T2, R) basis */
	Minv = Minv * transpose(mat3(T1, T2, N));

	/* polygon (allocate 5 vertices for clipping) */
	vec3 L[5];
	L[0] = Minv * corners[0];
	L[1] = Minv * corners[1];
	L[2] = Minv * corners[2];
	L[3] = Minv * corners[3];

	int n = clip_quad_to_horizon(L);

	if (n == 0)
		return 0.0;

	/* project onto sphere */
	L[0] = normalize(L[0]);
	L[1] = normalize(L[1]);
	L[2] = normalize(L[2]);
	L[3] = normalize(L[3]);
	L[4] = normalize(L[4]);

	/* integrate */
	float sum = 0.0;

	sum += edge_integral(L[0], L[1]);
	sum += edge_integral(L[1], L[2]);
	sum += edge_integral(L[2], L[3]);
	if (n >= 4)
		sum += edge_integral(L[3], L[4]);
	if (n == 5)
		sum += edge_integral(L[4], L[0]);

	return abs(sum);
}

/* Aproximate circle with an octogone */
#define LTC_CIRCLE_RES 8
float ltc_evaluate_circle(vec3 N, vec3 V, mat3 Minv, vec3 p[LTC_CIRCLE_RES])
{
	/* construct orthonormal basis around N */
	vec3 T1, T2;
	T1 = normalize(V - N*dot(V, N));
	T2 = cross(N, T1);

	/* rotate area light in (T1, T2, R) basis */
	Minv = Minv * transpose(mat3(T1, T2, N));

	for (int i = 0; i < LTC_CIRCLE_RES; ++i) {
		p[i] = Minv * p[i];
		/* clip to horizon */
		p[i].z = max(0.0, p[i].z);
		/* project onto sphere */
		p[i] = normalize(p[i]);
	}

	/* integrate */
	float sum = 0.0;
	for (int i = 0; i < LTC_CIRCLE_RES - 1; ++i) {
		sum += edge_integral(p[i], p[i + 1]);
	}
	sum += edge_integral(p[LTC_CIRCLE_RES - 1], p[0]);

	return max(0.0, sum);
}


```


*bsdf_common_lib.glsl*
```glsl

#define M_PI        3.14159265358979323846  /* pi */
#define M_1_PI      0.318309886183790671538  /* 1/pi */
#define M_1_2PI     0.159154943091895335768  /* 1/(2*pi) */
#define M_1_PI2     0.101321183642337771443  /* 1/(pi^2) */

/* ------- Structures -------- */

struct LightData {
	vec4 position_influence;     /* w : InfluenceRadius */
	vec4 color_spec;          /* w : Spec Intensity */
	vec4 spotdata_shadow;  /* x : spot size, y : spot blend */
	vec4 rightvec_sizex;         /* xyz: Normalized up vector, w: Lamp Type */
	vec4 upvec_sizey;      /* xyz: Normalized right vector, w: Lamp Type */
	vec4 forwardvec_type;     /* xyz: Normalized forward vector, w: Lamp Type */
};

/* convenience aliases */
#define l_color        color_spec.rgb
#define l_spec         color_spec.a
#define l_position     position_influence.xyz
#define l_influence    position_influence.w
#define l_sizex        rightvec_sizex.w
#define l_radius       rightvec_sizex.w
#define l_sizey        upvec_sizey.w
#define l_right        rightvec_sizex.xyz
#define l_up           upvec_sizey.xyz
#define l_forward      forwardvec_type.xyz
#define l_type         forwardvec_type.w
#define l_spot_size    spotdata_shadow.x
#define l_spot_blend   spotdata_shadow.y

struct AreaData {
	vec3 corner[4];
	float solid_angle;
};

struct ShadingData {
	vec3 V; /* View vector */
	vec3 N; /* World Normal of the fragment */
	vec3 W; /* World Position of the fragment */
	vec3 R; /* Reflection vector */
	vec3 L; /* Current Light vector (normalized) */
	vec3 spec_dominant_dir; /* dominant direction of the specular rays */
	vec3 l_vector; /* Current Light vector */
	float l_distance; /* distance(l_position, W) */
	AreaData area_data; /* If current light is an area light */
};

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

float line_aligned_plane_intersect_dist(vec3 lineorigin, vec3 linedirection, vec3 planeorigin)
{
	/* aligned plane normal */
	vec3 L = planeorigin - lineorigin;
	float diskdist = length(L);
	vec3 planenormal = -normalize(L);
	return -diskdist / dot(planenormal, linedirection);
}

vec3 line_aligned_plane_intersect(vec3 lineorigin, vec3 linedirection, vec3 planeorigin)
{
	float dist = line_aligned_plane_intersect_dist(lineorigin, linedirection, planeorigin);
	if (dist < 0) {
		/* if intersection is behind we fake the intersection to be
		 * really far and (hopefully) not inside the radius of interest */
		dist = 1e16;
	}
	return lineorigin + linedirection * dist;
}

float rectangle_solid_angle(AreaData ad)
{
	vec3 n0 = normalize(cross(ad.corner[0], ad.corner[1]));
	vec3 n1 = normalize(cross(ad.corner[1], ad.corner[2]));
	vec3 n2 = normalize(cross(ad.corner[2], ad.corner[3]));
	vec3 n3 = normalize(cross(ad.corner[3], ad.corner[0]));

	float g0 = acos(dot(-n0, n1));
	float g1 = acos(dot(-n1, n2));
	float g2 = acos(dot(-n2, n3));
	float g3 = acos(dot(-n3, n0));

	return max(0.0, (g0 + g1 + g2 + g3 - 2.0 * M_PI));
}

vec3 get_specular_dominant_dir(vec3 N, vec3 R, float roughness)
{
	return normalize(mix(N, R, 1.0 - roughness * roughness));
}

/* From UE4 paper */
vec3 mrp_sphere(LightData ld, ShadingData sd, vec3 dir, inout float roughness, out float energy_conservation)
{
	roughness = max(3e-3, roughness); /* Artifacts appear with roughness below this threshold */

	/* energy preservation */
	float sphere_angle = saturate(ld.l_radius / sd.l_distance);
	energy_conservation = pow(roughness / saturate(roughness + 0.5 * sphere_angle), 2.0);

	/* sphere light */
	float inter_dist = dot(sd.l_vector, dir);
	vec3 closest_point_on_ray = inter_dist * dir;
	vec3 center_to_ray = closest_point_on_ray - sd.l_vector;

	/* closest point on sphere */
	vec3 closest_point_on_sphere = sd.l_vector + center_to_ray * saturate(ld.l_radius * inverse_distance(center_to_ray));

	return normalize(closest_point_on_sphere);
}

vec3 mrp_area(LightData ld, ShadingData sd, vec3 dir, inout float roughness, out float energy_conservation)
{
	roughness = max(3e-3, roughness); /* Artifacts appear with roughness below this threshold */

	/* FIXME : This needs to be fixed */
	energy_conservation = pow(roughness / saturate(roughness + 0.5 * sd.area_data.solid_angle), 2.0);

	vec3 refproj = line_plane_intersect(sd.W, dir, ld.l_position, ld.l_forward);

	/* Project the point onto the light plane */
	vec3 refdir = refproj - ld.l_position;
	vec2 mrp = vec2(dot(refdir, ld.l_right), dot(refdir, ld.l_up));

	/* clamp to light shape bounds */
	vec2 area_half_size = vec2(ld.l_sizex, ld.l_sizey);
	mrp = clamp(mrp, -area_half_size, area_half_size);

	/* go back in world space */
	vec3 closest_point_on_rectangle = sd.l_vector + mrp.x * ld.l_right + mrp.y * ld.l_up;

	float len = length(closest_point_on_rectangle);
	energy_conservation /= len * len;

	return closest_point_on_rectangle / len;
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
	return NX + sqrt(NX * (NX - NX * a2) + a2);
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

float direct_diffuse_point(LightData ld, ShadingData sd)
{
	float bsdf = max(0.0, dot(sd.N, sd.L));
	bsdf /= sd.l_distance * sd.l_distance;
	return bsdf;
}

/* From Frostbite PBR Course
 * Analitical irradiance from a sphere with correct horizon handling
 * http://www.frostbite.com/wp-content/uploads/2014/11/course_notes_moving_frostbite_to_pbr.pdf */
float direct_diffuse_sphere(LightData ld, ShadingData sd)
{
	float radius = max(ld.l_sizex, 0.0001);
	float costheta = clamp(dot(sd.N, sd.L), -0.999, 0.999);
	float h = min(ld.l_radius / sd.l_distance , 0.9999);
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

	bsdf = max(bsdf, 0.0);
	bsdf *= M_1_PI2;

	return bsdf;
}

/* From Frostbite PBR Course
 * http://www.frostbite.com/wp-content/uploads/2014/11/course_notes_moving_frostbite_to_pbr.pdf */
float direct_diffuse_rectangle(LightData ld, ShadingData sd)
{
#ifdef USE_LTC
	float bsdf = ltc_evaluate(sd.N, sd.V, mat3(1.0), sd.area_data.corner);
	bsdf *= M_1_2PI;
#else
	float bsdf = sd.area_data.solid_angle * 0.2 * (
		max(0.0, dot(normalize(sd.area_data.corner[0]), sd.N)) +
		max(0.0, dot(normalize(sd.area_data.corner[1]), sd.N)) +
		max(0.0, dot(normalize(sd.area_data.corner[2]), sd.N)) +
		max(0.0, dot(normalize(sd.area_data.corner[3]), sd.N)) +
		max(0.0, dot(sd.L, sd.N))
	);
	bsdf *= M_1_PI;
#endif
	return bsdf;
}

/* infinitly far away point source, no decay */
float direct_diffuse_sun(LightData ld, ShadingData sd)
{
	float bsdf = max(0.0, dot(sd.N, sd.L));
	bsdf *= M_1_PI; /* Normalize */
	return bsdf;
}

#if 0
float direct_diffuse_unit_disc(vec3 N, vec3 L)
{

}
#endif

/* ----------- GGx ------------ */
float direct_ggx_point(ShadingData sd, float roughness)
{
	float bsdf = bsdf_ggx(sd.N, sd.L, sd.V, roughness);
	bsdf /= sd.l_distance * sd.l_distance;
	return bsdf;
}

float direct_ggx_sphere(LightData ld, ShadingData sd, float roughness)
{
#ifdef USE_LTC
	vec3 P = line_aligned_plane_intersect(vec3(0.0), sd.spec_dominant_dir, sd.l_vector);

	vec3 Px = normalize(P - sd.l_vector) * ld.l_radius;
	vec3 Py = cross(Px, sd.L);

	float NV = max(dot(sd.N, sd.V), 1e-8);
	vec2 uv = ltc_coords(NV, sqrt(roughness));
	mat3 ltcmat = ltc_matrix(uv);

// #define HIGHEST_QUALITY
#ifdef HIGHEST_QUALITY
	vec3 Pxy1 = normalize( Px + Py) * ld.l_radius;
	vec3 Pxy2 = normalize(-Px + Py) * ld.l_radius;

	/* counter clockwise */
	vec3 points[8];
	points[0] = sd.l_vector + Px;
	points[1] = sd.l_vector - Pxy2;
	points[2] = sd.l_vector - Py;
	points[3] = sd.l_vector - Pxy1;
	points[4] = sd.l_vector - Px;
	points[5] = sd.l_vector + Pxy2;
	points[6] = sd.l_vector + Py;
	points[7] = sd.l_vector + Pxy1;
	float bsdf = ltc_evaluate_circle(sd.N, sd.V, ltcmat, points);
#else
	vec3 points[4];
	points[0] = sd.l_vector + Px;
	points[1] = sd.l_vector - Py;
	points[2] = sd.l_vector - Px;
	points[3] = sd.l_vector + Py;
	float bsdf = ltc_evaluate(sd.N, sd.V, ltcmat, points);
	/* sqrt(pi/2) difference between square and disk area */
	bsdf *= 1.25331413731;
#endif

	bsdf *= texture(ltcMag, uv).r; /* Bsdf intensity */
	bsdf *= M_1_2PI * M_1_PI;
#else
	float energy_conservation;
	vec3 L = mrp_sphere(ld, sd, sd.spec_dominant_dir, roughness, energy_conservation);
	float bsdf = bsdf_ggx(sd.N, L, sd.V, roughness);

	bsdf *= energy_conservation / (sd.l_distance * sd.l_distance);
	bsdf *= max(ld.l_radius * ld.l_radius, 1e-16); /* radius is already inside energy_conservation */
#endif
	return bsdf;
}

float direct_ggx_rectangle(LightData ld, ShadingData sd, float roughness)
{
#ifdef USE_LTC
	float NV = max(dot(sd.N, sd.V), 1e-8);
	vec2 uv = ltc_coords(NV, sqrt(roughness));
	mat3 ltcmat = ltc_matrix(uv);

	float bsdf = ltc_evaluate(sd.N, sd.V, ltcmat, sd.area_data.corner);
	bsdf *= texture(ltcMag, uv).r; /* Bsdf intensity */
	bsdf *= M_1_2PI;
#else
	float energy_conservation;
	vec3 L = mrp_area(ld, sd, sd.spec_dominant_dir, roughness, energy_conservation);
	float bsdf = bsdf_ggx(sd.N, L, sd.V, roughness);

	bsdf *= energy_conservation;
	/* fade mrp artifacts */
	bsdf *= max(0.0, dot(-sd.spec_dominant_dir, ld.l_forward));
	bsdf *= max(0.0, -dot(L, ld.l_forward));
#endif
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

float light_diffuse(LightData ld, ShadingData sd)
{
	if (ld.l_type == SUN) {
		return direct_diffuse_sun(ld, sd);
	}
	else if (ld.l_type == AREA) {
		return direct_diffuse_rectangle(ld, sd);
	}
	else {
		return direct_diffuse_sphere(ld, sd);
	}
}

float light_specular(LightData ld, ShadingData sd, float roughness)
{
	if (ld.l_type == SUN) {
		return direct_ggx_point(sd, roughness);
	}
	else if (ld.l_type == AREA) {
		return direct_ggx_rectangle(ld, sd, roughness);
	}
	else {
		// return direct_ggx_point(sd, roughness);
		return direct_ggx_sphere(ld, sd, roughness);
	}
}

float light_visibility(LightData ld, ShadingData sd)
{
	float vis = 1.0;

	if (ld.l_type == SPOT) {
		float z = dot(ld.l_forward, sd.l_vector);
		vec3 lL = sd.l_vector / z;
		float x = dot(ld.l_right, lL) / ld.l_sizex;
		float y = dot(ld.l_up, lL) / ld.l_sizey;

		float ellipse = 1.0 / sqrt(1.0 + x * x + y * y);

		float spotmask = smoothstep(0.0, 1.0, (ellipse - ld.l_spot_size) / ld.l_spot_blend);

		vis *= spotmask;
	}
	else if (ld.l_type == AREA) {
		vis *= step(0.0, -dot(sd.L, ld.l_forward));
	}

	return vis;
}

/* Calculation common to all bsdfs */
float light_common(inout LightData ld, inout ShadingData sd)
{
	float vis = 1.0;

	if (ld.l_type == SUN) {
		sd.L = -ld.l_forward;
	}
	else {
		sd.L = sd.l_vector / sd.l_distance;
	}

	if (ld.l_type == AREA) {
		sd.area_data.corner[0] = sd.l_vector + ld.l_right * -ld.l_sizex + ld.l_up *  ld.l_sizey;
		sd.area_data.corner[1] = sd.l_vector + ld.l_right * -ld.l_sizex + ld.l_up * -ld.l_sizey;
		sd.area_data.corner[2] = sd.l_vector + ld.l_right *  ld.l_sizex + ld.l_up * -ld.l_sizey;
		sd.area_data.corner[3] = sd.l_vector + ld.l_right *  ld.l_sizex + ld.l_up *  ld.l_sizey;
#ifndef USE_LTC
		sd.area_data.solid_angle = rectangle_solid_angle(sd.area_data);
#endif
	}

	return vis;
}

void main()
{
	ShadingData sd;
	sd.N = normalize(worldNormal);
	sd.V = (ProjectionMatrix[3][3] == 0.0) /* if perspective */
	            ? normalize(cameraPos - worldPosition)
	            : normalize(eye);
    sd.W = worldPosition;
    sd.R = reflect(-sd.V, sd.N);

	/* hardcoded test vars */
	vec3 albedo = vec3(0.0);
	vec3 specular = mix(vec3(1.0), vec3(1.0), pow(max(0.0, 1.0 - dot(sd.N, sd.V)), 5.0));
	float roughness = 0.1;

    sd.spec_dominant_dir = get_specular_dominant_dir(sd.N, sd.R, roughness);

	vec3 radiance = vec3(0.0);
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

	fragColor = vec4(radiance, 1.0);
}
```

## 传入光源参数参数
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
		float mat[4][4], scale[3], power;

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
			evli->sizex = MAX2(0.0001f, la->area_size * scale[0] * 0.5f);
			if (la->area_shape == LA_AREA_RECT) {
				evli->sizey = MAX2(0.0001f, la->area_sizey * scale[1] * 0.5f);
			}
			else {
				evli->sizey = evli->sizex;
			}
		}
		else {
			evli->sizex = MAX2(0.001f, la->area_size);
		}

		/* Make illumination power constant */
		if (la->type == LA_AREA) {
			power = 1.0f / (evli->sizex * evli->sizey * 4.0f * M_PI) /* 1/(w*h*Pi) */
			        * 80.0f; /* XXX : Empirical, Fit cycles power */
		}
		else if (la->type == LA_SPOT || la->type == LA_LOCAL) {
			power = 1.0f / (4.0f * evli->sizex * evli->sizex * M_PI * M_PI) /* 1/(4*r²*Pi²) */
			        * M_PI * M_PI * M_PI * 10.0; /* XXX : Empirical, Fit cycles power */

			/* for point lights (a.k.a radius == 0.0) */
			// power = M_PI * M_PI * 0.78; /* XXX : Empirical, Fit cycles power */
		}
		else {
			power = 1.0f;
		}
		mul_v3_fl(evli->color, power);

		/* Lamp Type */
		evli->lamptype = (float)la->type;
	}

	/* Upload buffer to GPU */
	DRW_uniformbuffer_update(stl->lights_ubo, stl->lights_data);
}
```
每一个光源需要的参数和[上一篇](http://shaderstore.cn/2020/01/11/blender-eevee-2017-3-30-eevee-diffuse-light-2/)一致

## 注意
### ltcMat 和 ltcMag
代码中使用到这两张图片，可以使用RenderDoc来获取。如果不想用截帧工具获得的话，可以在 eevee_lut.h 中找到这两张图片的数据，有了数据可以自己创建Texture，注意这两张图片的格式，ltcMat是R16G16B16A16, ltcMag是R16.


### BLENDER_VERSION 问题
如果 BLENDER_VERSION 的值变化了，需要执行CMakePredefinedTargets下的INSTALL。
BLENDER_VERSION 在 xxx\blender\source\blender\blenkernel\BKE_blender_version.h中，在目录 xxx\build_windows_Full_x64_vc14_Release\bin\Debug 中会有对应的版本目录(例如 2.78, 2.80), blender在启动的时候需要用到这个目录，如果没有对应的版本目录的话，就启动不了了。

### CG 和 GLSL 差别
#### 矩阵乘法
cg要用 mul 来代替 * 
> 
*glsl*
```glsl
mat3 Minv;
Minv = Minv * mat3(T1, T2, N);
```
*cg*
```glsl
mat3 Minv;
Minv = mul(Minv, mat3(T1, T2, N));
```



#### 矩阵初始化
cg 填充的是行，GLSL 是列
> 
 matrices can be initialized and constructed. Note that the values specified in a matrix constructor are consumed to fill the first row, then the second row, etc.:
 ```
 float3x3 m = float3x3(
   1.1, 1.2, 1.3, // first row (not column as in GLSL!)
   2.1, 2.2, 2.3, // second row
   3.1, 3.2, 3.3  // third row
);
float3 row0 = float3(0.0, 1.0, 0.0);
float3 row1 = float3(1.0, 0.0, 0.0);
float3 row2 = float3(0.0, 0.0, 1.0);
float3x3 n = float3x3(row0, row1, row2); // sets rows of matrix n
 ```
