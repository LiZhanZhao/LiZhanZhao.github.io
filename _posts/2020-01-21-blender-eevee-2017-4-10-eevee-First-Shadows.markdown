---
layout:     post
title:      "blender eevee Shadow 1"
subtitle:   ""
date:       2020-01-21 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## Unity3D 复现效果
*Shadow*
![](/img/Eevee/Shadow-1/Sun.gif)
![](/img/Eevee/Shadow-1/Point.gif)

## 理论
看了Blender的阴影的做法，其实主要就是 Shadow Mapping 和 Point Shadows
理解可以参考  
<br>
[Shadow Mapping](https://learnopengl-cn.github.io/05%20Advanced%20Lighting/03%20Shadows/01%20Shadow%20Mapping/)
<br>
[Point Shadows](https://learnopengl-cn.github.io/05%20Advanced%20Lighting/03%20Shadows/02%20Point%20Shadows/)
<br>
todo...

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

	/* Refresh shadows */
	EEVEE_draw_shadows((EEVEE_Data *)vedata);

	/* Attach depth to the hdr buffer and bind it */	
	DRW_framebuffer_texture_detach(dtxl->depth);
	DRW_framebuffer_texture_attach(fbl->main, dtxl->depth, 0);
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
	DRW_framebuffer_texture_attach(dfbl->default_fb, dtxl->depth, 0);
	DRW_framebuffer_bind(dfbl->default_fb);

	DRW_draw_pass(psl->tonemap);
}
```
这里主要关注 *EEVEE_draw_shadows((EEVEE_Data *)vedata)* , 在这个函数中进行渲染一些实现阴影需要用到的Texture. 下面是对应的代码：


```c
/* this refresh lamps shadow buffers */
void EEVEE_draw_shadows(EEVEE_Data *vedata)
{
	EEVEE_StorageList *stl = vedata->stl;
	EEVEE_FramebufferList *fbl = vedata->fbl;
	EEVEE_LampsInfo *linfo = stl->lamps;
	Object *ob;
	int i;

	/* Cube Shadow Maps */
	/* For old hardware support, we render each face of the shadow map
	 * onto 6 layer of a big 2D texture array and sample manualy the right layer
	 * in the fragment shader. */
	DRW_framebuffer_bind(fbl->shadow_cube_fb);
	DRW_framebuffer_clear(false, true, false, NULL, 1.0);

	/* Render each shadow to one layer of the array */
	for (i = 0; (ob = linfo->shadow_cube_ref[i]) && (i < MAX_SHADOW_CUBE); i++) {
		EEVEE_LampEngineData *led = (EEVEE_LampEngineData *)DRW_lamp_engine_data_get(ob, &viewport_eevee_type);
		EEVEE_ShadowCubeData *evscd = (EEVEE_ShadowCubeData *)led->sto;

		for (int j = 0; j < 6; ++j) {
			linfo->layer = i * 6 + j;
			copy_m4_m4(linfo->shadowmat, evscd->viewprojmat[j]);
			DRW_draw_pass(vedata->psl->shadow_pass);
		}
	}

	/* Standard Shadow Maps */
	DRW_framebuffer_bind(fbl->shadow_map_fb);
	DRW_framebuffer_clear(false, true, false, NULL, 1.0);

	/* Render each shadow to one layer of the array */
	for (i = 0; (ob = linfo->shadow_map_ref[i]) && (i < MAX_SHADOW_MAP); i++) {
		EEVEE_LampEngineData *led = (EEVEE_LampEngineData *)DRW_lamp_engine_data_get(ob, &viewport_eevee_type);
		EEVEE_ShadowMapData *evsmd = (EEVEE_ShadowMapData *)led->sto;

		linfo->layer = i;
		copy_m4_m4(linfo->shadowmat, evsmd->viewprojmat);
		DRW_draw_pass(vedata->psl->shadow_pass);
	}

	// DRW_framebuffer_bind(e_data.shadow_cascade_fb);
}
```



在渲染的时候，需要准备数据，例如 *linfo->shadowmat* 是怎么来的，看下面的代码

```c


void EEVEE_lights_update(EEVEE_StorageList *stl)
{
	EEVEE_LampsInfo *linfo = stl->lamps;
	Object *ob;
	int i;

	for (i = 0; (ob = linfo->light_ref[i]) && (i < MAX_LIGHT); i++) {
		EEVEE_LampEngineData *led = (EEVEE_LampEngineData *)DRW_lamp_engine_data_get(ob, &viewport_eevee_type);
		eevee_light_setup(ob, linfo, led);
	}

	for (i = 0; (ob = linfo->shadow_cube_ref[i]) && (i < MAX_SHADOW_CUBE); i++) {
		EEVEE_LampEngineData *led = (EEVEE_LampEngineData *)DRW_lamp_engine_data_get(ob, &viewport_eevee_type);
		eevee_shadow_cube_setup(ob, linfo, led);
	}

	for (i = 0; (ob = linfo->shadow_map_ref[i]) && (i < MAX_SHADOW_MAP); i++) {
		EEVEE_LampEngineData *led = (EEVEE_LampEngineData *)DRW_lamp_engine_data_get(ob, &viewport_eevee_type);
		eevee_shadow_map_setup(ob, linfo, led);
	}

	// for (i = 0; (ob = linfo->shadow_cascade_ref[i]) && (i < MAX_SHADOW_CASCADE); i++) {
	// 	EEVEE_LampEngineData *led = (EEVEE_LampEngineData *)DRW_lamp_engine_data_get(ob, &viewport_eevee_type);
	// 	eevee_shadow_map_setup(ob, linfo, led);
	// }

	DRW_uniformbuffer_update(stl->light_ubo, &linfo->light_data);
	DRW_uniformbuffer_update(stl->shadow_ubo, &linfo->shadow_cube_data); /* Update all data at once */
}

/* Update buffer with lamp data */
static void eevee_light_setup(Object *ob, EEVEE_LampsInfo *linfo, EEVEE_LampEngineData *led)
{
	/* TODO only update if data changes */
	EEVEE_LightData *evld = led->sto;
	EEVEE_Light *evli = linfo->light_data + evld->light_id;
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
		evli->radius = MAX2(0.001f, la->area_size);
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
		evli->radius = MAX2(0.001f, la->area_size);
	}

	/* Make illumination power constant */
	if (la->type == LA_AREA) {
		power = 1.0f / (evli->sizex * evli->sizey * 4.0f * M_PI) /* 1/(w*h*Pi) */
		        * 80.0f; /* XXX : Empirical, Fit cycles power */
	}
	else if (la->type == LA_SPOT || la->type == LA_LOCAL) {
		power = 1.0f / (4.0f * evli->radius * evli->radius * M_PI * M_PI) /* 1/(4*r²*Pi²) */
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

	/* No shadow by default */
	evli->shadowid = -1.0f;
}

static void eevee_shadow_cube_setup(Object *ob, EEVEE_LampsInfo *linfo, EEVEE_LampEngineData *led)
{
	float projmat[4][4];

	EEVEE_ShadowCubeData *evsmp = (EEVEE_ShadowCubeData *)led->sto;
	EEVEE_Light *evli = linfo->light_data + evsmp->light_id;
	EEVEE_ShadowCube *evsh = linfo->shadow_cube_data + evsmp->shadow_id;
	Lamp *la = (Lamp *)ob->data;

	perspective_m4(projmat, -la->clipsta, la->clipsta, -la->clipsta, la->clipsta, la->clipsta, la->clipend);

	for (int i = 0; i < 6; ++i) {
		float tmp[4][4];
		unit_m4(tmp);
		negate_v3_v3(tmp[3], ob->obmat[3]);
		mul_m4_m4m4(tmp, cubefacemat[i], tmp);
		mul_m4_m4m4(evsmp->viewprojmat[i], projmat, tmp);
	}

	evsh->bias = 0.05f * la->bias;
	evsh->near = la->clipsta;
	evsh->far = la->clipend;

	evli->shadowid = (float)(evsmp->shadow_id);
}

static void eevee_shadow_map_setup(Object *ob, EEVEE_LampsInfo *linfo, EEVEE_LampEngineData *led)
{
	float viewmat[4][4], projmat[4][4];

	EEVEE_ShadowMapData *evsmp = (EEVEE_ShadowMapData *)led->sto;
	EEVEE_Light *evli = linfo->light_data + evsmp->light_id;
	EEVEE_ShadowMap *evsh = linfo->shadow_map_data + evsmp->shadow_id;
	Lamp *la = (Lamp *)ob->data;

	invert_m4_m4(viewmat, ob->obmat);
	normalize_v3(viewmat[0]);
	normalize_v3(viewmat[1]);
	normalize_v3(viewmat[2]);

	float wsize = la->shadow_frustum_size;
	orthographic_m4(projmat, -wsize, wsize, -wsize, wsize, la->clipsta, la->clipend);

	mul_m4_m4m4(evsmp->viewprojmat, projmat, viewmat);
	mul_m4_m4m4(evsh->shadowmat, texcomat, evsmp->viewprojmat);

	evsh->bias = 0.005f * la->bias;

	evli->shadowid = (float)(MAX_SHADOW_CUBE + evsmp->shadow_id);
}

/**
 * Matches `glOrtho` result.
 */
void orthographic_m4(float matrix[4][4], const float left, const float right, const float bottom, const float top,
                     const float nearClip, const float farClip)
{
	float Xdelta, Ydelta, Zdelta;

	Xdelta = right - left;
	Ydelta = top - bottom;
	Zdelta = farClip - nearClip;
	if (Xdelta == 0.0f || Ydelta == 0.0f || Zdelta == 0.0f) {
		return;
	}
	unit_m4(matrix);
	matrix[0][0] = 2.0f / Xdelta;
	matrix[3][0] = -(right + left) / Xdelta;
	matrix[1][1] = 2.0f / Ydelta;
	matrix[3][1] = -(top + bottom) / Ydelta;
	matrix[2][2] = -2.0f / Zdelta; /* note: negate Z	*/
	matrix[3][2] = -(farClip + nearClip) / Zdelta;
}

/**
 * Matches `glFrustum` result.
 */
void perspective_m4(float mat[4][4], const float left, const float right, const float bottom, const float top,
                    const float nearClip, const float farClip)
{
	const float Xdelta = right - left;
	const float Ydelta = top - bottom;
	const float Zdelta = farClip - nearClip;

	if (Xdelta == 0.0f || Ydelta == 0.0f || Zdelta == 0.0f) {
		return;
	}
	mat[0][0] = nearClip * 2.0f / Xdelta;
	mat[1][1] = nearClip * 2.0f / Ydelta;
	mat[2][0] = (right + left) / Xdelta; /* note: negate Z	*/
	mat[2][1] = (top + bottom) / Ydelta;
	mat[2][2] = -(farClip + nearClip) / Zdelta;
	mat[2][3] = -1.0f;
	mat[3][2] = (-2.0f * nearClip * farClip) / Zdelta;
	mat[0][1] = mat[0][2] = mat[0][3] =
	mat[1][0] = mat[1][2] = mat[1][3] =
	mat[3][0] = mat[3][1] = mat[3][3] = 0.0f;

}

static float texcomat[4][4] = { /* From NDC to TexCo */
	{0.5, 0.0, 0.0, 0.0},
	{0.0, 0.5, 0.0, 0.0},
	{0.0, 0.0, 0.5, 0.0},
	{0.5, 0.5, 0.5, 1.0}
};


//texcomat下面的 static float cubefacemat[6][4][4]

static float x_pos[4][4] = { 
	{0.0, 0.0, -1.0, 0.0},
	 {0.0, -1.0, 0.0, 0.0},
	 {-1.0, 0.0, 0.0, 0.0},
	 {0.0, 0.0, 0.0, 1.0}
};

static float x_neg[4][4] = { 
	{0.0, 0.0, 1.0, 0.0},
	 {0.0, -1.0, 0.0, 0.0},
	 {1.0, 0.0, 0.0, 0.0},
	 {0.0, 0.0, 0.0, 1.0}
};

static float y_pos[4][4] = { 
	{1.0, 0.0, 0.0, 0.0},
	 {0.0, 0.0, 1.0, 0.0},
	 {0.0, -1.0, 0.0, 0.0},
	 {0.0, 0.0, 0.0, 1.0}
};

static float y_neg[4][4] = { 
	{1.0, 0.0, 0.0, 0.0},
	 {0.0, 0.0, -1.0, 0.0},
	 {0.0, 1.0, 0.0, 0.0},
	 {0.0, 0.0, 0.0, 1.0}
};

static float z_pos[4][4] = { 
	{1.0, 0.0, 0.0, 0.0},
	 {0.0, -1.0, 0.0, 0.0},
	 {0.0, 0.0, -1.0, 0.0},
	 {0.0, 0.0, 0.0, 1.0}
};

static float z_neg[4][4] = { 
	{-1.0, 0.0, 0.0, 0.0},
	 {0.0, -1.0, 0.0, 0.0},
	 {0.0, 0.0, 1.0, 0.0},
	 {0.0, 0.0, 0.0, 1.0}
};


```

这里注意的注意的就是，*Cube Shadow Maps*在渲染6个面的时候，是针对 +x, -x, +y, -y, +z, -z轴的，没有经过任何的旋转。  
<br> 
perspective_m4(projmat, -la->clipsta, la->clipsta, -la->clipsta, la->clipsta, la->clipsta, la->clipend) 主要是变换z，对于x,y 不执行变化。  
<br> 
orthographic_m4(projmat, -wsize, wsize, -wsize, wsize, la->clipsta, la->clipend) 也差不多的道理。


### Shader

看代码  lit_surface_vert.glsl, lit_surface_frag.glsl, bsdf_common_lib.glsl, bsdf_direct_lib.glsl， 最主要的是 *light_visibility*函数



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
uniform sampler2DArrayShadow shadowCubes;
uniform sampler2DArrayShadow shadowMaps;
// uniform sampler2DArrayShadow shadowCascades;

layout(std140) uniform light_block {
	LightData lights_data[MAX_LIGHT];
};

layout(std140) uniform shadow_block {
	ShadowCubeData    shadows_cube_data[MAX_SHADOW_CUBE];
	ShadowMapData     shadows_map_data[MAX_SHADOW_MAP];
	ShadowCascadeData shadows_cascade_data[MAX_SHADOW_CASCADE];
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
		vis *= step(0.0, -dot(sd.L, ld.l_forward));
	}
	else if (ld.l_type == AREA) {
		vis *= step(0.0, -dot(sd.L, ld.l_forward));
	}

	/* shadowing */
	if (ld.l_shadowid >= (MAX_SHADOW_MAP + MAX_SHADOW_CUBE)) {
		/* Shadow Cascade */
	}
	else if (ld.l_shadowid >= MAX_SHADOW_CUBE) {
		/* Shadow Map */
		float shid = ld.l_shadowid - MAX_SHADOW_CUBE;
		ShadowMapData smd = shadows_map_data[int(shid)];
		vec4 shpos = smd.shadowmat * vec4(sd.W, 1.0);
		shpos.z -= smd.sh_map_bias * shpos.w;
		shpos.xyz /= shpos.w;

		if (shpos.w > 0.0 && min(shpos.x, shpos.y) > 0.0 && max(shpos.x, shpos.y) < 1.0) {
			vis *= texture(shadowMaps, vec4(shpos.xy, shid, shpos.z));
		}
	}
	else {
		/* Shadow Cube */
		float shid = ld.l_shadowid;
		ShadowCubeData scd = shadows_cube_data[int(shid)];

		float face;
		vec2 uvs;
		vec3 Linv = sd.L;
		vec3 Labs = abs(Linv);
		vec3 maj_axis;

		if (max(Labs.y, Labs.z) < Labs.x) {
			if (Linv.x > 0.0) {
				face = 1.0;
				uvs = vec2(1.0, -1.0) * Linv.zy / -Linv.x;
				maj_axis = vec3(1.0, 0.0, 0.0);
			}
			else {
				face = 0.0;
				uvs = -Linv.zy / Linv.x;
				maj_axis = vec3(-1.0, 0.0, 0.0);
			}
		}
		else if (max(Labs.x, Labs.z) < Labs.y) {
			if (Linv.y > 0.0) {
				face = 2.0;
				uvs = vec2(-1.0, 1.0) * Linv.xz / Linv.y;
				maj_axis = vec3(0.0, 1.0, 0.0);
			}
			else {
				face = 3.0;
				uvs = -Linv.xz / -Linv.y;
				maj_axis = vec3(0.0, -1.0, 0.0);
			}
		}
		else {
			if (Linv.z > 0.0) {
				face = 5.0;
				uvs = Linv.xy / Linv.z;
				maj_axis = vec3(0.0, 0.0, 1.0);
			}
			else {
				face = 4.0;
				uvs = vec2(-1.0, 1.0) * Linv.xy / -Linv.z;
				maj_axis = vec3(0.0, 0.0, -1.0);
			}
		}

		uvs = uvs * 0.5 + 0.5;

		/* Depth in lightspace to compare against shadow map */
		float w = dot(maj_axis, sd.l_vector);
		w -= scd.sh_map_bias * w;
		float shdepth = buffer_depth(w, scd.sh_cube_far, scd.sh_cube_near);

		vis *= texture(shadowCubes, vec4(uvs, shid * 6.0 + face, shdepth));
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
	vec3 albedo = vec3(0.8);
	vec3 specular = mix(vec3(0.03), vec3(1.0), pow(max(0.0, 1.0 - dot(sd.N, sd.V)), 5.0));
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


*bsdf_common_lib.glsl*
```glsl

#define M_PI        3.14159265358979323846  /* pi */
#define M_1_PI      0.318309886183790671538  /* 1/pi */
#define M_1_2PI     0.159154943091895335768  /* 1/(2*pi) */
#define M_1_PI2     0.101321183642337771443  /* 1/(pi^2) */

/* ------- Structures -------- */

struct LightData {
	vec4 position_influence;      /* w : InfluenceRadius */
	vec4 color_spec;              /* w : Spec Intensity */
	vec4 spotdata_radius_shadow;  /* x : spot size, y : spot blend, z : radius, w: shadow id */
	vec4 rightvec_sizex;          /* xyz: Normalized up vector, w: area size X or spot scale X */
	vec4 upvec_sizey;             /* xyz: Normalized right vector, w: area size Y or spot scale Y */
	vec4 forwardvec_type;         /* xyz: Normalized forward vector, w: Lamp Type */
};

/* convenience aliases */
#define l_color        color_spec.rgb
#define l_spec         color_spec.a
#define l_position     position_influence.xyz
#define l_influence    position_influence.w
#define l_sizex        rightvec_sizex.w
#define l_sizey        upvec_sizey.w
#define l_right        rightvec_sizex.xyz
#define l_up           upvec_sizey.xyz
#define l_forward      forwardvec_type.xyz
#define l_type         forwardvec_type.w
#define l_spot_size    spotdata_radius_shadow.x
#define l_spot_blend   spotdata_radius_shadow.y
#define l_radius       spotdata_radius_shadow.z
#define l_shadowid     spotdata_radius_shadow.w


struct ShadowCubeData {
	vec4 near_far_bias;
};

/* convenience aliases */
#define sh_cube_near   near_far_bias.x
#define sh_cube_far    near_far_bias.y
#define sh_cube_bias   near_far_bias.z


struct ShadowMapData {
	mat4 shadowmat;
	vec4 near_far_bias;
};

/* convenience aliases */
#define sh_map_near   near_far_bias.x
#define sh_map_far    near_far_bias.y
#define sh_map_bias   near_far_bias.z

struct ShadowCascadeData {
	mat4 shadowmat[MAX_CASCADE_NUM];
	vec4 bias_count;
	float near[MAX_CASCADE_NUM];
	float far[MAX_CASCADE_NUM];
};

/* convenience aliases */
#define sh_cascade_bias   bias_count.x
#define sh_cascade_count  bias_count.y


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

float linear_depth(float z, float zf, float zn)
{
	if (gl_ProjectionMatrix[3][3] == 0.0) {
		return (zn  * zf) / (z * (zn - zf) + zf);
	}
	else {
		return (z * 2.0 - 1.0) * zf;
	}
}

float buffer_depth(float z, float zf, float zn)
{
	if (gl_ProjectionMatrix[3][3] == 0.0) {
		return (zf * (zn - z)) / (z * (zn - zf));
	}
	else {
		return (z / (zf * 2.0)) + 0.5;
	}
}

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

看代码  shadow_vert.glsl, shadow_geom.glsl, shadow_frag.glsl 主要是用于绘制Shadow Cubemap 的6个面

*shadow_vert.glsl*
```glsl

uniform mat4 ShadowMatrix;
uniform mat4 ModelMatrix;

in vec3 pos;

out vec4 vPos;

void main() {
	vPos = ModelMatrix * vec4(pos, 1.0);
}
```

*shadow_geom.glsl*
```glsl

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

uniform mat4 ShadowMatrix;
uniform int Layer;

in vec4 vPos[];

void main() {
	gl_Layer = Layer;
	gl_Position = ShadowMatrix * vPos[0];
	EmitVertex();
	gl_Layer = Layer;
	gl_Position = ShadowMatrix * vPos[1];
	EmitVertex();
	gl_Layer = Layer;
	gl_Position = ShadowMatrix * vPos[2];
	EmitVertex();
	EndPrimitive();
}
```

*shadow_frag.glsl*
```glsl

void main() {
}


```


