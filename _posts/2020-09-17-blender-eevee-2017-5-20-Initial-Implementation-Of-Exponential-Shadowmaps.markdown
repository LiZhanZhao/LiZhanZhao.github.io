---
layout:     post
title:      "blender eevee Initial Implementation Of Exponential Shadowmaps"
subtitle:   ""
date:       2020-12-25 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/5/20  * Eevee: Initial implementation of exponential shadowmaps  <br>
						Also fixes the rendering of depth.  <br>
  

> SVN : 2017/4/5  MSVC 2015 windows x64 (vc140) Alembic 1.7.1



## 作用 
- 初始实现 exponential shadowmaps 效果


## 编译
- 直接编译


## 效果

![](/img/Eevee/ExponentialShadowmaps/1.png)

## 概括
这篇很多知识都用到了上一篇[blender eevee Move Cube Shadows To Octahedron Shadowmaps](http://shaderstore.cn/2020/12/21/blender-eevee-2017-5-20-Move-Cube-Shadows-To-Octahedron-Shadowmaps/) 

## 优化生成shadow cubemap 的 Shader

*shadow_vert.glsl*
```

uniform mat4 ShadowModelMatrix;

in vec3 pos;

out vec4 vPos;

flat out int face;

void main() {
	vPos = ShadowModelMatrix * vec4(pos, 1.0);
	face = gl_InstanceID;
}
```


*shadow_gero.glsl*
```

layout(std140) uniform shadow_render_block {
	mat4 ShadowMatrix[6];
	vec4 lampPosition;
	int layer;
};

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

in vec4 vPos[];
flat in int face[];

out vec3 worldPosition;

void main() {
	int f = face[0];
	gl_Layer = f;

	for (int v = 0; v < 3; ++v) {
		gl_Position = ShadowMatrix[f] * vPos[v];
		worldPosition = vPos[v].xyz;
		EmitVertex();
	}

	EndPrimitive();
}
```

*shadow_frag.glsl*
```

layout(std140) uniform shadow_render_block {
	mat4 ShadowMatrix[6];
	vec4 lampPosition;
	int layer;
};

in vec3 worldPosition;

out vec4 FragColor;

void main() {
	float dist = distance(lampPosition.xyz, worldPosition.xyz);
	FragColor = vec4(exp(5.0 * dist), 0.0, 0.0, 1.0);
}

```

>
- 现在cubemap保存的是 exp(5.0 * dist), 上一篇是 dist，这里是主要的区别



## 优化 cubemap -> texture 的 Shader

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

void make_orthonormal_basis(vec3 N, out vec3 T, out vec3 B)
{
	vec3 UpVector = abs(N.z) < 0.99999 ? vec3(0.0,0.0,1.0) : vec3(1.0,0.0,0.0);
	T = normalize( cross(UpVector, N) );
	B = cross(N, T);
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

	vec3 T, B;
	make_orthonormal_basis(cubevec, T, B);

	vec2 offsetvec = texelSize.xy * vec2(-1.0, 1.0); /* Totally arbitrary */

	/* get cubemap shadow value */
	FragColor  = texture(shadowCube, cubevec + offsetvec.x * T + offsetvec.x * B).rrrr;
	FragColor += texture(shadowCube, cubevec + offsetvec.x * T + offsetvec.y * B).rrrr;
	FragColor += texture(shadowCube, cubevec + offsetvec.y * T + offsetvec.x * B).rrrr;
	FragColor += texture(shadowCube, cubevec + offsetvec.y * T + offsetvec.y * B).rrrr;

	FragColor /= 4.0;
}
```
>
- 跟上一篇最大的差别就是 这里计算 offsetvec , 去附近的像素进行模糊,  

如下：
```
vec3 T, B;
make_orthonormal_basis(cubevec, T, B);

vec2 offsetvec = texelSize.xy * vec2(-1.0, 1.0); /* Totally arbitrary */

/* get cubemap shadow value */
FragColor  = texture(shadowCube, cubevec + offsetvec.x * T + offsetvec.x * B).rrrr;
FragColor += texture(shadowCube, cubevec + offsetvec.x * T + offsetvec.y * B).rrrr;
FragColor += texture(shadowCube, cubevec + offsetvec.y * T + offsetvec.x * B).rrrr;
FragColor += texture(shadowCube, cubevec + offsetvec.y * T + offsetvec.y * B).rrrr;

FragColor /= 4.0;
```



## 应用
*lit_surface_frag.glsl*

```


float light_visibility(LightData ld, ShadingData sd)
{
	float vis = 1.0;

	...

	/* shadowing */
	if (ld.l_shadowid >= (MAX_SHADOW_MAP + MAX_SHADOW_CUBE)) {
		/* Shadow Cascade */
		...
	}
	else if (ld.l_shadowid >= 0.0) {
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

		float z = texture(shadowCubes, vec3(uvs, shid)).r;

		float esm_test = min(1.0, exp(-5.0 * dist) * z);
		float sh_test = step(0, z - dist);

		vis *= esm_test;
	}

	return vis;
}

```

主要的不同是
```
float z = texture(shadowCubes, vec3(uvs, shid)).r;
float esm_test = min(1.0, exp(-5.0 * dist) * z);
vis *= esm_test;
```

>
- 上面可以知道，z保存的是 exp(5.0 * dist_min), dist_min 最近的距离
- min(1.0, exp(-5.0 * dist) * z);   就相当于  exp(-5.0 * dist) * exp(5.0 * dist_min) = exp(5.0 * (dist_min - dist))
- 那么esm_test 计算出来的 0-1 之间的值，在阴影内的，会有一个衰减的效果，因为做了 exp 运算