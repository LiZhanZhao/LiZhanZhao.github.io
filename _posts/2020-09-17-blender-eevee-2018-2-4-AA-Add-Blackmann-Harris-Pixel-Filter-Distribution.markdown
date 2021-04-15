---
layout:     post
title:      "blender eevee AA Add Blackmann-Harris pixel filter distribution"
subtitle:   ""
date:       2021-4-15 20:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - blender
---

## 来源

- 主要看这个commit

> GIT : 2018/2/4  *   Eevee : AA Add Blackmann-Harris pixel filter distribution. <br> 

> 
This leads to a huge improvement of AntiAliasing quality.
There is no other distribution now and there is not settings displayed to the user. That's for another commit.


> SVN : 2017/9/23  [msvc] add support for jpeg2000 in oiio to resolve T52845 on windows. 


## 作用
改善抗锯齿

### eevee_temporal_sampling.c
```
#define FILTER_CDF_TABLE_SIZE 512

static struct {
	/* Temporal Anti Aliasing */
	struct GPUShader *taa_resolve_sh;

	/* Pixel filter table: Only blackman-harris for now. */
	float inverted_cdf[FILTER_CDF_TABLE_SIZE];
} e_data = {NULL}; /* Engine data */


static float blackman_harris(float x)
{
	/* Hardcoded 1px footprint [-0.5..0.5]. We resize later. */
	const float width = 1.0f;
	x = 2.0f * M_PI * (x / width + 0.5f);
	return 0.35875f - 0.48829f * cosf(x) + 0.14128f * cosf(2.0f * x) - 0.01168f * cosf(3.0f * x);
}

/* Compute cumulative distribution function of a discrete function. */
static void compute_cdf(float (*func)(float x), float cdf[FILTER_CDF_TABLE_SIZE])
{
	cdf[0] = 0.0f;
	/* Actual CDF evaluation. */
	for(int u = 0; u < FILTER_CDF_TABLE_SIZE - 1; ++u) {
		float x = (float)(u + 1) / (float)(FILTER_CDF_TABLE_SIZE - 1);
		cdf[u + 1] = cdf[u] + func(x - 0.5f); /* [-0.5..0.5]. We resize later. */
	}
	/* Normalize the CDF. */
	for(int u = 0; u < FILTER_CDF_TABLE_SIZE - 1; u++) {
		cdf[u] /= cdf[FILTER_CDF_TABLE_SIZE - 1];
	}
	/* Just to make sure. */
	cdf[FILTER_CDF_TABLE_SIZE - 1] = 1.0f;
}

static void invert_cdf(const float cdf[FILTER_CDF_TABLE_SIZE], float invert_cdf[FILTER_CDF_TABLE_SIZE])
{
	for (int u = 0; u < FILTER_CDF_TABLE_SIZE; u++) {
		float x = (float)u / (float)(FILTER_CDF_TABLE_SIZE - 1);
		for (int i = 0; i < FILTER_CDF_TABLE_SIZE; ++i) {
			if (cdf[i] >= x) {
				if (i == FILTER_CDF_TABLE_SIZE - 1) {
					invert_cdf[u] = 1.0f;
				}
				else {
					float t = (x - cdf[i])/(cdf[i+1] - cdf[i]);
					invert_cdf[u] = ((float)i + t) / (float)(FILTER_CDF_TABLE_SIZE - 1);
				}
				break;
			}
		}
	}
}

/* Evaluate a discrete function table with linear interpolation. */
static float eval_table(float *table, float x)
{
	CLAMP(x, 0.0f, 1.0f);
	x = x * (FILTER_CDF_TABLE_SIZE - 1);

	int index = min_ii((int)(x), FILTER_CDF_TABLE_SIZE - 1);
	int nindex = min_ii(index + 1, FILTER_CDF_TABLE_SIZE - 1);
	float t = x - index;

	return (1.0f - t) * table[index] + t * table[nindex];
}

static void eevee_create_cdf_table_temporal_sampling(void)
{
	float *cdf_table = MEM_mallocN(sizeof(float) * FILTER_CDF_TABLE_SIZE, "Eevee Filter CDF table");

	/* Use blackman-harris filter. */
	compute_cdf(blackman_harris, cdf_table);
	invert_cdf(cdf_table, e_data.inverted_cdf);

	MEM_freeN(cdf_table);
}

void EEVEE_temporal_sampling_matrices_calc(
        EEVEE_EffectsInfo *effects, float viewmat[4][4], float persmat[4][4], const double ht_point[2])
{
	const float *viewport_size = DRW_viewport_size_get();

	float filter_size = 1.5f; /* TODO Option. */
	filter_size *= 2.0f; /* Because of Blackmann Harris to gaussian width matching. */
	/* The cdf precomputing is giving distribution inside [0..1].
	 * We need to map it to [-1..1] THEN scale by the desired filter size. */
	filter_size *= 2.0f;
	float ofs_x = (eval_table(e_data.inverted_cdf, (float)(ht_point[0])) - 0.5f) * filter_size;
	float ofs_y = (eval_table(e_data.inverted_cdf, (float)(ht_point[1])) - 0.5f) * filter_size;

	window_translate_m4(
	        effects->overide_winmat, persmat,
	        ofs_x / viewport_size[0],
	        ofs_y / viewport_size[1]);

	mul_m4_m4m4(effects->overide_persmat, effects->overide_winmat, viewmat);
	invert_m4_m4(effects->overide_persinv, effects->overide_persmat);
	invert_m4_m4(effects->overide_wininv, effects->overide_winmat);
}


int EEVEE_temporal_sampling_init(EEVEE_ViewLayerData *UNUSED(sldata), EEVEE_Data *vedata)
{
	...
	if ((BKE_collection_engine_property_value_get_int(props, "taa_samples") != 1 &&
	    /* FIXME the motion blur camera evaluation is tagging view_updated
	     * thus making the TAA always reset and never stopping rendering. */
	    (effects->enabled_effects & EFFECT_MOTION_BLUR) == 0) ||
	    DRW_state_is_image_render())
	{
		const float *viewport_size = DRW_viewport_size_get();
		float persmat[4][4], viewmat[4][4];

		if (!e_data.taa_resolve_sh) {
			eevee_create_shader_temporal_sampling();
			eevee_create_cdf_table_temporal_sampling();  //<----------------------------
		}

		...

		EEVEE_temporal_sampling_matrices_calc(effects, viewmat, persmat, ht_point);
	}
	...
}
```
>
- 思路就是 eevee_create_cdf_table_temporal_sampling 计算 e_data.inverted_cdf 出来，
<br><br>
- 然后 EEVEE_temporal_sampling_matrices_calc 就利用 e_data.inverted_cdf 计算 effects->overide_persmat 等矩阵