---
layout:     post
title:      "blender eevee Move Cube Shadows To Octahedron Shadowmaps"
subtitle:   ""
date:       2020-12-21 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/5/20  * Eevee: Move cube shadows to octahedron shadowmaps.  <br> 
We render linear distance to the light in a R32 texture and store it into an octahedron projection inside a 2D texture array.  <br> 
This render the sampling function much more simpler and without edge artifacts. <br>  

> SVN : 2017/4/5  MSVC 2015 windows x64 (vc140) Alembic 1.7.1



## 作用 

个人理解就是，把Cube shadows 的做法优化掉了，在Shader不需要采样 Cubemap，而是采样一张 2D texture，这种2D texture是通过 
Cubemap 利用 *Octahedron Environment Mapping* 渲染出来的。


## 编译
- 如果之前用过命令 make.bat full nobuild 2015 生成vs工程 的话，需要重新生成。


## 效果

![](/img/Eevee/OctahedronShadowmap/1.png)

## 算法的理论基础
参考 [Octahedron Environment Maps](https://www.readkong.com/page/octahedron-environment-maps-6054207) <br>  
[Octahedron Environment Mapping 代码思路](https://www.shadertoy.com/view/MdsfzN) <br>

![](/img/Eevee/OctahedronShadowmap/2.png)
![](/img/Eevee/OctahedronShadowmap/3.png)

理解上面的 [Octahedron Environment Maps](https://www.readkong.com/page/octahedron-environment-maps-6054207) 文章讲述 : <br>
假设八面体顶点是 (r,0,0) (-r,0,0) (0,r,0) (0,-r,0) (0,0,r) (0,0,-r) <br>
只要满足 |px| + |py| + |pz| = r ，就表示 点 在 八面体 上 <br>
一个在八面体外的点 P，如果想要投影到 八面体上，那么必要的条件就是 |px| + |py| + |pz| = r <br>
因为我们最终的目的是把 一个八面体 展开 为一个 正方形 <br>
用r = 1 的 八面体进行展开可以简化问题 <br>
先就假设 r = 1, 如果 点p 的  |px| + |py| + |pz| != 1(r) 的话，就表示在 r = 1 的八面体上，<br>
所以我可以直接直接利用 公式1 ![](/img/Eevee/OctahedronShadowmap/4.png)<br> 
那么得到的 p' 就会在r = 1 的八面体上，<br> 
|p'x| = |px| / (|px| + |py| + |pz|)  <br> 
|p'y| = |py| / (|px| + |py| + |pz|)   <br> 
|p'z| = |pz| / (|px| + |py| + |pz|)   <br> 
|p'x| + |p'y| + |p'z| =  ( |px| + |py| + |pz| ) / (|px| + |py| + |pz|) = 1



## Cubemap -> 2D

```
vec2 octahedral_mapping(vec3 co)
{
    // projection onto octahedron
	co /= dot( vec3(1), abs(co) );

    // out-folding of the downward faces
    if ( co.y < 0.0 ) {
		co.xy = (1.0 - abs(co.zx)) * sign(co.xz);
    }

	// mapping to [0;1]ˆ2 texture space
	return co.xy * 0.5 + 0.5;
}
```
>
- co /= dot( vec3(1), abs(co) );   可以看上面的理论基础
- if ( co.y < 0.0 )   co.xy = (1.0 - abs(co.zx)) * sign(co.xz);  这个可以这样理解：
    ![](/img/Eevee/OctahedronShadowmap/5.png)
    如果把八面体展开，1，2，3，4 是八面体的上半部分，5，6，7，8是下半部分，
    1 和 5 的面 红色 的边是连在一起的，蓝色的边是连在一起的，所以，对于1 面来说，红色边是 y轴，但是对于 5面来说，红色边是x轴，蓝色边的道理类似。然后经过这一步之后，就可以把 点 转换为 八面体的 xy 平面上，范围是 [-1, 1]
- co.xy * 0.5 + 0.5;   范围变成 :    [-1,1] -> [0,1]


## 2D -> Cubemap 
```
vec3 octahedral_unmapping(vec2 co)
{
    co = co * 2.0 - 1.0;

    vec2 abs_co = abs(co);
    vec3 v = vec3(co, 1.0 - (abs_co.x + abs_co.y));

    if ( abs_co.x + abs_co.y > 1.0 ) {
        v.xy = (abs(co.yx) - 1.0) * -sign(co.xy);
    }

    return v;
}
```
>
-  co = co * 2.0 - 1.0;     范围变成 :    [0,1] -> [-1,1]
- vec3 v = vec3(co, 1.0 - (abs_co.x + abs_co.y));     
     ![](/img/Eevee/OctahedronShadowmap/5.png)
    如果 1.0 - (abs_co.x + abs_co.y) 为负数的话，表示正在采样面 5，6，7，8    ，因为 面1，2，3，4 的 (abs_co.x + abs_co.y) 不会大于1 <br>
    而且, abs_co.x + abs_co.y + abs_co.z = 1, 所有，就求出 abs_co.z 的值
    
- if ( abs_co.x + abs_co.y > 1.0 ) {  v.xy = (abs(co.yx) - 1.0) * -sign(co.xy); } <br>
    采样面 5，6，7，8，参考 *Cubemap -> 2D* 的思路


## Shader
先看看shader是怎么计算 Shadow Cube
*lit_surface_frag.glsl*
```
uniform sampler2DArray shadowCubes;
...
/* Shadow Cube */
float shid = ld.l_shadowid;
ShadowCubeData scd = shadows_cube_data[int(shid)];

vec3 cubevec = sd.W - ld.l_position;
float dist = length(cubevec);

/* projection onto octahedron */
cubevec /= dot( vec3(1), abs(cubevec) );

/* out-folding of the downward faces */
if ( cubevec.z < 0.0 ) {
    cubevec.xy = (1.0 - abs(cubevec.yx)) * sign(cubevec.xy);
}
vec2 texelSize = vec2(1.0 / 512.0);

/* mapping to [0;1]ˆ2 texture space */
vec2 uvs = cubevec.xy * (0.5) + 0.5;
uvs = uvs * (1.0 - 2.0 * texelSize) + 1.0 * texelSize; /* edge filtering fix */

float sh_test = step(0, texture(shadowCubes, vec3(uvs, shid)).r - dist);

vis *= sh_test;
```
>
- 上面的代码其实很大一部分都用到了 *Cubemap -> 2D* 的内容，通过 cubevec 可以计算得到 uvs，uvs用于采样 shadowCubes，
- shadowCubes 是一个2D Array, 假如有 n 个  Shadow Cube 光源 的话，这个array 就有 n 个，shadowCubes 保存的就是 Cubemap 经过 *Octahedron Environment Maps* 后的 2D texture
- 2D texture 保存类似于深度贴图，但是不是存储深度，而是 物体 到 光源 的 最近距离，利用自身到光源的距离 和 这个 最近距离来判断是否处理影子内

## 生成光源的shadow CubeMap

### 渲染
eevee_engine.c
```
static void EEVEE_cache_populate(void *vedata, Object *ob)
{
    ...
    const bool cast_shadow = true;

    if (cast_shadow) {
        EEVEE_lights_cache_shcaster_add(psl, stl, geom, ob->obmat);
    }
}

```

*eevee_lights.c*
```
/* Add a shadow caster to the shadowpasses */
void EEVEE_lights_cache_shcaster_add(EEVEE_PassList *psl, EEVEE_StorageList *stl, struct Batch *geom, float (*obmat)[4])
{
	DRWShadingGroup *grp = DRW_shgroup_instance_create(e_data.shadow_sh, psl->shadow_cube_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", stl->shadow_render_ubo);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	for (int i = 0; i < 6; ++i)
		DRW_shgroup_call_dynamic_add_empty(grp);

	grp = DRW_shgroup_instance_create(e_data.shadow_sh, psl->shadow_cascade_pass, geom);
	DRW_shgroup_uniform_block(grp, "shadow_render_block", stl->shadow_render_ubo);
	DRW_shgroup_uniform_mat4(grp, "ShadowModelMatrix", (float *)obmat);

	for (int i = 0; i < MAX_CASCADE_NUM; ++i)
		DRW_shgroup_call_dynamic_add_empty(grp);
}
```
>
- for (int i = 0; i < 6; ++i)  DRW_shgroup_call_dynamic_add_empty(grp);  这里是进行instance 6次，会改变 gl_InstanceID

<br>


```
void EEVEE_draw_shadows(EEVEE_Data *vedata)
{
    ...
    /* Render each shadow to one layer of the array */
	for (i = 0; (ob = linfo->shadow_cube_ref[i]) && (i < MAX_SHADOW_CUBE); i++) {
		EEVEE_LampEngineData *led = (EEVEE_LampEngineData *)DRW_lamp_engine_data_get(ob, &DRW_engine_viewport_eevee_type);
		EEVEE_ShadowCubeData *evscd = (EEVEE_ShadowCubeData *)led->sto;
		EEVEE_ShadowRender *srd = &linfo->shadow_render_data;

		srd->layer = i;
		copy_v3_v3(srd->position, ob->obmat[3]);
		for (int j = 0; j < 6; ++j) {
			copy_m4_m4(srd->shadowmat[j], evscd->viewprojmat[j]);
		}
		DRW_uniformbuffer_update(stl->shadow_render_ubo, &linfo->shadow_render_data);

		DRW_framebuffer_bind(fbl->shadow_cube_target_fb);
		DRW_framebuffer_clear(true, true, false, clear_color, 1.0);
		/* Render shadow cube */
		DRW_draw_pass(psl->shadow_cube_pass);

		/* Push it to shadowmap array */
		DRW_framebuffer_bind(fbl->shadow_cube_fb);
		DRW_draw_pass(psl->shadow_cube_store_pass);
	}
    ...
}
```
>
- 渲染Shadow Cube 光源是在 DRW_draw_pass(psl->shadow_cube_pass);处理
- psl->shadow_cube_pass 利用  shadow_vert.glsl 和 shadow_geom.glsl 和 shadow_frag.glsl 进行渲染 点光源的 cubemap

<br>

*shadow_vert.glsl* 

```

layout(std140) uniform shadow_render_block {
	mat4 ShadowMatrix[6];
	vec4 lampPosition;
	int layer;
};

uniform mat4 ShadowModelMatrix;

in vec3 pos;

out vec4 vPos;
out float lDist;

flat out int face;

void main() {
	vPos = ShadowModelMatrix * vec4(pos, 1.0);
	lDist = distance(lampPosition.xyz, vPos.xyz);
	face = gl_InstanceID;
}
```
>
- 这里的 gl_InstanceID 就是从 0-5 的变化，主要是把物体实例化了6次，每一次都为了渲染到cubemap的一个面上。

*shadow_geom.glsl*
```

layout(std140) uniform shadow_render_block {
	mat4 ShadowMatrix[6];
	vec4 lampPosition;
	int layer;
};

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

in vec4 vPos[];
in float lDist[];
flat in int face[];

out float linearDistance;

void main() {
	int f = face[0];
	gl_Layer = f;

	for (int v = 0; v < 3; ++v) {
		gl_Position = ShadowMatrix[f] * vPos[v];
		linearDistance = lDist[v];
		EmitVertex();
	}

	EndPrimitive();
}
```
>
- ShadowMatrix [6] 传入的就是cubemap 6个面的投影矩阵，实例化6次，分别把点投影到cubemap的6个面上

*shadow_frag.glsl*
```

in float linearDistance;

out vec4 FragColor;

void main() {
	FragColor = vec4(linearDistance, 0.0, 0.0, 1.0);
}

```

>
- ShadowModelMatrix 参数是 物体的世界矩阵 ，vPos = ShadowModelMatrix * vec4(pos, 1.0); 计算物体的世界坐标
- linearDistance 计算物体和光源的距离，渲染到cubemap上



## 光源的shadow CubeMap 转换为 2D texture

### 渲染
*eevee_lights.c*
```
void EEVEE_lights_cache_init(EEVEE_StorageList *stl, EEVEE_PassList *psl, EEVEE_TextureList *txl)
{
	...

	{
		psl->shadow_cube_store_pass = DRW_pass_create("Shadow Storage Pass", DRW_STATE_WRITE_COLOR);

		DRWShadingGroup *grp = DRW_shgroup_create(e_data.shadow_store_sh, psl->shadow_cube_store_pass);
		DRW_shgroup_uniform_buffer(grp, "shadowCube", &txl->shadow_color_cube_target);
		DRW_shgroup_uniform_block(grp, "shadow_render_block", stl->shadow_render_ubo);
		DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL);
	}

	...
}
```
>
- 这里是 DRW_shgroup_call_add(grp, DRW_cache_fullscreen_quad_get(), NULL); 表示 shadow_cube_store_pass 使用的是后处理来转换cubemap

<br>

```
void EEVEE_draw_shadows(EEVEE_Data *vedata)
{
    ...
    /* Render each shadow to one layer of the array */
	for (i = 0; (ob = linfo->shadow_cube_ref[i]) && (i < MAX_SHADOW_CUBE); i++) {
		EEVEE_LampEngineData *led = (EEVEE_LampEngineData *)DRW_lamp_engine_data_get(ob, &DRW_engine_viewport_eevee_type);
		EEVEE_ShadowCubeData *evscd = (EEVEE_ShadowCubeData *)led->sto;
		EEVEE_ShadowRender *srd = &linfo->shadow_render_data;

		srd->layer = i;
		copy_v3_v3(srd->position, ob->obmat[3]);
		for (int j = 0; j < 6; ++j) {
			copy_m4_m4(srd->shadowmat[j], evscd->viewprojmat[j]);
		}
		DRW_uniformbuffer_update(stl->shadow_render_ubo, &linfo->shadow_render_data);

		DRW_framebuffer_bind(fbl->shadow_cube_target_fb);
		DRW_framebuffer_clear(true, true, false, clear_color, 1.0);
		/* Render shadow cube */
		DRW_draw_pass(psl->shadow_cube_pass);

		/* Push it to shadowmap array */
		DRW_framebuffer_bind(fbl->shadow_cube_fb);
		DRW_draw_pass(psl->shadow_cube_store_pass);
	}
    ...
}
```

> 
- DRW_draw_pass(psl->shadow_cube_store_pass); 利用 shadow_store_vert.glsl  和  shadow_store_geom.glsl  和  shadow_store_frag.glsl 进行渲染，把cubemap渲染到texture上

<br>

### Shader

*shadow_store_vert.glsl*
```
in vec3 pos;

out vec4 vPos;

void main() {
	vPos = vec4(pos, 1.0);
}
```


*shadow_store_geom.glsl*
```

layout(std140) uniform shadow_render_block {
	mat4 ShadowMatrix[6];
	vec4 lampPosition;
	int layer;
};

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

in vec4 vPos[];

void main() {
	gl_Layer = layer;

	for (int v = 0; v < 3; ++v) {
		gl_Position = vPos[v];
		EmitVertex();
	}

	EndPrimitive();
}
```
>
- 渲染的时候是渲染到 texture 2D Array，Array 中每一个元素保存一个光源，在上面渲染的时候可以看到 srd->layer = i;, 这里gl_layer = layer 目的就是为了这个。

<br>

*shadow_store_frag.glsl*
```

uniform samplerCube shadowCube;

out vec4 FragColor;

vec3 octahedral_to_cubemap_proj(vec2 co)
{
	co = co * 2.0 - 1.0;

	vec2 abs_co = abs(co);
	vec3 v = vec3(co, 1.0 - (abs_co.x + abs_co.y));

	if ( abs_co.x + abs_co.y > 1.0 ) {
		v.xy = (abs(co.yx) - 1.0) * -sign(co.xy);
	}

	return v;
}

void main() {
	const vec2 texelSize = vec2(1.0 / 512.0);

	vec2 uvs = gl_FragCoord.xy * texelSize;

	/* add a 2 pixel border to ensure filtering is correct */
	uvs.xy *= 1.0 + texelSize * 2.0;
	uvs.xy -= texelSize;

	float pattern = 1.0;

	/* edge mirroring : only mirror if directly adjacent 
	 * (not diagonally adjacent) */   
	vec2 m = abs(uvs - 0.5) + 0.5;
	vec2 f = floor(m);
	if (f.x - f.y != 0.0) {
		uvs.xy = 1.0 - uvs.xy;
	}

	/* clamp to [0-1] */
	uvs.xy = fract(uvs.xy);

	/* get cubemap vector */
	vec3 cubevec = octahedral_to_cubemap_proj(uvs.xy);

	/* get cubemap vector */
	FragColor = texture(shadowCube, cubevec).rrrr;
}
```
>
- octahedral_to_cubemap_proj 的主要目的是转换 uv 为一个 向量，用来采样 cubemap，然后就可以达到 把 cubemap 渲染到 一张 texture 上
- octahedral_to_cubemap_proj 的原来可以参考  *2D -> Cubemap* 