---
layout:     post
title:      "blender eevee Do Shadow Cube Render In One Pass"
subtitle:   ""
date:       2020-06-12 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 效果展示
*Shadow*
![](/img/Eevee/DoShadowCubeRenderInOnePass/1.png)


## 理论
之前渲染*非方向光* 渲染6次来构造一个相对光源视角的存储好深度信息的cubemap, 现在利用GPU的Instance的特性来, 在一次渲染中构造一样的cubemap.




## 实践

### 渲染过程

跟 [上一篇](http://shaderstore.cn/2020/05/19/blender-eevee-2017-4-18-eevee-Introduction-of-world-preconvolved-envmap/) 一样，主要的函数 *EEVEE_draw_shadows*

```c

/* this refresh lamps shadow buffers */
void EEVEE_draw_shadows(EEVEE_Data *vedata)
{
	EEVEE_PassList *psl = vedata->psl;
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
		EEVEE_ShadowRender *srd = &linfo->shadow_render_data;

		srd->layer = i;
		for (int j = 0; j < 6; ++j) {
			copy_m4_m4(srd->shadowmat[j], evscd->viewprojmat[j]);
		}
		DRW_uniformbuffer_update(stl->shadow_render_ubo, &linfo->shadow_render_data);

		DRW_draw_pass(psl->shadow_cube_pass);
	}

	/* Standard Shadow Maps */
	...
}
```

psl->shadow_cube_pass 关联了  shadow_vert.glsl   shadow_geom.glsl    shadow_frag.glsl 


### Shader 
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

把 场景的物体 投影到 cubemap 的每一个面上, 计算cubemap每一个面的深度值

#### shadow_render_block 参数


```c
static DRWShadingGroup *eevee_cube_shadow_shgroup(EEVEE_PassList *psl, EEVEE_StorageList *stl, struct Batch *geom, float (*obmat)[4])
{
	DRWShadingGroup *grp = DRW_shgroup_instance_create(e_data.shadow_sh, psl->shadow_cube_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", stl->shadow_render_ubo, 0);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	for (int i = 0; i < 6; ++i)
		DRW_shgroup_dynamic_call_add_empty(grp);

	return grp;
}
```

```c
void EEVEE_draw_shadows(EEVEE_Data *vedata)
{
	...

	/* Render each shadow to one layer of the array */
	for (i = 0; (ob = linfo->shadow_cube_ref[i]) && (i < MAX_SHADOW_CUBE); i++) {
		EEVEE_LampEngineData *led = (EEVEE_LampEngineData *)DRW_lamp_engine_data_get(ob, &viewport_eevee_type);
		EEVEE_ShadowCubeData *evscd = (EEVEE_ShadowCubeData *)led->sto;
		EEVEE_ShadowRender *srd = &linfo->shadow_render_data;

		srd->layer = i;
		for (int j = 0; j < 6; ++j) {
			copy_m4_m4(srd->shadowmat[j], evscd->viewprojmat[j]);
		}
		DRW_uniformbuffer_update(stl->shadow_render_ubo, &linfo->shadow_render_data);

		DRW_draw_pass(psl->shadow_cube_pass);
	}

	...
}
```

shadow_render_block 关联到 stl->shadow_render_ubo
<br>

stl->shadow_render_ubo 关联到 linfo->shadow_render_data
<br>

copy_m4_m4(srd->shadowmat[j], evscd->viewprojmat[j]); 
修改了 linfo->shadow_render_data 的数据，所以 evscd->viewprojmat[j] 的数据会传入到shader的 shadow_render_block 中


srd->layer = i;  也是同样的道理
会直接传入到shader的shadow_render_block中


<br>

*viewprojmat的计算*
```c
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
```

