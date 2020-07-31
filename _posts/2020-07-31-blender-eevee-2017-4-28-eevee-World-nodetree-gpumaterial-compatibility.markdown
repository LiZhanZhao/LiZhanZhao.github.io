---
layout:     post
title:      "blender eevee World nodetree gpumaterial compatibility"
subtitle:   ""
date:       2020-07-31 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> 2017/4/28   * Eevee: World nodetree gpumaterial compatibility.
- Unify GPUMaterial creation (world/mesh).
- Support for multiple shader variations (not used for now).
- Convert GPUInputs to DRWUniforms to be used with the draw manager.
- Nodetree Update is not supported. The only way to refresh the shaders is to change render engine.
- Cleanup in GPUPass.
- Add new temporary Node Compatibility type. Compatibility types should be removed in the future.


## 作用 
*渲染环境贴图可以使用nodetree*


## 编译

- 直接编译


## 效果展示
在 *Node Editor* 中选择 World 分页 和 打勾 *Use Nodes*
![](/img/Eevee/WorldNodetree/1.png)

在 *Node Editor* 连接node tree, 3Dview中显示效果，下面只是简单地连接一个Shader用来渲染环境贴图
![](/img/Eevee/WorldNodetree/2.png)


## CPU

### 渲染
*eevee_engine.c*
```c

if (!e_data.frag_shader_lib) {
	DynStr *ds_frag = BLI_dynstr_new();
	BLI_dynstr_append(ds_frag, datatoc_bsdf_common_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_ltc_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_bsdf_direct_lib_glsl);
	BLI_dynstr_append(ds_frag, datatoc_lit_surface_frag_glsl);
	e_data.frag_shader_lib = BLI_dynstr_get_cstring(ds_frag);
	BLI_dynstr_free(ds_frag);
}

{
	psl->probe_background = DRW_pass_create("Probe Background Pass", DRW_STATE_WRITE_DEPTH | DRW_STATE_WRITE_COLOR);

	struct Batch *geom = DRW_cache_fullscreen_quad_get();
	DRWShadingGroup *grp = NULL;

	const DRWContextState *draw_ctx = DRW_context_state_get();
	Scene *scene = draw_ctx->scene;
	World *wo = scene->world;

	float *col = ts.colorBackground;
	if (wo) {
		col = &wo->horr;
	}

	if (wo && wo->use_nodes && wo->nodetree) {
		struct GPUMaterial *gpumat = GPU_material_from_nodetree(
			wo->nodetree, &wo->gpumaterial, &DRW_engine_viewport_eevee_type, 0,
			datatoc_probe_vert_glsl, datatoc_probe_geom_glsl, e_data.frag_shader_lib,
			"#define PROBE_CAPTURE\n"
			"#define MAX_LIGHT 128\n"
			"#define MAX_SHADOW_CUBE 42\n"
			"#define MAX_SHADOW_MAP 64\n"
			"#define MAX_SHADOW_CASCADE 8\n"
			"#define MAX_CASCADE_NUM 4\n");

		grp = DRW_shgroup_material_instance_create(gpumat, psl->probe_background, geom);

		if (grp) {
			DRW_shgroup_uniform_int(grp, "Layer", &zero, 1);

			for (int i = 0; i < 6; ++i)
				DRW_shgroup_call_dynamic_add_empty(grp);
		}
		else {
			/* Shader failed : pink background */
			static float pink[3] = {1.0f, 0.0f, 1.0f};
			col = pink;
		}
	}

	/* Fallback if shader fails or if not using nodetree. */
	if (grp == NULL) {
		grp = eevee_cube_shgroup(e_data.default_world, psl->probe_background, geom);
		DRW_shgroup_uniform_vec3(grp, "color", col, 1);
		DRW_shgroup_uniform_int(grp, "Layer", &zero, 1);
	}
}

```

*eevee_probes.c*
```c
void EEVEE_refresh_probe(EEVEE_Data *vedata)
{
	...
	DRW_framebuffer_bind(fbl->probe_fb);
	DRW_draw_pass(psl->probe_background);
	...
}
```

>
- 因为 psl->probe_background 是用来渲染 环境贴图, 可以参考之前的 [blender eevee World default shader](http://shaderstore.cn/2020/07/28/blender-eevee-2017-4-26-eevee-World-default-shader/)
- 上面的代码就分情况来创建 psl->probe_background， 如果不用nodetree的话，就是直接像 [blender eevee World default shader](http://shaderstore.cn/2020/07/28/blender-eevee-2017-4-26-eevee-World-default-shader/) 一样，直接使用一个颜色来作为环境贴图，
如果是使用nodetree的话，就选自己去连nodetree,这个nodetree生成shader代码来进行渲染环境贴图。



## GPU_material_from_nodetree 生成shader代码
```c

GPUPass *GPU_generate_pass_new(ListBase *nodes, struct GPUNodeLink *frag_outlink,
                               const char *vert_code, const char *geom_code,
                               const char *frag_lib, const char *defines)
{
	GPUShader *shader;
	GPUPass *pass;
	char *vertexgen, *geometrygen, *fragmentgen, *tmp;
	char *vertexcode, *geometrycode, *fragmentcode;

	/* prune unused nodes */
	gpu_nodes_prune(nodes, frag_outlink);

	/* generate code and compile with opengl */
	fragmentgen = code_generate_fragment(nodes, frag_outlink->output);
	// vertexgen = code_generate_vertex(nodes, GPU_MATERIAL_TYPE_MESH);
	// geometrygen = code_generate_geometry(nodes, false);
	UNUSED_VARS(vertexgen, geometrygen);

	tmp = BLI_strdupcat(frag_lib, glsl_material_library);
	fragmentcode = BLI_strdupcat(tmp, fragmentgen);
	vertexcode = BLI_strdup(vert_code);
	geometrycode = BLI_strdup(geom_code);

	shader = GPU_shader_create(vertexcode,
	                           fragmentcode,
	                           geometrycode,
	                           NULL,
	                           defines);

	MEM_freeN(tmp);

	/* failed? */
	if (!shader) {
		if (fragmentcode)
			MEM_freeN(fragmentcode);
		if (vertexcode)
			MEM_freeN(vertexcode);
		if (geometrycode)
			MEM_freeN(geometrycode);
		MEM_freeN(fragmentgen);
		gpu_nodes_free(nodes);
		return NULL;
	}

	/* create pass */
	pass = MEM_callocN(sizeof(GPUPass), "GPUPass");
	pass->shader = shader;
	pass->fragmentcode = fragmentcode;
	pass->geometrycode = geometrycode;
	pass->vertexcode = vertexcode;
	pass->libcode = glsl_material_library;

	/* extract dynamic inputs and throw away nodes */
	gpu_nodes_extract_dynamic_inputs_new(pass, nodes);
	gpu_nodes_free(nodes);

	MEM_freeN(fragmentgen);

	return pass;
}
```
>
- *gpu_material.c* 中 GPU_material_from_nodetree 调用 GPU_generate_pass_new
- *gpu_codegen,c* 中 GPU_generate_pass_new 主要负责生成shader代码
- vertexcode = BLI_strdup(vert_code); 这个是外部直接传入的 vs 代码
- geometrycode = BLI_strdup(geom_code); 这个是外部直接传入的 gs 代码
- fragmentgen = code_generate_fragment(nodes, frag_outlink->output); 这个是根据node的连接生成的fs的核心代码。
- frag_lib 由  bsdf_common_lib.glsl, ltc_lib.glsl, bsdf_direct_lib.glsl, lit_surface_frag.glsl 组成
- tmp = BLI_strdupcat(frag_lib, glsl_material_library); 这个就是把 外部传入的 frag_lib 和 *blender/source/blender/gpu/shaders/gpu_shader_material.glsl* 拼接在一起
- fragmentcode = BLI_strdupcat(tmp, fragmentgen); 最后组成真正的 fs 代码


## code_generate_fragment 生成frament代码

*gpu_codegen.c 文件中的 code_generate_fragment 函数*
```c
static char *code_generate_fragment(ListBase *nodes, GPUOutput *output)
{
	DynStr *ds = BLI_dynstr_new();
	char *code;
	int builtins;

#ifdef WITH_OPENSUBDIV
	GPUNode *node;
	GPUInput *input;
#endif


#if 0
	BLI_dynstr_append(ds, FUNCTION_PROTOTYPES);
#endif

	codegen_set_unique_ids(nodes);
	builtins = codegen_print_uniforms_functions(ds, nodes);

#if 0
	if (G.debug & G_DEBUG)
		BLI_dynstr_appendf(ds, "/* %s */\n", name);
#endif

	BLI_dynstr_append(ds, "void main()\n{\n");

	if (builtins & GPU_VIEW_NORMAL)
		BLI_dynstr_append(ds, "\tvec3 facingnormal = gl_FrontFacing? varnormal: -varnormal;\n");

	/* Calculate tangent space. */
#ifdef WITH_OPENSUBDIV
	{
		bool has_tangent = false;
		for (node = nodes->first; node; node = node->next) {
			for (input = node->inputs.first; input; input = input->next) {
				if (input->source == GPU_SOURCE_ATTRIB && input->attribfirst) {
					if (input->attribtype == CD_TANGENT) {
						BLI_dynstr_appendf(ds, "#ifdef USE_OPENSUBDIV\n");
						BLI_dynstr_appendf(ds, "\t%s var%d;\n",
						                   GPU_DATATYPE_STR[input->type],
						                   input->attribid);
						if (has_tangent == false) {
							BLI_dynstr_appendf(ds, "\tvec3 Q1 = dFdx(inpt.v.position.xyz);\n");
							BLI_dynstr_appendf(ds, "\tvec3 Q2 = dFdy(inpt.v.position.xyz);\n");
							BLI_dynstr_appendf(ds, "\tvec2 st1 = dFdx(inpt.v.uv);\n");
							BLI_dynstr_appendf(ds, "\tvec2 st2 = dFdy(inpt.v.uv);\n");
							BLI_dynstr_appendf(ds, "\tvec3 T = normalize(Q1 * st2.t - Q2 * st1.t);\n");
						}
						BLI_dynstr_appendf(ds, "\tvar%d = vec4(T, 1.0);\n", input->attribid);
						BLI_dynstr_appendf(ds, "#endif\n");
					}
				}
			}
		}
	}
#endif

	codegen_declare_tmps(ds, nodes);
	codegen_call_functions(ds, nodes, output);

	BLI_dynstr_append(ds, "}\n");

	/* create shader */
	code = BLI_dynstr_get_cstring(ds);
	BLI_dynstr_free(ds);

#if 0
	if (G.debug & G_DEBUG) printf("%s\n", code);
#endif

	return code;
}
```
>
- 可以直接在这个函数进行打印生成的 fs 核心代码 ，例如：

<br>

```glsl
const vec4 cons1 = vec4(0.050876088440, 0.050876088440, 0.050876088440, 1.000000000000);
const float cons2 = float(1.000000000000);
in vec3 varnormal;
const vec4 cons6 = vec4(0.000000000000, 0.000000000000, 0.000000000000, 1.000000000000);

void main()
{
	vec3 facingnormal = gl_FrontFacing? varnormal: -varnormal;
	vec4 tmp4;
	vec4 tmp7;

	node_background(cons1, cons2, facingnormal, tmp4);
	node_output_world(tmp4, cons6, tmp7);

	fragColor = tmp7;
}

```
>
- 在这个函数生成fs的核心代码实例


## 总结 nodetree 渲染环境贴图 使用到的shader文件
*vs*
- probe_vert.glsl

*gs*
- probe_geom.glsl

*fs*
- bsdf_common_lib.glsl
- ltc_lib.glsl
- bsdf_direct_lib.glsl
- lit_surface_frag.glsl
- *blender/source/blender/gpu/shaders/gpu_shader_material.glsl*
- *gpu_codegen.c 文件中的 code_generate_fragment 函数* 自动生成的代码