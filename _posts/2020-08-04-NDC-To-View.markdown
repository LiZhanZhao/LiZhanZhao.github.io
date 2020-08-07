---
layout:     post
title:      "NDC To View 记录"
subtitle:   ""
date:       2020-08-07 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Other
---




## NDC空间变换到View空间
### 理论方案
- P(xyz, 1) 经过矩阵P*V，可以变成 P1(NDC*w0, w0), 那么再除以w0可以得到P2(NDC, 1)
- P2(NDC, 1) 经过矩阵(P*V)^-1, 可以得到 P1(xyz/w0, 1/w0), 那么就只需要除以 1/w0，就可以得到 P(xyz, 1)

>
let's say your original position was (Pxyz, 1), after multiplication by P*V, that would give you (NDC*w0, w0)
the w divide then gives (NDC, 1).
multiplying (NDC, 1) by (P*V)^-1 gives you (Pxyz/w0, 1/w0).
So to get back to Pxyz, you divide Pxyz/w0 by 1/w0. That 1/w0 is what comes in tmp.w

### 验证理论
*Unity*
```
Camera cam = Camera.main;
        
Vector4 p = new Vector4(11, 12, 13, 1);

Matrix4x4 ProjectMat = GL.GetGPUProjectionMatrix(cam.projectionMatrix,false);
Vector4 clipPos = ProjectMat * p;
float w = clipPos.w;
Vector4 ndc = clipPos / clipPos.w;

Debug.Log("W : " + w);
Debug.Log("1/W : " + 1/w);
Debug.Log("NDC : " + ndc.ToString("f4"));


Vector4 ndc_2 = new Vector4(ndc.x, ndc.y, ndc.z, 1);
Matrix4x4 inverseVP = Matrix4x4.Inverse(ProjectMat);
Vector4 clipPos_2 = inverseVP * ndc_2;


Debug.Log("1/W : " + clipPos_2.w);

Vector4 p_2 = clipPos_2 / clipPos_2.w;
Debug.Log(p_2.ToString("f4"));
```

*结果*
![](/img/NDCToView/1.png)
>
- 从上面的结果可以看到理论是正确的。


### 例子
*vs*
```
// Calculate camera to far plane ray in vertex shader
float4 cameraRay = float4( vertex.uv * 2.0 - 1.0, 1.0, 1.0);
cameraRay = mul( _CameraInverseProjectionMatrix, cameraRay);
output.cameraRay = cameraRay.xyz / cameraRay.w;
```

*fs*
```
// Calculate camera space pixel position in fragment shader
float3 decodedNormal;
float decodedDepth;
DecodeDepthNormal( tex2D( _CameraDepthNormalsTexture, input.uv), decodedDepth, decodedNormal);
float3 pixelPosition = input.cameraRay * decodedDepth;
```

>
- 上面的例子基于后期，vs 中，屏幕空间 -> NDC空间 -> View空间，把远裁剪面的四个顶点都变换到view空间中
- 将这个方向向量传给frag函数，我们便可以得到一个插值后的结果，这个结果就是对应各个像素的镜头到远裁剪面的射线
- 将这个射线乘以depth，我们便可以得到像素对应在View空间中的位置