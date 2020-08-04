---
layout:     post
title:      "Early Z Rejection 记录"
subtitle:   ""
date:       2020-08-04 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Other
---




## 资料
[官网文档](https://software.intel.com/content/www/us/en/develop/articles/early-z-rejection-sample.html) 

## 编译
以上的网址有下载的Demo, 然后可以用vs进行编译,vs2012 vs2010 vs2008 都可以

## 效果
*VisualizeOverdraw*
![](/img/EarlyZ/VisualizeOverdraw.png)
<br>

*VisualizeOverdraw+Sort*
![](/img/EarlyZ/VisualizeOverdraw+Sort.png)
<br>

*VisualizeOverdraw+ZPrepass*
![](/img/EarlyZ/VisualizeOverdraw+ZPrepass.png)

## 理解

### Shader

*VS.hlsl*
```hlsl

cbuffer cbPerObject : register(b0)
{
	matrix		g_mWorldViewProjection	: packoffset(c0);
	matrix		g_mWorld				: packoffset(c4);
	float4		g_vEyePos				: packoffset(c8);
};

// VS input
struct VS_INPUT
{
	float4 position		: POSITION;
	float3 normal		: NORMAL;
	float2 texcoord		: TEXCOORD0;
	float3 tangent		: TANGENT;
	float3 binormal		: BINORMAL;
};

struct VS_INPUT_Z_PASS
{
	float4 position		: POSITION;
};


// VS output
struct VS_OUTPUT
{
	float4 position		: SV_POSITION;
	float3 normal		: NORMAL;
	float2 texcoord		: TEXCOORD0;
	float3 tangent		: TANGENT;
	float3 binormal		: BINORMAL;
	float3 viewDir		: TEXCOORD1;
	float3 worldPos		: TEXCOORD2;
};

// Vertex shader
// This is the VS for the scene render. Transforms the vtx positions and prepares
// any other data needed by the PS
VS_OUTPUT VSMain(VS_INPUT Input)
{
	VS_OUTPUT Output;
	
	Output.position = mul(Input.position, g_mWorldViewProjection);
	Output.normal = mul(Input.normal, (float3x3)g_mWorld);
	Output.texcoord = Input.texcoord;
	Output.tangent	= mul(Input.tangent, (float3x3)g_mWorld);
	Output.binormal = mul(Input.binormal, (float3x3)g_mWorld);
		
	float4 worldPos;
	worldPos = mul(Input.position, g_mWorld);

    // Determine the viewing direction based on the position of the camera and the position of the vertex in the world.
    Output.viewDir = g_vEyePos.xyz - worldPos.xyz;
	Output.worldPos = worldPos.xyz;
	return Output;
}

// Vertex shader
// This is the VS for the z pre pass. Its only job is to transform the vtx
// position. Since there is no pixel shader, there is no need for any other
// information to be received as input or sent as output.
float4 VSMainZPass(VS_INPUT_Z_PASS Input) : SV_POSITION
{
	return mul(Input.position, g_mWorldViewProjection);
}

```
>
- 看shader代码可以看到，vs中有VSMain 和 VSMainZPass，注释可以看到其作用

*PS.hlsl* 
```hlsl

#define MAX_LIGHTS 16

cbuffer cbPerFrame : register( b0 )
{
	float4		g_vLightPositions[MAX_LIGHTS];
	float4		g_lightParams[MAX_LIGHTS];
	float4		g_vLightDirections[MAX_LIGHTS];
	float4		g_vLightColors[MAX_LIGHTS];
	int			g_numLights;
	float3		_reservedb0_0;
	float4		g_vDefaultLightDir;
	float4		g_vDefaultLightColor;
};

cbuffer cbPerObject : register( b1 )
{
	float4		g_vDiffuseColor;
	float4		g_vSpecularColor;
	float		g_fSpecularPow;
	float3		_reservedb1_0;
};

// Textures and Samplers
Texture2D		g_txDiffuse			: register( t0 );
Texture2D		g_txNormal			: register( t1 );
Texture2D		g_txSpecular		: register( t2 );
Texture2D		g_txAmbient			: register( t3 );

// Sampler state
SamplerState	g_samLinear			: register( s0 );

// PS input
struct PS_INPUT
{
	float4 position		: SV_POSITION;
	float3 normal		: NORMAL;
	float2 texcoord		: TEXCOORD0;
	float3 tangent		: TANGENT;
	float3 binormal		: BINORMAL;
	float3 viewDir		: TEXCOORD1;
	float3 worldPos		: TEXCOORD2;
};

struct PS_INPUT_OVERDRAW
{
	float4 position		: SV_POSITION;
};

#ifndef AMBIENT_MAP 
#define AMBIENT_MAP 0
#endif
#ifndef SPECULAR_MAP
#define SPECULAR_MAP 0
#endif

// Pixel shader to support normal mapping, specular mapping, and ambient occlusion mapping
// Shader is compiled multiple times so the proper version is used. For example, some objects
// do not require an ambient occlusion map.
float4 PSMain( PS_INPUT Input ) : SV_TARGET
{
	float4 normal = 2.0f * g_txNormal.Sample( g_samLinear, Input.texcoord ) - 1.0f;
	normal.y = -normal.y;
	float4 texDiffuse = g_txDiffuse.Sample( g_samLinear, Input.texcoord );
	float ambient = 1.0f;
	
	if(AMBIENT_MAP == 1)
	{
		ambient = g_txAmbient.Sample( g_samLinear, Input.texcoord).r;
	}
    float3 bump = normalize(Input.normal + normal.x * Input.tangent + normal.y * Input.binormal); 
	float4 texSpecular = 1.0f;

	if(SPECULAR_MAP == 1)
	{
		texSpecular = g_txSpecular.Sample( g_samLinear, Input.texcoord );
	}

	float4 diffuseColor = (float4)0.0f;
	float4 specularColor = (float4)0.0f;
	float4 litValue = (float4)0.0f;
	float3 view = normalize(Input.viewDir);

	float ndotl = dot(bump, -g_vDefaultLightDir.xyz);
	float ndoth = dot(bump, normalize(view - g_vDefaultLightDir.xyz));
	litValue = lit(ndotl, ndoth, g_fSpecularPow);
	diffuseColor += litValue.g * g_vDefaultLightColor;
	specularColor += litValue.b * g_vDefaultLightColor;

	[unroll]for(int i = 0; i < g_numLights; i++)
	{
	    float3 lightDir = normalize(g_vLightPositions[i].xyz - Input.worldPos);
		float ndotl = dot(bump, lightDir);
		float ndoth = dot(bump, normalize(view + lightDir));
		litValue = lit(ndotl, ndoth, g_fSpecularPow);
		float lightAngle = saturate(dot(-lightDir.xyz, g_vLightDirections[i].xyz));
		float intensity = smoothstep(g_lightParams[i].y, g_lightParams[i].r, lightAngle);
		
		diffuseColor += litValue.g * g_vLightColors[i] * intensity;
		specularColor += litValue.b * g_vLightColors[i] * intensity;
	}

	return float4(saturate(diffuseColor * texDiffuse + specularColor * texSpecular).rgb * ambient, 1.0f);
}

// Pixel Shader for rendering overdraw
float4 PSOverdraw( PS_INPUT_OVERDRAW Input ) : SV_TARGET
{
	return float4(1.0, 1.0f, 1.0f, 0.1f);
}

```
>
- 看shader代码可以看到，ps中有PSMain和 PSOverdraw，注释可以看到其作用


### VisualizeOverdraw 渲染
*Early Z Rejection.cpp*

```c
// Create blend states
CD3D11_BLEND_DESC bsDesc;//  = CD3D11_BLEND_DESC(D3D11_DEFAULT);

bsDesc.AlphaToCoverageEnable = FALSE;
bsDesc.IndependentBlendEnable = FALSE;
bsDesc.RenderTarget[0].BlendEnable = TRUE;
bsDesc.RenderTarget[0].BlendOp = D3D11_BLEND_OP_ADD;
bsDesc.RenderTarget[0].BlendOpAlpha = D3D11_BLEND_OP_ADD;
bsDesc.RenderTarget[0].DestBlend = D3D11_BLEND_INV_SRC_ALPHA;
bsDesc.RenderTarget[0].DestBlendAlpha = D3D11_BLEND_INV_SRC_ALPHA;
bsDesc.RenderTarget[0].RenderTargetWriteMask = D3D11_COLOR_WRITE_ENABLE_ALL;
bsDesc.RenderTarget[0].SrcBlend = D3D11_BLEND_SRC_ALPHA;
bsDesc.RenderTarget[0].SrcBlendAlpha = D3D11_BLEND_SRC_ALPHA;

pD3DDevice->CreateBlendState(&bsDesc, &g_pAlphaState);

...

// Pixel shader for rendering overdraw
V_RETURN( CompileShaderFromFile( L"PS.hlsl", L"PS", L"HLSL", "PSOverdraw",
                                    "ps_4_0", NULL, &pPSBlob ) );
hr = pD3DDevice->CreatePixelShader( pPSBlob->GetBufferPointer(), pPSBlob->GetBufferSize(),
                                    NULL, &g_pPixelShaderOverdraw );
```

```c
if(overdraw)
{
    // Visualizing overdraw can use the same simplified VS as the z only pass. The
    // difference is that there is now a PS specified.
    // VS
    V( pD3DImmediateContext->Map( g_pCBVSPerObjectZOnly, 0, D3D11_MAP_WRITE_DISCARD, 0, &MappedResource ) );
    CB_VS_PER_OBJECT_Z_ONLY* pVSPerObjectZOnly = ( CB_VS_PER_OBJECT_Z_ONLY* )MappedResource.pData;

    D3DXMatrixTranspose( &pVSPerObjectZOnly->g_mWorldViewProj, &mWorldViewProjection );

    pD3DImmediateContext->Unmap( g_pCBVSPerObjectZOnly, 0 );
    pD3DImmediateContext->VSSetConstantBuffers( g_uCBVSPerObjectBind, 1, &g_pCBVSPerObjectZOnly );
    pD3DImmediateContext->VSSetShader( g_pVertexShaderZPass, NULL, 0 );

    // Enable alpha blending
    float blendfactor[] = {1.0f, 1.0f, 1.0f, 1.0f};
    unsigned int samplemask = 0xffffffff;
    pD3DImmediateContext->OMSetBlendState(g_pAlphaState, blendfactor, samplemask);

    pD3DImmediateContext->PSSetShader(g_pPixelShaderOverdraw, NULL, 0);
}
```
>
- VisualizeOverdraw 渲染 vs 使用 VSMainZPass
- VisualizeOverdraw 渲染 fs 使用  PSOverdraw
- 设置 blender模式，利用透明来区别是否有重复渲染




### Sort 近到远排序

```c
  // Update the scene
void CALLBACK OnFrameMove( double fTime,
                           float fElapsedTime,
                           void* pUserContext )
{
    g_Camera.FrameMove( fElapsedTime );

	unsigned int numModels = g_sceneDescription.GetNumModels();
	for(unsigned int i = 0; i < numModels; i++)
		g_pModelOrder[i] = i;

	if(g_sort)
	{
		D3DXVECTOR3 eye = *g_Camera.GetEyePt();
		SortModels(g_pModelOrder, numModels, &eye);
	}
}

void SortModels(int* pModelOrder, int numModels, D3DXVECTOR3* pEye)
{
	float* pModelDistance = new float[numModels];
	for(int i = 0; i < numModels; i++)
	{
		D3DXVECTOR4 wp4;
		D3DXVec3Transform(&wp4, &g_pModels[i].center, &g_pModels[i].world);
		D3DXVECTOR3 wp = D3DXVECTOR3(wp4.x, wp4.y, wp4.z);
		D3DXVECTOR3 d = wp - *pEye;
		pModelDistance[i] = D3DXVec3Length(&d);
	}
	for(int i = 0; i < numModels; i++)
	{
		for(int j = 0; j < numModels - (i+1); j++)
		{
			if(pModelDistance[j] > pModelDistance[j+1])
			{
				float d = pModelDistance[j+1];
				int n = pModelOrder[j+1];
				pModelDistance[j+1] = pModelDistance[j];
				pModelDistance[j] = d;
				pModelOrder[j+1] = pModelOrder[j];
				pModelOrder[j] = n;
			}
		}
	}
	delete pModelDistance;

}
```
>
- 先渲染近的，再渲染远的物体


### ZPrepass

```c
// If this is the z-only pass then there is no need to set a render target, only a depth buffer
pD3DImmediateContext->OMSetRenderTargets(0, NULL, pDSV);
RenderScene(pD3DDevice, pD3DImmediateContext, fTime, fElapsedTime, true, false, pUserContext);

// Next, the scene is rendered to the color buffer. Since the depth values have already been
// written, there is no need to write again to the depth buffer so that can be disabled.
pD3DImmediateContext->OMSetDepthStencilState(g_pDepthStencilLessEqual, 0);
pD3DImmediateContext->OMSetRenderTargets(1, &pRTV, pDSV);

RenderScene(pD3DDevice, pD3DImmediateContext, fTime, fElapsedTime, false, g_VisualizeOverdraw, pUserContext);

pD3DImmediateContext->OMSetDepthStencilState(g_pRenderDSState, 0);

```
>
- 第一个 RenderScene 也就是第一个pass 只写深度缓存，没有写颜色缓存。
- 第二个 RenderScene 也就是第二个pass 不写深度缓存，开启深度测试，写颜色缓存。

## 注意
>
- 这个Early Z 处理的是不透明物体
