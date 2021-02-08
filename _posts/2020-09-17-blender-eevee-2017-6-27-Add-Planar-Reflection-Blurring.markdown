---
layout:     post
title:      "blender eevee Add Planar reflection Blurring"
subtitle:   ""
date:       2021-2-8 18:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2017/6/27  * Eevee :  Add Planar reflection blurring.

> This method is very cheap and inaccurate. This will fill the gap untill better model is supported.

> SVN : 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51
		
## 效果
![](/img/Eevee/PlanarReflections/4.png)

## 作用 
镜面反射模糊效果

## 编译
- 重新生成SLN
- git 定位到  2017/6/27  * Eevee : Add Planar reflection blurring .<br> 
- svn 定位到 2017/6/8  [MSVC/2013/2015/x86/x64] Update OpenCollada to 1.6.51    (单号:61894)

## 渲染前
*eevee_lightprobe.c*
```
void EEVEE_lightprobes_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *UNUSED(vedata))
{
	...
	e_data.probe_planar_downsample_sh = DRW_shader_create(
		        datatoc_lightprobe_planar_downsample_vert_glsl,
		        datatoc_lightprobe_planar_downsample_geom_glsl,
		        datatoc_lightprobe_planar_downsample_frag_glsl,
		        NULL);
	...
}


void EEVEE_lightprobes_cache_init(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	{
		psl->probe_planar_downsample_ps = DRW_pass_create("LightProbe Planar Downsample", DRW_STATE_WRITE_COLOR);

		struct Gwn_Batch *geom = DRW_cache_fullscreen_quad_get();
		DRWShadingGroup *grp = stl->g_data->planar_downsample = DRW_shgroup_instance_create(e_data.probe_planar_downsample_sh, psl->probe_planar_downsample_ps, geom);
		DRW_shgroup_uniform_buffer(grp, "source", &txl->planar_pool);
		DRW_shgroup_uniform_vec2(grp, "texelSize", stl->g_data->texel_size, 1);
	}
	...
}
```
>
- psl->probe_planar_downsample_ps 由 lightprobe_planar_downsample_vert.glsl, lightprobe_planar_downsample_geom.glsl, lightprobe_planar_downsample_frag.glsl 组成


## 渲染
*eevee_lightprobe.c*
```
void EEVEE_lightprobes_cache_finish(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
	/* Setup planar filtering pass */
	DRW_shgroup_set_instance_count(stl->g_data->planar_downsample, pinfo->num_planar);
	...
}


static void downsample_planar(void *vedata, int level)
{
	EEVEE_PassList *psl = ((EEVEE_Data *)vedata)->psl;
	EEVEE_StorageList *stl = ((EEVEE_Data *)vedata)->stl;

	const float *size = DRW_viewport_size_get();
	copy_v2_v2(stl->g_data->texel_size, size);
	for (int i = 0; i < level - 1; ++i) {
		stl->g_data->texel_size[0] /= 2.0f;
		stl->g_data->texel_size[1] /= 2.0f;
		min_ff(floorf(stl->g_data->texel_size[0]), 1.0f);
		min_ff(floorf(stl->g_data->texel_size[1]), 1.0f);
	}
	invert_v2(stl->g_data->texel_size);

	DRW_draw_pass(psl->probe_planar_downsample_ps);
}

void EEVEE_lightprobes_refresh(EEVEE_SceneLayerData *sldata, EEVEE_Data *vedata)
{
	...
update_planar:

	for (int i = 0; (ob = pinfo->probes_planar_ref[i]) && (i < MAX_PLANAR); i++) {
		EEVEE_LightProbeEngineData *ped = EEVEE_lightprobe_data_get(ob);

		if (ped->need_update) {
			/* Temporary Remove all planar reflections (avoid lag effect). */
			int tmp_num_planar = pinfo->num_planar;
			pinfo->num_planar = 0;

			render_scene_to_planar(vedata, i, ped->viewmat, ped->persmat, ped->planer_eq_offset);

			/* Restore */
			pinfo->num_planar = tmp_num_planar;

			ped->need_update = false;
			ped->probe_id = i;
		}
	}

	/* If there is at least one planar probe */
	if (pinfo->num_planar > 0) {
		DRW_framebuffer_recursive_downsample(vedata->fbl->minmaxz_fb, txl->planar_pool, 5, &downsample_planar, vedata);
	}
}
```
>
- 这里当把场景的物体渲染到 planar_pool RT 的时候, 再使用downsample_planar函数渲染 planar_pool RT 的 mipmap。
- 这里需要主要的是 DRW_shgroup_set_instance_count(stl->g_data->planar_downsample, pinfo->num_planar); 是使用了 instance，shader 中  gl_InstanceID 有数值。
- downsample_planar 其实就是 就是调用 probe_planar_downsample_ps 来instance渲染 planar_pool 这个 sampler2DArray 的 所有元素的mipmap。

## Shader (probe_planar_downsample_ps)

*lightprobe_planar_downsample_vert.glsl*
```
in vec2 pos;

out int instance;
out vec2 vPos;

void main() {
	instance = gl_InstanceID;
	vPos = pos;
}
```
<br><br>

*lightprobe_planar_downsample_geom.glsl*
```

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

in int instance[];
in vec2 vPos[];

flat out float layer;

void main() {
	gl_Layer = instance[0];
	layer = float(instance[0]);

	gl_Position = vec4(vPos[0], 0.0, 0.0);
	EmitVertex();

	gl_Position = vec4(vPos[1], 0.0, 0.0);
	EmitVertex();

	gl_Position = vec4(vPos[2], 0.0, 0.0);
	EmitVertex();

	EndPrimitive();
}
```
<br><br>

*lightprobe_planar_downsample_frag.glsl*
```
/**
 * Simple downsample shader. Takes the average of the 4 texels of lower mip.
 **/

uniform sampler2DArray source;
uniform vec2 texelSize;

in vec2 uvs;
flat in float layer;

out vec4 FragColor;

void main()
{
	/* Reconstructing Target uvs like this avoid missing pixels */
	vec2 uvs = floor(gl_FragCoord.xy) * 2.0 * texelSize + texelSize;

	/* Downsample with a 4x4 box filter */
	vec4 d = texelSize.xyxy * vec4(-1, -1, +1, +1);

	FragColor  = texture(source, vec3(uvs + d.xy, layer)).rgba;
	FragColor += texture(source, vec3(uvs + d.zy, layer)).rgba;
	FragColor += texture(source, vec3(uvs + d.xw, layer)).rgba;
	FragColor += texture(source, vec3(uvs + d.zw, layer)).rgba;

	FragColor /= 4.0;
}
```
>
- 以上的代码就是加权平均进行模糊RT



## Shader (lit_surface_frag)

*lit_surface_frag.glsl*
```
vec3 eevee_surface_lit(vec3 world_normal, vec3 albedo, vec3 f0, float roughness, float ao)
{
	...
	/* Planar Reflections */
	for (int i = 0; i < MAX_PLANAR && i < planar_count && spec_accum.a < 0.999; ++i) {
		PlanarData pd = planars_data[i];

		float influence = planar_attenuation(sd.W, sd.N, pd);

		if (influence > 0.0) {
			float influ_spec = min(influence, (1.0 - spec_accum.a));

			/* Sample reflection depth. */
			vec4 refco = pd.reflectionmat * vec4(sd.W, 1.0);
			refco.xy /= refco.w;
			float ref_depth = textureLod(probePlanars, vec3(refco.xy, i), 0.0).a;

			/* Find view vector / reflection plane intersection. (dist_to_plane is negative) */
			float dist_to_plane = line_plane_intersect_dist(cameraPos, sd.V, pd.pl_plane_eq);
			vec3 point_on_plane = cameraPos + sd.V * dist_to_plane;

			/* How far the pixel is from the plane. */
			ref_depth = ref_depth + dist_to_plane;

			/* Compute distorded reflection vector based on the distance to the reflected object.
			 * In other words find intersection between reflection vector and the sphere center
			 * around point_on_plane. */
			vec3 proj_ref = reflect(R * ref_depth, pd.pl_normal);

			/* Final point in world space. */
			vec3 ref_pos = point_on_plane + proj_ref;

			/* Reproject to find texture coords. */
			refco = pd.reflectionmat * vec4(ref_pos, 1.0);
			refco.xy /= refco.w;

			/* Distance to roughness */ ------------------------------------
			float linear_roughness = sqrt(roughness);
			float distance_roughness = min(linear_roughness, ref_depth * linear_roughness);
			linear_roughness = mix(distance_roughness, linear_roughness, linear_roughness);

			/* Decrease influence for high roughness */
			influ_spec *= saturate((1.0 - linear_roughness) * 5.0 - 2.0);

			float lod = linear_roughness * 2.5 * 5.0;
			vec3 sample = textureLod(probePlanars, vec3(refco.xy, i), lod).rgb;

			/* Use a second sample randomly rotated to blur out the lowres aspect */
			vec2 rot_sample = (1.0 / vec2(textureSize(probePlanars, 0).xy)) * vec2(cos(rand.a * M_2PI), sin(rand.a * M_2PI)) * lod;
			sample += textureLod(probePlanars, vec3(refco.xy + rot_sample, i), lod).rgb;
			sample *= 0.5;
			// -------------------------------------------------

			spec_accum.rgb += sample * influ_spec;
			spec_accum.a += influ_spec;
		}
	}
	...
}
```
>
- 主要的代码在 //--------------------- 
- 思路就是根据 roughness 采样 probePlanars mipmap 层，probePlanars mipmap是经过模糊的，所以就会有planar Reflection Blurring 效果。