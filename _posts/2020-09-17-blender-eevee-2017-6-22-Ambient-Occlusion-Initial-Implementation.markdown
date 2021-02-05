---
layout:     post
title:      "blender eevee Ambient Occlusion Initial implementation."
subtitle:   ""
date:       2021-2-5 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/22  * Eevee : Ambient Occlusion: Initial implementation.<br> 

>
- Implement GTAO (Ground Truth Ambient Occlusion) which is a special case of Horizon Based Ambient Occlusion that is more physically accurate.
Also add a bent normal option to sample indirect irradiance (diffuse lighting) with the least occluded direction.


> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		

## 作用 
实现GTAO, 后处理AO效果


## 编译
- 重新生成SLN
- git 定位到  2017/6/22  * Eevee : Ambient Occlusion: Initial implementation .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 效果
开启功能的效果对比
![](/img/Eevee/GTAO/0.png)
![](/img/Eevee/GTAO/1.png)
![](/img/Eevee/GTAO/2.png)

## Shader
GTAO的效果，最主要是在Shader中实现的

*lit_surface_frag.glsl*
```

float compute_occlusion(vec3 N, float micro_occlusion, vec2 randuv, out vec3 bent_normal)
{
#ifdef USE_AO /* Screen Space Occlusion */

	float macro_occlusion;
	vec3 vnor = mat3(ViewMatrix) * N;

#ifdef USE_BENT_NORMAL
	gtao(vnor, viewPosition, randuv, macro_occlusion, bent_normal);
	bent_normal = mat3(ViewMatrixInverse) * bent_normal;
#else
	gtao(vnor, viewPosition, randuv, macro_occlusion);
	bent_normal = N;
#endif
	return min(macro_occlusion, micro_occlusion);

#else /* No added Occlusion. */

	bent_normal = N;
	return micro_occlusion;

#endif
}

vec3 eevee_surface_lit(vec3 world_normal, vec3 albedo, vec3 f0, float roughness, float ao)
{
	...
	vec3 bent_normal;
	vec4 rand = textureLod(utilTex, vec3(gl_FragCoord.xy / LUT_SIZE, 2.0), 0.0).rgba;
	float final_ao = compute_occlusion(sd.N, ao, rand.rg, bent_normal);
	...

	vec3 indirect_radiance =
	        spec_accum.rgb * F_ibl(f0, brdf_lut) * float(specToggle) * specular_occlusion(dot(sd.N, sd.V), final_ao, roughness) +
	        diff_accum.rgb * albedo * final_ao;

	return radiance + indirect_radiance;
}
```
>
- 核心是如何在函数compute_occlusion中计算出finial_ao, compute_occlusion函数中核心就是 **函数gtao**




## gtao函数理论

最主要是参考 [这里](http://blog.selfshadow.com/publications/s2016-shading-course/activision/s2016_pbs_activision_occlusion.pdf)

### What is ambient occlusion

Let’s start with the rendering equation, where you can see the emitted and incoming 
radiance and the BRDF.<br>
(我们从渲染公式开始)

![](/img/Eevee/GTAO/PDF/0.png)<br>


If we assume there is no emission, and we use the Lambertian BRDF…<br>
(这里假设没有自发光+使用Lambertian BRDF)

![](/img/Eevee/GTAO/PDF/1.png)<br>

If we then assume a constant white dome is illuminating the scene, and we only 
calculate a single bounce of light…<br>
(假设一个 恒定不变的白色光源 去照亮场景, 然后只考虑光源射出的光线只反弹一次)

Notice a new visibility term appeared, that specifies if a ray hits the sky or not.<br>
(个人理解这个 visibility term, 就是在 受光点往wi方向 射出 ray，如果ray碰到了sky就变成这个wi是可视的，如果碰不到就是表示wi方向是不可视的)


![](/img/Eevee/GTAO/PDF/2.png)<br>


So, we can say that the ambient occlusion multiplied by the albedo is the lighting for 
the case of:<br>
a white dome,<br>
a single bounce of light and<br>
a Lambertian surface.<br>
(在上面3个假设的前提下，整理公式如下)

![](/img/Eevee/GTAO/PDF/3.png)<br>


We can calculate the ambient occlusion integral as a double integral in polar coordinates.<br>
(用极坐标表示积分公式)<br><br>
The inner integral integrates the visibility for a slice of the hemisphere, as you can see 
in the left,and the outer integral swipes this slice to cover the full hemisphere<br>
(个人理解，受光点会收到周围360度的入射光射入，这360度入射光可以形成一个半圆，内部的积分的意思就是，先取半圆的一片，1角度为一片半圆，就是 a slice of the hemisphere 的意思, 计算出 a slice of the hemisphere 的 值，然后在进行外部的积分，0-180度，每一度一片半圆积分起来)

![](/img/Eevee/GTAO/PDF/4.png)<br>

<br>
The simplest solution would be to just numerically solve both integrals.
But the solution we chosen, horizon-based ambient occlusion, which was introduced 
by Louis Bavoil in 2008,
made the key observation that the occlusion as pictured here can’t happen when 
working with height fields.
Using height-fields we would never be able to tell that the areas in green here, are actually visible.
<br>
<br>
The key consequence of this, is that we can just search for the two horizons h1 and 
h2 and that captures all the visibility information that can be extracted from a height 
map,for a given slice.
<br>
(个人理解就是，目前在受光点的左右两边，找到h1和h2, h1 和 h2 之间是可视的，其他都不是可视的, 判断其他点和受光点的角度就可以 计算出 h1 和 h2)

![](/img/Eevee/GTAO/PDF/5.png)
![](/img/Eevee/GTAO/PDF/6.png)<br>


To solve ambient occlusion with cosine weighting,
we integrate the visibility from the view vector to h1, marked in green,
and the visibility from the view vector to h2, marked in blue,
taking the cosine term into account, marked in purple on the equation<br>
(得到h1 和 h2 的话，就可以计算vd)
![](/img/Eevee/GTAO/PDF/7.png)<br>



(这里还要考虑Normal不在slice的情况)
![](/img/Eevee/GTAO/PDF/8.png)<br>




## gtao函数

*ambient_occlusion_lib.glsl*
```
/* Based on Practical Realtime Strategies for Accurate Indirect Occlusion
 * http://blog.selfshadow.com/publications/s2016-shading-course/activision/s2016_pbs_activision_occlusion.pdf
 * http://blog.selfshadow.com/publications/s2016-shading-course/activision/s2016_pbs_activision_occlusion.pptx */

#define MAX_PHI_STEP 32
/* NOTICE : this is multiplied by 2 */
#define MAX_THETA_STEP 6.0

uniform sampler2D minMaxDepthTex;
uniform float aoDistance;
uniform float aoSamples;
uniform float aoFactor;

float sample_depth(vec2 co, int level)
{
	return textureLod(minMaxDepthTex, co, float(level)).g;
}

//x view space coordinate
float get_max_horizon(vec2 co, vec3 x, float h, float step)
{
	if (co.x > 1.0 || co.x < 0.0 || co.y > 1.0 || co.y < 0.0)
		return h;

	float depth = sample_depth(co, int(step));

	/* Background case */
	if (depth == 1.0)
		return h;

	vec3 s = get_view_space_from_depth(co, depth); /* s View coordinate */
	vec3 omega_s = s - x;
	float len = length(omega_s);

	//  omega_s.z / len => cos (h) 的结果
	float max_h = max(h, omega_s.z / len);
	/* Blend weight after half the aoDistance to fade artifacts */
	float blend = saturate((1.0 - len / aoDistance) * 2.0);

	return mix(h, max_h, blend);
}

// position -> viewPosition
void gtao(vec3 normal, vec3 position, vec2 noise, out float visibility
#ifdef USE_BENT_NORMAL
	, out vec3 bent_normal
#endif
	)
{
	vec2 screenres = vec2(textureSize(minMaxDepthTex, 0)) * 2.0;
	vec2 pixel_size = vec2(1.0) / screenres.xy;

	/* Renaming */
	vec2 x_ = gl_FragCoord.xy * pixel_size; /* x^ Screen coordinate */
	vec3 x = position; /* x view space coordinate */

	/* NOTE : We set up integration domain around the camera forward axis
	 * and not the view vector like in the paper.
	 * This allows us to save a lot of dot products. */
	/* omega_o = vec3(0.0, 0.0, 1.0); */

	vec2 pixel_ratio = vec2(screenres.y / screenres.x, 1.0);
	float pixel_len = length(pixel_size);
	// viewPosition.z -> ClipPosition.z
	float homcco = ProjectionMatrix[2][3] * position.z + ProjectionMatrix[3][3];
	float max_dist = aoDistance / homcco; /* Search distance */

	/* Integral over PI */
	visibility = 0.0;
#ifdef USE_BENT_NORMAL
	bent_normal = vec3(0.0);
#endif
	for (float i = 0.0; i < aoSamples && i < MAX_PHI_STEP; i++) {
		// noise.r -> [0,1]
		float phi = M_PI * ((noise.r + i) / aoSamples);

		// 旋转
		/* Rotate with random direction to get jittered result. */
		vec2 t_phi = vec2(cos(phi), sin(phi)); /* Screen space direction */

		/* Search maximum horizon angles h1 and h2 */
		float h1 = -1.0, h2 = -1.0; /* init at cos(pi) */
		float ofs = 1.5 * pixel_len;
		for (float j = 0.0; ofs < max_dist && j < MAX_THETA_STEP; j += 0.5) {
			ofs += ofs; /* Step size is doubled each iteration */

			vec2 s_ = t_phi * ofs * noise.g * pixel_ratio; /* s^ Screen coordinate */
			vec2 co;

			co = x_ + s_;
			h1 = get_max_horizon(co, x, h1, j);

			co = x_ - s_;
			h2 = get_max_horizon(co, x, h2, j);
		}

		/* (Slide 54) */
		h1 = -acos(h1);
		h2 = acos(h2);

		/* Projecting Normal to Plane P defined by t_phi and omega_o */
		vec3 h = vec3(t_phi.y, -t_phi.x, 0.0); /* Normal vector to Integration plane */ //积分平面的法向量 sn
		vec3 t = vec3(-t_phi, 0.0);														//积分平面的切向量
		vec3 n_proj = normal - h * dot(h, normal);
		float n_proj_len = max(1e-16, length(n_proj));

		/* Clamping thetas (slide 58) */
		float cos_n = clamp(n_proj.z / n_proj_len, -1.0, 1.0);
		float n = sign(dot(n_proj, t)) * acos(cos_n); /* Angle between view vec and normal */
		h1 = n + max(h1 - n, -M_PI_2);
		h2 = n + min(h2 - n, M_PI_2);

		/* Solving inner integral */
		float sin_n = sin(n);
		float h1_2 = 2.0 * h1;
		float h2_2 = 2.0 * h2;
		float vd = (-cos(h1_2 - n) + cos_n + h1_2 * sin_n) + (-cos(h2_2 - n) + cos_n + h2_2 * sin_n);
		vd *= 0.25 * n_proj_len;
		visibility += vd;

#ifdef USE_BENT_NORMAL
		/* Finding Bent normal */
		float b_angle = (h1 + h2) / 2.0;
		/* The 0.5 factor below is here to equilibrate the accumulated vectors.
		 * (sin(b_angle) * -t_phi) will accumulate to (phi_step * result_nor.xy * 0.5).
		 * (cos(b_angle) * 0.5) will accumulate to (phi_step * result_nor.z * 0.5). */
		/* Weight sample by vd */
		bent_normal += vec3(sin(b_angle) * -t_phi, cos(b_angle) * 0.5) * vd;
#endif
	}

	visibility = min(1.0, visibility / aoSamples);

#ifdef USE_BENT_NORMAL
	/* The bent normal will show the facet look of the mesh. Try to minimize this. */
	bent_normal = normalize(mix(bent_normal / visibility, normal, visibility * visibility * visibility));
#endif

	/* Scale by user factor */
	visibility = max(0.0, mix(aoFactor, 1.0, visibility));
}

/* Multibounce approximation base on surface albedo.
 * Page 78 in the .pdf version. */
float gtao_multibounce(float visibility, vec3 albedo)
{
	/* Median luminance. Because Colored multibounce looks bad. */
	float lum = albedo.x * 0.3333;
	lum += albedo.y * 0.3333;
	lum += albedo.z * 0.3333;

	float a =  2.0404 * lum - 0.3324;
	float b = -4.7951 * lum + 0.6417;
	float c =  2.7552 * lum + 0.6903;

	float x = visibility;
	return max(x, ((x * a + b) * x + c) * x);
}
```
>
- 想要了解到上面的GTAO的算法思路，一定要看 [这里](http://blog.selfshadow.com/publications/s2016-shading-course/activision/s2016_pbs_activision_occlusion.pdf), 这个pdf把思路都整的明明白白。
<br>
- 