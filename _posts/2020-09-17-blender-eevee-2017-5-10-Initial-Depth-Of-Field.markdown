---
layout:     post
title:      "blender eevee Initial Depth Of Fiedld"
subtitle:   ""
date:       2020-12-14 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/5/10  * Eevee: Initial Depth Of Field commit. <br>  

> SVN : 2017/4/5  MSVC 2015 windows x64 (vc140) Alembic 1.7.1





## 作用 
Bloom 景深效果



## 编译

- 直接编译


## 效果展示

![](/img/Eevee/DepthOfField/Dof.png)


## 算法理论基础

### 1. lens equation (成像公式)
参考 [成像公式推导](https://baike.baidu.com/item/%E6%88%90%E5%83%8F%E5%85%AC%E5%BC%8F)

推到凸透镜的成像规律 : 1/u+1/v=1/f（即 : 物距的倒数与像距的倒数之和等于焦距的倒数。）

![](/img/Eevee/DepthOfField/lens_equation.png)

【题】如右图 ，用几何法证明1/u+1/v=1/f。<br>  
【解】∵△ABO∽△A'B'O				<br>  
∴AB:A'B'=u:v					<br>  
∵△COF∽△A'B'F					<br>  
∴CO:A'B'=f:(v-f)					<br>  
∵四边形ABOC为矩形					<br>  
∴AB=CO								<br>  
∴AB:A'B'=f:(v-f)					<br>  
∴u:v=f:(v-f)						<br>  
∴u(v-f)=vf							<br>  
∴uv-uf=vf							<br>  
∵uvf≠0							<br>  
∴(uv/uvf)-(uf/uvf)=vf/uvf			<br>  
∴1/f-1/v=1/u					<br>  
即：1/u+1/v=1/f					<br>  


### 2. Determining a circle of confusion diameter from the object field 

参考 [Circle of confusion](https://en.wikipedia.org/wiki/Circle_of_confusion#Determining_a_circle_of_confusion_diameter_from_the_object_field)

![](/img/Eevee/DepthOfField/coc.png)

To calculate the diameter(直径) of the circle of confusion in the image plane for an out-of-focus subject, one method is to first calculate the diameter of the blur circle in a virtual image in the object plane, which is simply done using similar triangles, and then multiply by the magnification(放大倍数) of the system, which is calculated with the help of the lens equation.

The blur circle, of diameter C, in the focused object plane at distance S1, is an unfocused virtual image of the object at distance S2 as shown in the diagram. It depends only on these distances and the aperture diameter A, via similar triangles, independent of the lens focal length:

![](/img/Eevee/DepthOfField/Exp-01.png) 
(以上可以利用相似三角形计算得到)

The circle of confusion in the image plane is obtained by multiplying by magnification m:
![](/img/Eevee/DepthOfField/Exp-02.jpg) 

where the magnification m is given by the ratio of focus distances:
![](/img/Eevee/DepthOfField/Exp-03.jpg) 

Using the lens equation we can solve for the auxiliary variable f1:
![](/img/Eevee/DepthOfField/Exp-04.jpg)

which yields
![](/img/Eevee/DepthOfField/Exp-05.jpg)

and express the magnification in terms of focused distance and focal length:
![](/img/Eevee/DepthOfField/Exp-06.jpg)



which gives the final result:
![](/img/Eevee/DepthOfField/Exp-07.jpg)

(f :  focused distance  <br>  
S1 :  focal length     <br>  
S2 :  unforcal length
 )


> 这里需要注意的是，以上的只是计算 far的coc，那就是 比焦点远的物体的coc，如果是计算比焦点近的coc的话, 上面的 ![](/img/Eevee/DepthOfField/Exp-01.jpg)  
要变成
![](/img/Eevee/DepthOfField/Exp-10.png) 
最后的near coc计算就是
![](/img/Eevee/DepthOfField/Exp-11.jpg) 
而且，通过以上我们可以得到，far coc / near coc = -1, 例如 可以计算near coc，如果计算出来的 near coc 是 负数的话，表示这个物体不是 比 焦点近，而是比 焦点远，这样就可以确定 哪一些物体在 焦点的前面，哪一些物体在焦点的后面 

### 3. A circle (r=1) is inscribed in a right isosceles triangle. Find the perimeter of the triangle? 
这里是计算周长，但是涉及到的数学知识是一致的，我们只是计算三角形的边

参考 [Math](https://www.quora.com/A-circle-r-8-is-inscribed-in-a-right-isosceles-triangle-Find-the-perimeter-of-the-triangle)

![](/img/Eevee/DepthOfField/right_isosceles_triangle.png) 
![](/img/Eevee/DepthOfField/right_isosceles_triangle_1.png) 

> 之后的代码会用到这里的，我们先弄一些符号，方便描述，
![](/img/Eevee/DepthOfField/right_isosceles_triangle_2.png) 
我们要计算 AB 的长度, 已知圆的半径为1  
FG = BG = 1  
cos (pi / 4)  = BG / BF  
cos (pi / 4)  = 1 / BF  
BF = 1  /   cos (pi / 4)  
BE = BF + FE =  ( 1  /   cos (pi / 4)  )  + 1  
cos (pi / 4) = BD / BE  
BD = BE * cos (pi / 4)  =  1 + cos (pi / 4)  
BD = AD
AB = AD + BD =  (  1 + cos (pi / 4)   ) * 2  
所以就有代码的
```
/* Extend to cover at least the unit circle */
	const float extend = (cos(M_PI / 4.0) + 1.0) * 2.0;
```








### 4. linear_depth
主要是Shader通过这个函数，把 buffer的深度，变换成 view 空间
参考<br>  
[linear_depth](http://web.archive.org/web/20130416194336/http://olivers.posterous.com/linear-depth-in-glsl-for-real)
<br>  
[projectionmatrix](http://www.songho.ca/opengl/gl_projectionmatrix.html)


the link between eye-space Z (z_e below) and normalised device coordinates (NDC) Z (z_n below). From there, we have

A   = -(zFar + zNear) / (zFar - zNear);
B   = -2*zFar*zNear / (zFar - zNear);
z_n = -(A*z_e + B) / z_e; // z_n in [-1, 1]

Note that the value stored in the depth buffer is actually in the range [0, 1], so the depth buffer value z_b is:

z_b = 0.5*z_n + 0.5; // z_b in [0, 1]

If we have rendered this depth buffer to a texture and wish to access the real depth in a later shader, we must undo the non-linear mapping above:

z_e = 2*zFar*zNear / (zFar + zNear - (zFar - zNear)*(2*z_b -1));

(经过代入计算, 感觉自己计算出来的结果和 以上的公式 多一个负号)


计算得到

```
float linearize_depth(float d,float zNear,float zFar)
{
    return zNear * zFar / (zFar + d * (zNear - zFar));
}
```





## Dof 算法思路

Depth of Field algorithm
		 
Overview :
- Downsample the color buffer into 2 buffers weighted with
CoC values. Also output CoC into a texture.
- Shoot quads for every pixel and expand it depending on the CoC.
Do one pass for near Dof and one pass for far Dof.
- Finally composite the 2 blurred buffers with the original render.


## 渲染
```
/* Depth Of Field */
	if ((effects->enabled_effects & EFFECT_DOF) != 0) {
		float clear_col[4] = {0.0f, 0.0f, 0.0f, 0.0f};

		/* Downsample */
		DRW_framebuffer_bind(fbl->dof_down_fb);
		DRW_draw_pass(psl->dof_down);

		/* Scatter Far */
		effects->unf_source_buffer = txl->dof_down_far;
		copy_v2_fl2(effects->dof_layer_select, 0.0f, 1.0f);
		DRW_framebuffer_bind(fbl->dof_scatter_far_fb);
		DRW_framebuffer_clear(true, false, false, clear_col, 0.0f);
		DRW_draw_pass(psl->dof_scatter);

		/* Scatter Near */
		effects->unf_source_buffer = txl->dof_down_near;
		copy_v2_fl2(effects->dof_layer_select, 1.0f, 0.0f);
		DRW_framebuffer_bind(fbl->dof_scatter_near_fb);
		DRW_framebuffer_clear(true, false, false, clear_col, 0.0f);
		DRW_draw_pass(psl->dof_scatter);

		/* Resolve */
		DRW_framebuffer_bind(effects->target_buffer);
		DRW_draw_pass(psl->dof_resolve);
		SWAP_BUFFERS();
	}
```
### Downsample

#### 传入Shader参数

```
/* Retreive Near and Far distance */
effects->dof_near_far[0] = -cam->clipsta;
effects->dof_near_far[1] = -cam->clipend;

/* Parameters */
/* TODO UI Options */
float fstop = cam->gpu_dof.fstop;
float blades = cam->gpu_dof.num_blades;
float rotation = 0.0f;
float ratio = 1.0f;
float sensor = BKE_camera_sensor_size(cam->sensor_fit, cam->sensor_x, cam->sensor_y);
float focus_dist = BKE_camera_object_dof_distance(v3d->camera);
float focal_len = cam->lens;

UNUSED_VARS(rotation, ratio);

/* this is factor that converts to the scene scale. focal length and sensor are expressed in mm
    * unit.scale_length is how many meters per blender unit we have. We want to convert to blender units though
    * because the shader reads coordinates in world space, which is in blender units.
    * Note however that focus_distance is already in blender units and shall not be scaled here (see T48157). */
float scale = (scene->unit.system) ? scene->unit.scale_length : 1.0f;
float scale_camera = 0.001f / scale;
/* we want radius here for the aperture number  */
float aperture = 0.5f * scale_camera * focal_len / fstop;
float focal_len_scaled = scale_camera * focal_len;
float sensor_scaled = scale_camera * sensor;

effects->dof_params[0] = aperture * fabsf(focal_len_scaled / (focus_dist - focal_len_scaled));
effects->dof_params[1] = -focus_dist;
effects->dof_params[2] = viewport_size[0] / sensor_scaled;

...


psl->dof_down = DRW_pass_create("DoF Downsample", DRW_STATE_WRITE_COLOR);

grp = DRW_shgroup_create(e_data.dof_downsample_sh, psl->dof_down);
DRW_shgroup_uniform_buffer(grp, "colorBuffer", &effects->source_buffer, 0);
DRW_shgroup_uniform_buffer(grp, "depthBuffer", &dtxl->depth, 1);
DRW_shgroup_uniform_vec2(grp, "nearFar", effects->dof_near_far, 1);
DRW_shgroup_uniform_vec3(grp, "dofParams", effects->dof_params, 1);
DRW_shgroup_call_add(grp, quad, NULL);

```


#### Shader

*effect_dof_vert.glsl*
```

void passthrough()
{
	uvcoord = uvs;
	gl_Position = vec4(pos, 0.0, 1.0);
}

void main()
{
    passthrough();
}

```


*effect_dof_frag.glsl*
```



uniform mat4 ProjectionMatrix;

uniform sampler2D colorBuffer;
uniform sampler2D depthBuffer;

uniform vec3 dofParams;

#define dof_aperturesize    dofParams.x
#define dof_distance        dofParams.y
#define dof_invsensorsize   dofParams.z

uniform vec3 bokehParams;

#define bokeh_sides         bokehParams.x /* Polygon Bokeh shape number of sides */
#define bokeh_rotation      bokehParams.y

uniform vec2 nearFar; /* Near & far view depths values */

/* initial uv coordinate */
in vec2 uvcoord;

layout(location = 0) out vec4 fragData0;
layout(location = 1) out vec4 fragData1;
layout(location = 2) out vec4 fragData2;

#define M_PI 3.1415926535897932384626433832795
#define M_2PI 6.2831853071795864769252868

/* -------------- Utils ------------- */

/* calculate 4 samples at once */
float calculate_coc(in float zdepth)
{
	float coc = dof_aperturesize * (dof_distance / zdepth - 1.0);

	/* multiply by 1.0 / sensor size to get the normalized size */
	return coc * dof_invsensorsize;
}

vec4 calculate_coc(in vec4 zdepth)
{
	vec4 coc = dof_aperturesize * (vec4(dof_distance) / zdepth - vec4(1.0));

	/* multiply by 1.0 / sensor size to get the normalized size */
	return coc * dof_invsensorsize;
}

float max4(vec4 x)
{
    return max(max(x.x, x.y), max(x.z, x.w));
}

float linear_depth(float z)
{
	/* if persp */
	if (ProjectionMatrix[3][3] == 0.0) {
		return (nearFar.x  * nearFar.y) / (z * (nearFar.x - nearFar.y) + nearFar.y);
	}
	else {
		return (z * 2.0 - 1.0) * nearFar.y;
	}
}

vec4 linear_depth(vec4 z)
{
	/* if persp */
	if (ProjectionMatrix[3][3] == 0.0) {
		return (nearFar.xxxx  * nearFar.yyyy) / (z * (nearFar.xxxx - nearFar.yyyy) + nearFar.yyyy);
	}
	else {
		return (z * 2.0 - 1.0) * nearFar.yyyy;
	}
}

#define THRESHOLD 0.0

/* ----------- Steps ----------- */

/* Downsample the color buffer to half resolution.
 * Weight color samples by
 * Compute maximum CoC for near and far blur. */
void step_downsample(void)
{
    //  当viewport 范围 为（0，0，800，600）时， gl_FragCoord 的 x, y 的取值范围为（0.5, 0.5, 799.5, 599.5),
	ivec4 uvs = ivec4(gl_FragCoord.xyxy) * 2 + ivec4(0, 0, 1, 1);  

    // 这里就是取4个像素的值出来，(xy) + { (0,0), (0,1), (1,0), (1,1) }
	/* custom downsampling */
	vec4 color1 = texelFetch(colorBuffer, uvs.xy, 0);
	vec4 color2 = texelFetch(colorBuffer, uvs.zw, 0);
	vec4 color3 = texelFetch(colorBuffer, uvs.zy, 0);
	vec4 color4 = texelFetch(colorBuffer, uvs.xw, 0);

	/* Leverage SIMD by combining 4 depth samples into a vec4 */
	vec4 depth;
	depth.r = texelFetch(depthBuffer, uvs.xy, 0).r;
	depth.g = texelFetch(depthBuffer, uvs.zw, 0).r;
	depth.b = texelFetch(depthBuffer, uvs.zy, 0).r;
	depth.a = texelFetch(depthBuffer, uvs.xw, 0).r;

	vec4 zdepth = linear_depth(depth);

    // 这里是计算 coc_near, 可以通过 coc_far 和 coc_near 互换相反数, 可以通过 判断 像素的 coc 的符号来判断是否是 near 或者 far
	/* Compute signed CoC for each depth samples */
	vec4 coc_near = calculate_coc(zdepth);
	vec4 coc_far = -coc_near;

	/* now we need to write the near-far fields premultiplied by the coc */
	vec4 near_weights = step(THRESHOLD, coc_near);
	vec4 far_weights = step(THRESHOLD, coc_far);

	/* now write output to weighted buffers. */
	fragData0 = color1 * near_weights.x +
	            color2 * near_weights.y +
	            color3 * near_weights.z +
	            color4 * near_weights.w;

	fragData1 = color1 * far_weights.x +
	            color2 * far_weights.y +
	            color3 * far_weights.z +
	            color4 * far_weights.w;

    // dot 这里计算4个像素有多少个像素处于 near 和 far, 因为平均像素，模糊操作
	float norm_near = dot(near_weights, near_weights);
	float norm_far = dot(far_weights, far_weights);

	if (norm_near > 0.0) {
		fragData0 /= norm_near;
	}

	if (norm_far > 0.0) {
		fragData1 /= norm_far;
	}

	float max_near_coc = max(max4(coc_near), 0.0);
	float max_far_coc = max(max4(coc_far), 0.0);

	fragData2 = vec4(max_near_coc, max_far_coc, 0.0, 1.0);
}


void main()
{
#ifdef STEP_DOWNSAMPLE
	step_downsample();
#elif defined(STEP_SCATTER)
	step_scatter();
#elif defined(STEP_RESOLVE)
	step_resolve();
#endif
}

```



### Scatter

#### 传入Shader参数
```
/* Scatter Far */
effects->unf_source_buffer = txl->dof_down_far;
copy_v2_fl2(effects->dof_layer_select, 0.0f, 1.0f);

... 

/* Scatter Near */
effects->unf_source_buffer = txl->dof_down_near;
copy_v2_fl2(effects->dof_layer_select, 1.0f, 0.0f);


...


float blades = cam->gpu_dof.num_blades;
float rotation = 0.0f;
float ratio = 1.0f;

effects->dof_bokeh[0] = blades;
effects->dof_bokeh[1] = rotation;
effects->dof_bokeh[2] = ratio

/* This create an empty batch of N triangles to be positioned
* by the vertex shader 0.4ms against 6ms with instancing */
const float *viewport_size = DRW_viewport_size_get();
const int sprite_ct = ((int)viewport_size[0]/2) * ((int)viewport_size[1]/2); /* brackets matters */
grp = DRW_shgroup_empty_tri_batch_create(e_data.dof_scatter_sh, psl->dof_scatter, sprite_ct);

DRW_shgroup_uniform_buffer(grp, "colorBuffer", &effects->unf_source_buffer, 0);
DRW_shgroup_uniform_buffer(grp, "cocBuffer", &txl->dof_coc, 1);
DRW_shgroup_uniform_vec2(grp, "layerSelection", effects->dof_layer_select, 1);
DRW_shgroup_uniform_vec3(grp, "bokehParams", effects->dof_bokeh, 1);
    
```


#### Shader

*effect_dof_vert.glsl*
```

flat out vec4 color;
out vec2 particlecoord;

#define M_PI 3.1415926535897932384626433832795

/* geometry shading pass, calculate a texture coordinate based on the indexed id */
void step_scatter()
{
	ivec2 tex_size = textureSize(cocBuffer, 0);
	vec2 texel_size = 1.0 / vec2(tex_size);

	int t_id = gl_VertexID / 3; /* Triangle Id */

	ivec2 texelco = ivec2(0);
	/* some math to get the target pixel */
	texelco.x = t_id % tex_size.x;
	texelco.y = t_id / tex_size.x;

	float coc = dot(layerSelection, texelFetch(cocBuffer, texelco, 0).rg);

	/* Clamp to max size for performance */
	coc = min(coc, 100.0);

	if (coc >= 1.0) {
		color = texelFetch(colorBuffer, texelco, 0);
		/* find the area the pixel will cover and divide the color by it */
		float alpha = 1.0 / (coc * coc * M_PI); // Area of a Circle
		color *= alpha;
		color.a = alpha;
	}
	else {
		color = vec4(0.0);
	}

	/* Generate Triangle : less memory fetches from a VBO */
	int v_id = gl_VertexID % 3; /* Vertex Id */

	// https://www.quora.com/A-circle-r-8-is-inscribed-in-a-right-isosceles-triangle-Find-the-perimeter-of-the-triangle
	/* Extend to cover at least the unit circle */
	const float extend = (cos(M_PI / 4.0) + 1.0) * 2.0;
	/* Crappy diagram
	 * ex 1
	 *    | \
	 *    |   \
	 *  1 |     \
	 *    |       \
	 *    |         \
	 *  0 |     x     \
	 *    |   Circle    \
	 *    |   Origin      \
	 * -1 0 --------------- 2
	 *   -1     0     1     ex
	 **/
    // float(v_id / 2) 得到的结果依次是 (0, 0, 1)
	gl_Position.x = float(v_id / 2) * extend - 1.0; /* int divisor round down */
	gl_Position.y = float(v_id % 2) * extend - 1.0;
	gl_Position.z = 0.0;
	gl_Position.w = 1.0;

	/* Generate Triangle */
	particlecoord = gl_Position.xy;

    // gl_Position 需要的 [-1,1] 的值
	gl_Position.xy *= coc * texel_size;
	gl_Position.xy -= 1.0 - 0.5 * texel_size; /* NDC Bottom left */   // 这里 相当于 gl_Position.xy =  - 1 + 0.5 * texel_size + gl_Position.xy , 那就是从左下角-1 开始计算
	gl_Position.xy += (0.5 + vec2(texelco) * 2.0) * texel_size;         // 因为是 从 -1 开始的，所以 长度就是 是 2，值限制在 从 [-1, 1]，然后这里就是添加中心点的意思
}



void main()
{
	step_scatter();
}

```

*effect_dof_frag.glsl*
```
void main()
{
	step_scatter();
}

/* coordinate used for calculating radius et al set in geometry shader */
in vec2 particlecoord;
flat in vec4 color; 

/* accumulate color in the near/far blur buffers */
void step_scatter(void)
{
    // 在圆外的像素都剔除掉
	/* Early out */
	float dist_sqrd = dot(particlecoord, particlecoord);

	/* Circle Dof */
	if (dist_sqrd > 1.0) {
		discard;
	}

	/* Regular Polygon Dof */
	if (bokeh_sides > 0.0) {
		/* Circle parametrization */
		float theta = atan(particlecoord.y, particlecoord.x) + bokeh_rotation;
		float r;

		r = cos(M_PI / bokeh_sides) /
		    (cos(theta - (M_2PI / bokeh_sides) * floor((bokeh_sides * theta + M_PI) / M_2PI)));

		if (dist_sqrd > r * r) {
			discard;
		}
	}

	fragData0 = color;
}

```


### Resolve

#### 传入Shader参数
```

grp = DRW_shgroup_create(e_data.dof_resolve_sh, psl->dof_resolve);
DRW_shgroup_uniform_buffer(grp, "colorBuffer", &effects->source_buffer, 0);
DRW_shgroup_uniform_buffer(grp, "nearBuffer", &txl->dof_near_blur, 1);
DRW_shgroup_uniform_buffer(grp, "farBuffer", &txl->dof_far_blur, 2);
DRW_shgroup_uniform_buffer(grp, "depthBuffer", &dtxl->depth, 3);
DRW_shgroup_uniform_vec2(grp, "nearFar", effects->dof_near_far, 1);
DRW_shgroup_uniform_vec3(grp, "dofParams", effects->dof_params, 1);
DRW_shgroup_call_add(grp, quad, NULL);



```

#### shader

*effect_dof_vert.glsl*
```
void passthrough()
{
	uvcoord = uvs;
	gl_Position = vec4(pos, 0.0, 1.0);
}

void main()
{
    passthrough();
}
```


*effect_dof_frag.glsl*

```


#define MERGE_THRESHOLD 4.0

uniform sampler2D farBuffer;
uniform sampler2D nearBuffer;

/* Combine the Far and Near color buffers */
void step_resolve(void)
{
	/* Recompute Near / Far CoC */
	float depth = texture(depthBuffer, uvcoord).r;
	float zdepth = linear_depth(depth);
	float coc_signed = calculate_coc(zdepth);
	float coc_far = max(-coc_signed, 0.0);
	float coc_near = max(coc_signed, 0.0);

	/* Recompute Near / Far CoC */
	vec4 srccolor = texture(colorBuffer, uvcoord);
	vec4 farcolor = texture(farBuffer, uvcoord);
	vec4 nearcolor = texture(nearBuffer, uvcoord);

	float farweight = farcolor.a;
	if (farweight > 0.0)
		farcolor /= farweight;

	float mixfac = smoothstep(1.0, MERGE_THRESHOLD, coc_far);

	farweight = mix(1.0, farweight, mixfac);

	float nearweight = nearcolor.a;
	if (nearweight > 0.0) {
		nearcolor /= nearweight;
	}

	if (coc_near > 1.0) {
		fragData0 = nearcolor;
	}
	else {
		float totalweight = nearweight + farweight;
		vec4 finalcolor = mix(srccolor, farcolor, mixfac);
		fragData0 = mix(finalcolor, nearcolor, nearweight / totalweight);
	}
}

void main()
{
	step_resolve();

}
```