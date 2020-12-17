---
layout:     post
title:      "blender eevee Add Tonemapping Using Ocio"
subtitle:   ""
date:       2020-12-17 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/5/11  * evee: Add tonemapping using ocio. Actually it's done by the Draw Manager, so other engines can use it. <br>  

> SVN : 2017/4/5  MSVC 2015 windows x64 (vc140) Alembic 1.7.1



## 作用 
去掉Shader中得tonemapping代码，直接使用color manager 来进行后处理



## 编译

- 直接编译

## 效果

![](/img/Eevee/Tonemapping-OCIO/tonemapping-ocio.png)


## 渲染
*eevee_effects.c*
```
void EEVEE_draw_effects(EEVEE_Data *vedata)
{
	...
	/* Tonemapping */
	DRW_transform_to_display(effects->source_buffer);
}
```
>
- DRW_transform_to_display 会调用 ocio_impl_glsl.cc 中得 setupGLSLDraw 进行配置 Shader
- vs 使用  blender-git\blender\intern\opencolorio\gpu_shader_display_transform_vertex.glsl
- fs 使用  blender-git\blender\intern\opencolorio\gpu_shader_display_transform.glsl

## Shader

### vs

*gpu_shader_display_transform_vertex.glsl*
```

uniform mat4 ModelViewProjectionMatrix;

#if __VERSION__ == 120
  attribute vec2 texCoord;
  attribute vec2 pos;
  varying vec2 texCoord_interp;
#else
  in vec2 texCoord;
  in vec2 pos;
  out vec2 texCoord_interp;
#endif

void main()
{
	gl_Position = ModelViewProjectionMatrix * vec4(pos.xy, 0.0f, 1.0f);
	texCoord_interp = texCoord;
}

```

### fs

*gpu_shader_display_transform.glsl*

```
uniform sampler2D image_texture;
uniform sampler3D lut3d_texture;

#ifdef USE_DITHER
uniform float dither;
#endif

#ifdef USE_TEXTURE_SIZE
uniform float image_texture_width;
uniform float image_texture_height;
#endif

#if __VERSION__ < 130
  varying vec2 texCoord_interp;
  #define fragColor gl_FragColor
#else
  in vec2 texCoord_interp;
  out vec4 fragColor;
  #define texture2D texture
#endif

#ifdef USE_CURVE_MAPPING
/* Curve mapping parameters
 *
 * See documentation for OCIO_CurveMappingSettings to get fields descriptions.
 * (this ones pretyt much copies stuff from C structure.)
 */
uniform sampler1D curve_mapping_texture;
uniform int curve_mapping_lut_size;
uniform ivec4 use_curve_mapping_extend_extrapolate;
uniform vec4 curve_mapping_mintable;
uniform vec4 curve_mapping_range;
uniform vec4 curve_mapping_ext_in_x;
uniform vec4 curve_mapping_ext_in_y;
uniform vec4 curve_mapping_ext_out_x;
uniform vec4 curve_mapping_ext_out_y;
uniform vec4 curve_mapping_first_x;
uniform vec4 curve_mapping_first_y;
uniform vec4 curve_mapping_last_x;
uniform vec4 curve_mapping_last_y;
uniform vec3 curve_mapping_black;
uniform vec3 curve_mapping_bwmul;

float read_curve_mapping(int table, int index)
{
	/* TODO(sergey): Without -1 here image is getting darken after applying unite curve.
	 *               But is it actually correct to subtract 1 here?
	 */
	float texture_index = float(index) / float(curve_mapping_lut_size  - 1);
	return texture1D(curve_mapping_texture, texture_index)[table];
}

float curvemap_calc_extend(int table, float x, vec2 first, vec2 last)
{
	if (x <= first[0]) {
		if (use_curve_mapping_extend_extrapolate[table] == 0) {
			/* no extrapolate */
			return first[1];
		}
		else {
			if (curve_mapping_ext_in_x[table] == 0.0)
				return first[1] + curve_mapping_ext_in_y[table] * 10000.0;
			else
				return first[1] + curve_mapping_ext_in_y[table] * (x - first[0]) / curve_mapping_ext_in_x[table];
		}
	}
	else if (x >= last[0]) {
		if (use_curve_mapping_extend_extrapolate[table] == 0) {
			/* no extrapolate */
			return last[1];
		}
		else {
			if (curve_mapping_ext_out_x[table] == 0.0)
				return last[1] - curve_mapping_ext_out_y[table] * 10000.0;
			else
				return last[1] + curve_mapping_ext_out_y[table] * (x - last[0]) / curve_mapping_ext_out_x[table];
		}
	}
	return 0.0;
}

float curvemap_evaluateF(int table, float value)
{
	float mintable_ = curve_mapping_mintable[table];
	float range = curve_mapping_range[table];
	float mintable = 0.0;
	int CM_TABLE = curve_mapping_lut_size - 1;

	float fi;
	int i;

	/* index in table */
	fi = (value - mintable) * range;
	i = int(fi);

	/* fi is table float index and should check against table range i.e. [0.0 CM_TABLE] */
	if (fi < 0.0 || fi > float(CM_TABLE)) {
		return curvemap_calc_extend(table, value,
		                            vec2(curve_mapping_first_x[table], curve_mapping_first_y[table]),
		                            vec2(curve_mapping_last_x[table], curve_mapping_last_y[table]));
	}
	else {
		if (i < 0) return read_curve_mapping(table, 0);
		if (i >= CM_TABLE) return read_curve_mapping(table, CM_TABLE);

		fi = fi - float(i);
		return (1.0 - fi) * read_curve_mapping(table, i) + fi * read_curve_mapping(table, i + 1);
	}
}

vec4 curvemapping_evaluate_premulRGBF(vec4 col)
{
	vec4 result = col;
	result[0] = curvemap_evaluateF(0, (col[0] - curve_mapping_black[0]) * curve_mapping_bwmul[0]);
	result[1] = curvemap_evaluateF(1, (col[1] - curve_mapping_black[1]) * curve_mapping_bwmul[1]);
	result[2] = curvemap_evaluateF(2, (col[2] - curve_mapping_black[2]) * curve_mapping_bwmul[2]);
	result[3] = col[3];
	return result;
}
#endif

#ifdef USE_DITHER
float dither_random_value(vec2 co)
{
	return fract(sin(dot(co.xy, vec2(12.9898, 78.233))) * 43758.5453) * 0.005 * dither;
}

vec2 round_to_pixel(vec2 st)
{
	vec2 result;
#ifdef USE_TEXTURE_SIZE
	vec2 size = vec2(image_texture_width, image_texture_height);
#else
	vec2 size = textureSize(image_texture, 0);
#endif
	result.x = float(int(st.x * size.x)) / size.x;
	result.y = float(int(st.y * size.y)) / size.y;
	return result;
}

vec4 apply_dither(vec2 st, vec4 col)
{
	vec4 result;
	float random_value = dither_random_value(round_to_pixel(st));
	result.r = col.r + random_value;
	result.g = col.g + random_value;
	result.b = col.b + random_value;
	result.a = col.a;
	return result;
}
#endif

void main()
{
	vec4 col = texture2D(image_texture, texCoord_interp.st);
#ifdef USE_CURVE_MAPPING
	col = curvemapping_evaluate_premulRGBF(col);
#endif

#ifdef USE_PREDIVIDE
	if (col[3] > 0.0 && col[3] < 1.0) {
		float inv_alpha = 1.0 / col[3];
		col[0] *= inv_alpha;
		col[1] *= inv_alpha;
		col[2] *= inv_alpha;
	}
#endif

	/* NOTE: This is true we only do de-premul here and NO premul
	 *       and the reason is simple -- opengl is always configured
	 *       for straight alpha at this moment
	 */

	vec4 result = OCIODisplay(col, lut3d_texture);

#ifdef USE_DITHER
	result = apply_dither(texCoord_interp.st, result);
#endif

	fragColor = result;
}

```
>
- 看源码，会发现少了 OCIODisplay 函数，OCIODisplay 函数主要是在 setupGLSLDraw 函数中进行生成，可以参考
```
/* Step 1: Create a GPU Shader Description */
GpuShaderDesc shaderDesc;
shaderDesc.setLanguage(GPU_LANGUAGE_GLSL_1_3);
shaderDesc.setFunctionName("OCIODisplay");
shaderDesc.setLut3DEdgeLen(LUT3D_EDGE_SIZE);
...
os << ocio_processor->getGpuShaderText(shaderDesc) << "\n";
os << datatoc_gpu_shader_display_transform_glsl;
```
- 通过截帧代码可以发现
``` 
vec4 OCIODisplay(in vec4 inPixel, 
    const sampler3D lut3d) 
{
	vec4 out_pixel = inPixel; 
	out_pixel.rgb = max(vec3(1.17549e-38, 1.17549e-38, 1.17549e-38), vec3(1, 1, 1) * out_pixel.rgb + vec3(0, 0, 0));
	out_pixel.rgb = vec3(1.4427, 1.4427, 1.4427) * log(out_pixel.rgb) + vec3(0, 0, 0);
	out_pixel = vec4(0.047619, 0.047619, 0.047619, 1) * out_pixel;
	out_pixel = vec4(0.714286, 0.714286, 0.714286, 0) + out_pixel;
	out_pixel.rgb = texture3D(lut3d, 0.984375 * out_pixel.rgb + 0.0078125).rgb;
	return out_pixel;
}
```  
但是这个函数得来源可以参考 : [OpenColorIO](https://github.com/AcademySoftwareFoundation/OpenColorIO) 下的OpenColorIO-master\include\OpenColorIO\OpenColorIO.h 文件
如果不想折腾代码生成的话，就直接截帧，不同的参数会导致不同的 OCIODisplay 函数