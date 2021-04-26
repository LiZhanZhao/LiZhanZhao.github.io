---
layout:     post
title:      "blender eevee Replace ESM and VSM by PCF shadow mapping"
subtitle:   ""
date:       2021-4-26 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2019/8/26  *   Eevee : Replace ESM and VSM by PCF shadow mapping. <br> 

> 
PCF shadowmaps are less prone to light leaking and are faster to
render.

>
This remove a substantial part of the shadowing code.


> SVN : 2019/8/20  *  Windows: OpenXR SDK 1.0.0


<br><br>

## 作用
实现 PCF


<br><br>

## 效果

![](/img/Eevee/PCF/01/1.png)

<br><br>


### Shader

*lights_lib.glsl*
```
float light_visibility(LightData ld,
                       vec3 W,
#ifndef VOLUMETRICS
                       vec3 viewPosition,
                       vec3 vN,
#endif
                       vec4 l_vector)

{
	 if (ld.l_type == SUN) {
      vis *= shadow_cascade(data, int(data.sh_data_start), data.sh_tex_start, W);
    }
    else {
      vis *= shadow_cubemap(
          data, shadows_cube_data[int(data.sh_data_start)], data.sh_tex_start, W);
    }
}
```
>
- 最主要的是这两个函数 shadow_cascade + shadow_cubemap

<br><br>


#### shadow_cascade

```
float shadow_cascade(ShadowData sd, int scd_id, float texid, vec3 W)
{
  vec4 view_z = vec4(dot(W - cameraPos, cameraForward));
  vec4 weights = smoothstep(shadows_cascade_data[scd_id].split_end_distances,
                            shadows_cascade_data[scd_id].split_start_distances.yzwx,
                            view_z);

  weights.yzw -= weights.xyz;

  vec4 vis = vec4(1.0);

  /* Branching using (weights > 0.0) is reaally slooow on intel so avoid it for now. */
  /* TODO OPTI: Only do 2 samples and blend. */
  vis.x = evaluate_cascade(sd,
                           shadows_cascade_data[scd_id].shadowmat[0],
                           shadows_cascade_data[scd_id].shadowmat[0],
                           W,
                           texid + 0);
  vis.y = evaluate_cascade(sd,
                           shadows_cascade_data[scd_id].shadowmat[1],
                           shadows_cascade_data[scd_id].shadowmat[0],
                           W,
                           texid + 1);
  vis.z = evaluate_cascade(sd,
                           shadows_cascade_data[scd_id].shadowmat[2],
                           shadows_cascade_data[scd_id].shadowmat[0],
                           W,
                           texid + 2);
  vis.w = evaluate_cascade(sd,
                           shadows_cascade_data[scd_id].shadowmat[3],
                           shadows_cascade_data[scd_id].shadowmat[0],
                           W,
                           texid + 3);

  float weight_sum = dot(vec4(1.0), weights);
  if (weight_sum > 0.9999) {
    float vis_sum = dot(vec4(1.0), vis * weights);
    return vis_sum / weight_sum;
  }
  else {
    float vis_sum = dot(vec4(1.0), vis * step(0.001, weights));
    return mix(1.0, vis_sum, weight_sum);
  }
}



float evaluate_cascade(ShadowData sd, mat4 shadowmat, mat4 shadowmat0, vec3 W, float texid)
{
  vec4 shpos = shadowmat * vec4(W, 1.0);

  vec2 co = shpos.xy;
  float bias = sd.sh_bias;

	// co 进行一个随机偏移
#ifndef VOLUMETRICS
  float fac = length(shadowmat0[0].xyz) / length(shadowmat[0].xyz);
  vec3 rand = texelfetch_noise_tex(gl_FragCoord.xy).zwy;
  float ofs_len = fast_sqrt(rand.z) * sd.sh_blur * 0.05 / fac;
  /* This assumes that shadowmap is a square (w == h)
   * and that all cascades are the same dimensions. */
  bias *= ofs_len * 60.0 * fac;
  co += rand.xy * ofs_len;
#endif

  float vis = sample_cascade(shadowCascadeTexture, co, shpos.z - bias, texid);

  /* If fragment is out of shadowmap range, do not occlude */
  if (shpos.z < 1.0 && shpos.z > 0.0) {
    return vis;
  }
  else {
    return 1.0;
  }
}

float sample_cascade(sampler2DArrayShadow tex, vec2 co, float dist, float cascade_id)
{
  return texture(tex, vec4(co, cascade_id, dist));
}

```
>
- 看注释，主要是 evaluate_cascade 里面，对co进行一个随机数偏移

<br><br>


#### shadow_cubemap

```
float cubeFaceIndexEEVEE(vec3 P)
{
  vec3 aP = abs(P);
  if (all(greaterThan(aP.xx, aP.yz))) {
    return (P.x > 0.0) ? 0.0 : 1.0;
  }
  else if (all(greaterThan(aP.yy, aP.xz))) {
    return (P.y > 0.0) ? 2.0 : 3.0;
  }
  else {
    return (P.z > 0.0) ? 4.0 : 5.0;
  }
}

vec2 cubeFaceCoordEEVEE(vec3 P)
{
  vec3 aP = abs(P);
  if (all(greaterThan(aP.xx, aP.yz))) {
    return (P.zy / P.x) * vec2(-0.5, -sign(P.x) * 0.5) + 0.5;
  }
  else if (all(greaterThan(aP.yy, aP.xz))) {
    return (P.xz / P.y) * vec2(sign(P.y) * 0.5, 0.5) + 0.5;
  }
  else {
    return (P.xy / P.z) * vec2(0.5, -sign(P.z) * 0.5) + 0.5;
  }
}


vec4 sample_cube(sampler2DArray tex, vec3 cubevec, float cube)
{
  /* Manual Shadow Cube Layer indexing. */
  /* TODO Shadow Cube Array. */
  vec3 coord = vec3(cubeFaceCoordEEVEE(cubevec), cube * 6.0 + cubeFaceIndexEEVEE(cubevec));
  return texture(tex, coord);
}


float shadow_cubemap(ShadowData sd, ShadowCubeData scd, float texid, vec3 W)
{
  vec3 cubevec = W - scd.position.xyz;
  float dist = max_v3(abs(cubevec));
  float bias = sd.sh_bias;

#ifndef VOLUMETRICS
  vec3 rand = texelfetch_noise_tex(gl_FragCoord.xy).zwy;
  float ofs_len = fast_sqrt(rand.z) * sd.sh_blur * 0.1;
  bias *= ofs_len * 400.0;
  cubevec = normalize(cubevec);
  vec3 T, B;
  make_orthonormal_basis(cubevec, T, B);
  cubevec += (T * rand.x + B * rand.y) * ofs_len;
#endif
  dist = buffer_depth(true, dist - bias, sd.sh_far, sd.sh_near);

  return sample_cube(shadowCubeTexture, cubevec, dist, texid);
}
```
>
- 这里主要的代码是 shadow_cubemap ，是对 cubevec 进行了偏移