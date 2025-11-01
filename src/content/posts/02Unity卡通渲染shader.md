---
title: 卡通渲染shader
published: 2025-10-15
description: 使用shader实现卡通渲染。"
image: "https://typorytp.oss-cn-hangzhou.aliyuncs.com/titlekatong01.png"
tags: ["Shader"]
category: Unity Shader
draft: false
---


# Unity卡通渲染shader

前言：卡通渲染是一个复合的渲染效果，主要包括边缘线绘制，色阶阴影，面部阴影，边缘光，高光反射等复合而成。

## 边缘线绘制

边缘线绘制的总体思路是将模型正面剔除，将模型的顶点在法线方向上膨胀一段距离，因为正面剔除的缘故，所以本该包裹在模型正面的边缘线就不会显示，边缘线只会绘制在背面，但背面的片元又会被遮挡，这样就可以让线在边缘处生成。

~~~glsl
// 定义顶点着色器输入结构
struct Attributes
{
    float4 positionOS   : POSITION;   // 对象空间位置
    float3 normalOS     : NORMAL;     // 对象空间法线
    float4 tangentOS    : TANGENT;    // 对象空间切线
    float4 color        : COLOR;      // 顶点颜色（用于控制轮廓线宽度）
    float2 uv           : TEXCOORD0;  // 主UV坐标
    float3 smoothNormal : TEXCOORD7;  // 平滑法线（用于生成更平滑的轮廓线）
};

// 定义顶点着色器输出/片段着色器输入结构
struct Varyings
{
    float2 uv           : TEXCOORD0;  // UV坐标
    float4 positionCS   : SV_POSITION; // 裁剪空间位置
};
~~~



### 计算边缘线宽度

​	这个函数用于计算边缘线到宽度，参数是模型的观察空间的z分量。fovFactor用于消除摄像机的fov导致的影响，positionVS_Z相当于眼睛到这个物体的物理距离，z 不是一个真实的距离，而是一个用来保证“不管用什么镜头看，屏幕大小差不多的模型，z是一个视觉距离。

然后我们根据z来计算边缘线的宽度。_OutlineWidthParams的xyzw分量分别表示最小距离、最大距离、最小宽度，最大宽度。计算得到的k就是在最小距离和最大距离之间的百分比，是一个0-1的映射，这样以来就保证在距离区间范围内边缘线宽度有一个比较好的粗细插值，当然超出距离区间之后边缘线就是最大最小宽度了。

```glsl
// 根据远近计算轮廓线宽度函数,在视觉上保持一致
float GetOutlineWidth(float positionVS_Z)
{
    //消除不同相机焦距（FOV）对轮廓线观感的影响
    float fovFactor = 2.414 / UNITY_MATRIX_P[1].y;
    // 计算基于距离的z值（考虑视场角）
    float z = abs(positionVS_Z * fovFactor);

    // 获取轮廓线宽度参数
    float4 params = _OutlineWidthParams;
    // 计算基于距离的插值因子（0到1之间）
    float k = saturate((z - params.x) / (params.y - params.x));
    // 根据k计算轮廓线宽度
    float width = lerp(params.z, params.w, k);
    // 应用全局轮廓线宽度乘数并缩放
    return 0.01 * _OutlineWidth * width;
}
```

### 计算边缘线位置

这个函数用于计算边缘线的生成位置以及效果，首先计算视图空间下的z，通过上述函数计算出边缘线的宽度，再获取到法线的视图空间下的坐标，并将z轴忽略，只考虑轮廓在xy方向上的扩展，同时将顶点位置向后偏移，使用参数_OutlineZOffset控制z轴方向的移动，防止深度冲突。然后就是最关键的计算方法，**在顶点当前的位置，沿着法线方向扩展width的长度，得到新的顶点位置**，根据参数_ScreenOffset在外部控制它的xy方向的偏移量，再转换到裁切空间返回出去。

~~~glsl
// 获取轮廓线位置函数
float4 GetOutlinePosition(VertexPositionInputs vertexInput, VertexNormalInputs normalInput, half4 vertexColor)
{
    // 获取视图空间Z坐标
    float z = vertexInput.positionVS.z;
    // 计算轮廓线宽度，考虑顶点颜色的alpha通道（可用于控制不同部位的轮廓线宽度）
    float width = GetOutlineWidth(z) * vertexColor.a;

    // 将法线转换到视图空间
    half3 normalVS = TransformWorldToViewNormal(normalInput.normalWS);
    // 确保法线在XY平面上（忽略Z分量），使轮廓线在屏幕平面上扩展
    normalVS = SafeNormalize(half3(normalVS.xy, 0.0));

    // 获取视图空间位置
    float3 positionVS = vertexInput.positionVS;
    // 应用Z偏移（沿视图方向偏移，防止深度冲突）
    positionVS += 0.01 * _OutlineZOffset * SafeNormalize(positionVS);
    // 沿法线方向扩展轮廓线
    positionVS += width * normalVS;

    // 转换到裁剪空间
    float4 positionCS = TransformWViewToHClip(positionVS);
    // 应用屏幕空间偏移
    positionCS.xy += _ScreenOffset.zw * positionCS.w;

    return positionCS;
}
~~~

### 顶点——片元着色器

​	顶点着色器主要用于计算上面两个函数所需的参数，计算顶点、法线等在坐标系空间之间的转化，片元着色器根据LightMap的a通道选取轮廓线颜色。

其中平滑法线

~~~glsl
// 轮廓线顶点着色器
Varyings OutlinePassVertex(Attributes input)
{
    // 获取顶点位置输入（对象空间到世界、视图、裁剪空间的转换）
    VertexPositionInputs vertexInput = GetVertexPositionInputs(input.positionOS.xyz);
    // 获取法线输入（对象空间到世界空间的转换）
    VertexNormalInputs normalInput = GetVertexNormalInputs(input.normalOS, input.tangentOS);

    // 构建切线到世界空间的变换矩阵
    half3x3 tangentToWorld = half3x3(normalInput.tangentWS, normalInput.bitangentWS, normalInput.normalWS);
    // 解包平滑法线（从[0,1]范围转换到[-1,1]范围）
    half3 normalTS = 2.0 * (input.smoothNormal - 0.5);
    // 将法线从切线空间转换到世界空间
    half3 normalWS = TransformTangentToWorld(normalTS, tangentToWorld, true);
    // 根据设置选择使用原始法线或平滑法线
    normalInput.normalWS = lerp(normalInput.normalWS, normalWS, _UseSmoothNormal);

    // 计算轮廓线位置
    float4 positionCS = GetOutlinePosition(vertexInput, normalInput, input.color);

    // 初始化输出结构
    Varyings output = (Varyings)0;
    // 变换UV坐标（应用纹理的缩放和偏移）
    output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
    // 设置裁剪空间位置
    output.positionCS = positionCS;

    return output;
}

// 轮廓线片段着色器
half4 OutlinePassFragment(Varyings input) : SV_TARGET
{
    // 采样光照贴图获取材质信息
    half4 lightMap = SAMPLE_TEXTURE2D(_LightMap, sampler_LightMap, input.uv);
    // 获取材质类型（存储在光照贴图的alpha通道中）
    half material = lightMap.a;

    // 根据材质类型选择轮廓线颜色
    // 使用阶梯式插值，根据材质值选择不同的轮廓线颜色
    half4 color = _OutlineColor5; // 默认颜色（材质值最低）
    color = lerp(color, _OutlineColor4, step(0.2, material)); // 材质值>0.2使用颜色4
    color = lerp(color, _OutlineColor3, step(0.4, material)); // 材质值>0.4使用颜色3
    color = lerp(color, _OutlineColor2, step(0.6, material)); // 材质值>0.6使用颜色2
    color = lerp(color, _OutlineColor, step(0.8, material));  // 材质值>0.8使用颜色1

    // 返回轮廓线颜色
    return color;
}
~~~



## 前向渲染着色器

在这个着色器实现了许多核心功能，下面进行逐一的分析解释。

~~~glsl
// 定义顶点着色器输入结构
struct Attributes
{
    float4 positionOS   : POSITION;   // 对象空间位置
    float3 normalOS     : NORMAL;     // 对象空间法线
    float4 tangentOS    : TANGENT;    // 对象空间切线
    float4 color        : COLOR;      // 顶点颜色
    float2 uv           : TEXCOORD0;  // 主UV坐标
    float2 backUV       : TEXCOORD1;  // 背面UV坐标（用于双面渲染）
};

// 定义顶点着色器输出/片段着色器输入结构
struct Varyings
{
    float2 uv           : TEXCOORD0;  // 主UV坐标
    float2 backUV       : TEXCOORD1;  // 背面UV坐标
    float3 positionWS   : TEXCOORD2;  // 世界空间位置
    half3 tangentWS     : TEXCOORD3;  // 世界空间切线
    half3 bitangentWS   : TEXCOORD4;  // 世界空间副切线
    half3 normalWS      : TEXCOORD5;  // 世界空间法线
    float4 positionNDC  : TEXCOORD6;  // 标准化设备坐标
    half4 color         : COLOR;      // 顶点颜色
    float4 positionCS   : SV_POSITION; // 裁剪空间位置
};
~~~



### 阴影

卡通角色的光照效果主要基于半兰伯特漫反射实现，再通过环境光遮蔽(AO)来增强细节。AO的体现在衣服袖口褶皱以及袖口下的皮肤投射袖口阴影等细节上，不受光照影响，一直显示。

~~~glsl
// 计算阴影函数
half GetShadow(Varyings input, half3 lightDirection, half aoFactor)
{
    // 计算法线与光源方向的点积（兰伯特系数）
    half NDotL = dot(input.normalWS, lightDirection);
    // 转换为半兰伯特值（范围从[-1,1]到[0,1]）
    half halfLambert = 0.5 * NDotL + 0.5;
    // 应用环境光遮蔽因子计算阴影
    half shadow = saturate(2.0 * halfLambert * aoFactor);
    // 如果AO因子接近1，则不显示阴影
    return lerp(shadow, 1.0, step(0.9, aoFactor));
}
~~~

![image-20251008230029679](https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20251008230029679.png)

<center>有AO时的阴影效果<center>

![image-20251008230157670](https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20251008230157670.png)

<center>无AO时的阴影效果<center>

其中AO是通过采样光照贴图的g通道和顶点颜色的r通道点乘获得的。

~~~glsl
// 普通阴影计算（考虑环境光遮蔽）【lightmap和color点乘制作的AO效果更好】
//光照贴图提供的AO是逐纹素的，精度高，但可能不够平滑（尤其是低分辨率的光照贴图）。
//顶点颜色提供的AO是逐顶点的，精度较低（通过顶点插值），但可以平滑过渡。
half aoFactor = lightMap.g * input.color.r;
half shadow = GetShadow(input, lightDirection, aoFactor);
~~~

它的返回值是0-1的数据，用于表示阴影信息。

### 面部阴影（有向距离场SDF）

面部阴影如果直接采用漫反射的方式会非常难看，通常采用光照贴图的方式来实现。对于面部光照，一般只考虑光照从左到右的效果，而不考虑从上到下的光照效果。所以在计算时需要忽略Y方向上的影响，防止对结果产生影响。



<center>面部光照贴图</center>

![image-20251008155325167](https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20251008155325167.png)

<center>面部阴影遮罩贴图</center>

首先就是获取面部的朝前向量以及光源方向，计算他们的点积与叉积，点积用于光照计算，叉积用于判断用于判断光源在面部的左方还是右方。面部前向向量需要在程序中实时修改才会显示正确的光照效果，光照原理其实本质和兰伯特差不多，可以想象将脸部抽象为一个平面。



<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20251008153854406.png" alt="image-20251008153854406" style="zoom: 80%;" />

<center>面部光照贴图</center>

在上图可以看到，这张面部光照贴图从右到左逐渐变暗，可以根据面部的光照信息来采样贴图数据就可以达到所需效果。观察面部光照贴图可以发现灯光不论从哪个方向来它的光照效果理论都是一致的，所以我们需要通过面部的前向向量与光照方向叉积，判断光源处在左边还是右边，然后对uv进行翻转操作。

对于脸部的光照其实原理就是根据光照和面部前方向量的点积比较这张光照贴图的值来决定是暗还是亮，面部阴影只有两种颜色效果，没有过渡。

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20251008155325167.png" alt="image-20251008155325167" style="zoom:67%;" />

<center>面部阴影遮罩贴图</center>

脸部的某些区域是不希望受到光照影响的，例如眼睛部分，在什么情况下都应该是常量，这就需要一个遮罩贴图来控制哪些部分受到阴影影响。上图就是脸部区域呈现黑色，而眼睛部分是白色。那么0.0(黑色区域)是完全遮罩，完全使用计算的面部阴影效果，1(白色区域)眼睛则完全不受阴影影响，保持明亮，0-1(灰色区域)启用面部阴影和完全明亮混合效果。

最后函数的返回值是0或1，表示哪些地方是暗，哪些地方是亮。

~~~glsl
// 计算面部阴影函数
half GetFaceShadow(Varyings input, half3 lightDirection)
{
    // 获取面部前方向量（忽略Y分量）[这个参数需要实时获取]
    half3 F = SafeNormalize(half3(_FaceDirection.x, 0.0, _FaceDirection.z));
    // 获取光源方向（忽略Y分量）
    half3 L = SafeNormalize(half3(lightDirection.x, 0.0, lightDirection.z));
    // 计算面部方向与光源方向的点积
    half FDotL = dot(F, L);
    // 计算叉积的Y分量，用于确定光源在面部的左方还是右方
    half FCrossL = cross(F, L).y;
    
    // 准备阴影UV
    half2 shadowUV = input.uv;
    // 根据左右翻转UV,因为面部阴影贴图通常只制作了一侧的光照效果，现在需要根据光照方向来确定贴图的方向从而保证效果
    shadowUV.x = lerp(shadowUV.x, 1.0 - shadowUV.x, step(0.0, FCrossL));
    // 采样面部光照贴图（用于影响面部光照与阴影的显示效果）
    half faceShadowMap = SAMPLE_TEXTURE2D(_FaceLightMap, sampler_FaceLightMap, shadowUV).r;
    // 计算面部阴影（返回0或1，0表示该区域暗，1表示该区域亮）
    half faceShadow = step(-0.5 * FDotL + 0.5 + _FaceShadowOffset, faceShadowMap);

    // 采样面部阴影遮罩(遮罩哪些区域受光照影响)[打开遮罩贴图的a通道法线，眼睛部分是白色的，而其余的面部是黑色的，部分区域为灰色]
    //[那么0.0(黑色区域)是完全遮罩，完全使用计算的面部阴影效果，1(白色区域)眼睛则完全不受阴影影响，保持明亮，0-1(灰色区域)启用面部阴影和完全明亮混合效果]
    half faceMask = SAMPLE_TEXTURE2D(_FaceShadow, sampler_FaceShadow, input.uv).a;
    // 应用遮罩（遮罩区域不显示阴影）
    half maskedFaceShadow = lerp(faceShadow, 1.0, faceMask);

    return maskedFaceShadow;
}

~~~

面部阴影效果如下所示：

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20251008162308681.png" alt="image-20251008162308681" style="zoom:50%;" />



### Ramp采样

Ramp采样是实现这种非真实感的核心。简单来说就是通过一张采样贴图，通过上述计算的阴影信息来选择贴图中的某一块颜色来作为阴影颜色显示。

~~~glsl
// 获取阴影颜色函数(Ramp采样)
half3 GetShadowColor(half shadow, half material, half day)
{
    // 根据材质类型确定阴影渐变纹理的V坐标
    int index = 4; // 默认索引
    index = lerp(index, 1, step(0.2, material)); // 材质>0.2使用索引1
    index = lerp(index, 2, step(0.4, material)); // 材质>0.4使用索引2
    index = lerp(index, 0, step(0.6, material)); // 材质>0.6使用索引0
    index = lerp(index, 3, step(0.8, material)); // 材质>0.8使用索引3

    // 计算阴影渐变纹理的U坐标
    half rangeMin = 0.5 + _ShadowOffset - _ShadowSmoothness;
    half rangeMax = 0.5 + _ShadowOffset;
    // 使用平滑步长计算UV坐标
    half2 rampUV = half2(smoothstep(rangeMin, rangeMax, shadow), index / 10.0 + 0.5 * day + 0.05);
    // 采样阴影渐变纹理
    half3 shadowRamp = SAMPLE_TEXTURE2D(_ShadowRamp, sampler_ShadowRamp, rampUV);

    // 应用阴影颜色
    half3 shadowColor = shadowRamp * lerp(_ShadowColor, 1.0, smoothstep(0.9, 1.0, rampUV.x));
    // 完全照亮区域不使用阴影颜色
    shadowColor = lerp(shadowColor, 1.0, step(rangeMax, shadow));

    return shadowColor;
}
~~~

其中材质这个参数使用

```glsl
// 采样光照贴图
half4 lightMap = SAMPLE_TEXTURE2D(_LightMap, sampler_LightMap, input.uv);
// 确定材质类型（使用自定义或贴图中的）
half material = lerp(lightMap.a, _CustomMaterialType, _UseCustomMaterialType);

 // 获取阴影颜色
    half3 shadowColor = GetShadowColor(shadow, material, _IsDay);
```

展示一下效果

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20251008220151123.png" alt="image-20251008220151123" style="zoom: 50%;" />

可以发现阴影颜色是分阶显示的，左边最暗，显示的是同种颜色，中间有一段过渡，介于暗和亮之间，右边就是完全亮，没有阴影颜色。下面开始解释函数实现原理。

前面说过最终计算得到的阴影身体部分是0-1的值，脸部是0或1的值，基于这个数值对其进行插值，采用Ramp图中对应的颜色即可实现这种色阶阴影效果。因为Ramp图本身就是离散化的采样图,没有过渡，所以界限分明。

index是材质类型，控制Ramp纹理的垂直坐标，表示材质类型，index默认受到光照贴图的a通道影响，或者自定义贴图，shadow控制水平坐标，表示阴影强度。

我们假设胳膊从左到右的shadow值：
shadow[0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9]  
颜色     [ 深色                   浅色                                ]    

​		↑                         ↑                   ↑
​	     最深阴影             过渡区          亮部

![022e3320352663c4696d9f8551dfb87b](https://typorytp.oss-cn-hangzhou.aliyuncs.com/022e3320352663c4696d9f8551dfb87b.png)

rangeMin和rangeMax就是定义最深阴影和过渡区这两个交界处的值，小于rangeMin的区域采用最暗的阴影颜色，大于rangeMin且小于rangeMax则采用较暗亮度的颜色，大于rangeMax的区域则不使用阴影颜色。

~~~glsl
half2 rampUV = half2(smoothstep(rangeMin, rangeMax, shadow), index / 10.0 + 0.5 * day + 0.05);
~~~

其实直接使用shadow作为u坐标也没问题，因为shadow值始终是0-1的范围，但是不使用smoothstep(rangeMin, rangeMax, shadow)作为u坐标，会导致效果变差，比如中间的浅色区域消失，因为这个区间的shadow面积很小，所以效果差，而使用参数调控的rangeMin, rangeMax则可以自由控制显示区域。

### 高光反射 

高光反射基于Blinn-Phong模型计算高光，使用MatCap(材质捕捉)技术实现反射细节。

```glsl
// 计算高光函数
half3 GetSpecular(Varyings input, half3 lightDirection, half3 albedo, half3 lightMap)
{
    // 获取视图方向
    half3 V = GetWorldSpaceNormalizeViewDir(input.positionWS);
    // 计算半角向量（Blinn-Phong模型）
    half3 H = SafeNormalize(lightDirection + V);
    // 计算法线与半角向量的点积
    half NDotH = dot(input.normalWS, H);
    // 使用Blinn-Phong模型计算高光
    half blinnPhong = pow(saturate(NDotH), _SpecularSmoothness);

    // 将法线转换到视图空间
    half3 normalVS = TransformWorldToViewNormal(input.normalWS, true);
    // 计算Matcap UV坐标（基于视图空间法线）
    half2 matcapUV = 0.5 * normalVS.xy + 0.5;
    // 采样金属贴图
    half3 metalMap = SAMPLE_TEXTURE2D(_MetalMap, sampler_MetalMap, matcapUV);

    // 计算非金属高光
    half3 nonMetallic = step(1.1, lightMap.b + blinnPhong) * lightMap.r * _NonmetallicIntensity;
    // 计算金属高光
    half3 metallic = blinnPhong * lightMap.b * albedo * metalMap * _MetallicIntensity;
    // 根据材质类型混合高光（lightMap.r > 0.9时为金属）
    half3 specular = lerp(nonMetallic, metallic, step(0.9, lightMap.r));

    return specular;
}
```

### 边缘光



~~~glsl
// 计算边缘光函数
half GetRim(Varyings input)
{
    // 将法线转换到视图空间
    half3 normalVS = TransformWorldToViewNormal(input.normalWS, true);
    // 计算屏幕空间UV
    float2 uv = input.positionNDC.xy / input.positionNDC.w;
    // 计算基于法线的偏移
    float2 offset = float2(_RimOffset * normalVS.x / _ScreenParams.x, 0.0);

    // 采样当前像素深度
    float depth = LinearEyeDepth(SampleSceneDepth(uv), _ZBufferParams);
    // 采样偏移后的深度
    float offsetDepth = LinearEyeDepth(SampleSceneDepth(uv + offset), _ZBufferParams);
    // 计算边缘光强度（基于深度差）
    half rim = smoothstep(0.0, _RimThreshold, offsetDepth - depth) * _RimIntensity;

    // 获取视图方向
    half3 V = GetWorldSpaceNormalizeViewDir(input.positionWS);
    // 计算法线与视图方向的点积
    half NDotV = dot(input.normalWS, V);
    // 使用Fresnel效应增强边缘光
    half fresnel = pow(saturate(1.0 - NDotV), 5.0);

    return rim * fresnel;
}
~~~



### 顶点——片元着色器

~~~glsl
// 前向渲染顶点着色器
Varyings ForwardPassVertex(Attributes input)
{
    // 获取顶点位置输入
    VertexPositionInputs vertexInput = GetVertexPositionInputs(input.positionOS.xyz);
    // 获取法线输入
    VertexNormalInputs normalInput = GetVertexNormalInputs(input.normalOS, input.tangentOS);

    // 初始化输出结构
    Varyings output = (Varyings)0;
    // 变换主UV坐标
    output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
    // 变换背面UV坐标
    output.backUV = TRANSFORM_TEX(input.backUV, _BaseMap);
    // 存储世界空间位置
    output.positionWS = vertexInput.positionWS;
    // 存储世界空间切线
    output.tangentWS = normalInput.tangentWS;
    // 存储世界空间副切线
    output.bitangentWS = normalInput.bitangentWS;
    // 存储世界空间法线
    output.normalWS = normalInput.normalWS;
    // 传递顶点颜色
    output.color = input.color;
    // 存储标准化设备坐标
    output.positionNDC = vertexInput.positionNDC;
    // 存储裁剪空间位置
    output.positionCS = vertexInput.positionCS;

    // 应用屏幕空间偏移
    output.positionCS.xy += _ScreenOffset.xy * output.positionCS.w;

    return output;
}

// 前向渲染片段着色器
half4 ForwardPassFragment(Varyings input, FRONT_FACE_TYPE facing : FRONT_FACE_SEMANTIC) : SV_TARGET
{
    // 双面渲染处理
    #if _DOUBLE_SIDED
        // 如果是背面，使用背面UV
        input.uv = lerp(input.uv, input.backUV, IS_FRONT_VFACE(facing, 0.0, 1.0));
    #endif

    // 采样基础纹理
    half4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv);
    // 应用基础颜色
    half3 albedo = baseMap.rgb * _BaseColor.rgb;
    // 获取透明度
    half alpha = baseMap.a;

    // 面部泛红效果
    #if _IS_FACE
        // 根据面部泛红强度和透明度混合颜色
        albedo = lerp(albedo, _FaceBlushColor.rgb, _FaceBlushStrength * alpha);
    #endif

    // 法线贴图处理
    #if _NORMAL_MAP
        // 构建切线到世界空间的变换矩阵
        half3x3 tangentToWorld = half3x3(input.tangentWS, input.bitangentWS, input.normalWS);
        // 采样法线贴图
        half4 normalMap = SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, input.uv);
        // 解包法线（从[0,1]到[-1,1]）
        half3 normalTS = UnpackNormal(normalMap);
        // 将法线从切线空间转换到世界空间
        half3 normalWS = TransformTangentToWorld(normalTS, tangentToWorld, true);
        // 更新世界空间法线
        input.normalWS = normalWS;
    #endif

    // 获取主光源信息
    Light mainLight = GetMainLight();
    // 应用光源方向乘数
    half3 lightDirection = SafeNormalize(mainLight.direction * _LightDirectionMultiplier);

    // 采样光照贴图
    half4 lightMap = SAMPLE_TEXTURE2D(_LightMap, sampler_LightMap, input.uv);
    // 确定材质类型（使用自定义或贴图中的）
    half material = lerp(lightMap.a, _CustomMaterialType, _UseCustomMaterialType);
    
    // 计算阴影
    #if _IS_FACE
        // 面部特殊阴影计算
        half shadow = GetFaceShadow(input, lightDirection);
    #else
    // 普通阴影计算（考虑环境光遮蔽）【lightmap和color点乘制作的AO效果更好】
    //光照贴图提供的AO是逐纹素的，精度高，但可能不够平滑（尤其是低分辨率的光照贴图）。
    //顶点颜色提供的AO是逐顶点的，精度较低（通过顶点插值），但可以平滑过渡。
        half aoFactor = lightMap.g * input.color.r;
        half shadow = GetShadow(input, lightDirection, aoFactor);
    #endif
    
    // 获取阴影颜色
    half3 shadowColor = GetShadowColor(shadow, material, _IsDay);

    // 高光计算
    half3 specular = 0.0;
    #if _SPECULAR
        specular = GetSpecular(input, lightDirection, albedo, lightMap.rgb);
    #endif

    // 自发光计算
    half3 emission = 0.0;
    #if _EMISSION
        emission = albedo * _EmissionIntensity * alpha;
    #endif

    // 边缘光计算
    half3 rim = 0.0;
    #if _RIM
        rim = albedo * GetRim(input);
    #endif

    // 最终颜色合成
    half3 finalColor = albedo * shadowColor + specular + rim + emission;
    // 固定透明度为1（不透明）
    half finalAlpha = 1.0;

    // 返回最终颜色
    return half4(finalColor, finalAlpha);
}
~~~





## 完整代码

~~~glsl
// 包含URP核心库文件
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
// 包含URP光照库文件
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
// 包含深度纹理声明库文件
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"

// 定义顶点着色器输入结构
struct Attributes
{
    float4 positionOS   : POSITION;   // 对象空间位置
    float3 normalOS     : NORMAL;     // 对象空间法线
    float4 tangentOS    : TANGENT;    // 对象空间切线
    float4 color        : COLOR;      // 顶点颜色
    float2 uv           : TEXCOORD0;  // 主UV坐标
    float2 backUV       : TEXCOORD1;  // 背面UV坐标（用于双面渲染）
};

// 定义顶点着色器输出/片段着色器输入结构
struct Varyings
{
    float2 uv           : TEXCOORD0;  // 主UV坐标
    float2 backUV       : TEXCOORD1;  // 背面UV坐标
    float3 positionWS   : TEXCOORD2;  // 世界空间位置
    half3 tangentWS     : TEXCOORD3;  // 世界空间切线
    half3 bitangentWS   : TEXCOORD4;  // 世界空间副切线
    half3 normalWS      : TEXCOORD5;  // 世界空间法线
    float4 positionNDC  : TEXCOORD6;  // 标准化设备坐标
    half4 color         : COLOR;      // 顶点颜色
    float4 positionCS   : SV_POSITION; // 裁剪空间位置
};

// 计算阴影函数
half GetShadow(Varyings input, half3 lightDirection, half aoFactor)
{
    // 计算法线与光源方向的点积（兰伯特系数）
    half NDotL = dot(input.normalWS, lightDirection);
    // 转换为半兰伯特值（范围从[-1,1]到[0,1]）
    half halfLambert = 0.5 * NDotL + 0.5;
    // 应用环境光遮蔽因子计算阴影
    half shadow = saturate(2.0 * halfLambert * aoFactor);
    // 如果AO因子接近1，则不显示阴影
    return lerp(shadow, 1.0, step(0.9, aoFactor));
}

// 计算面部阴影函数
half GetFaceShadow(Varyings input, half3 lightDirection)
{
    // 获取面部前方向量（忽略Y分量）
    half3 F = SafeNormalize(half3(_FaceDirection.x, 0.0, _FaceDirection.z));
    // 获取光源方向（忽略Y分量）
    half3 L = SafeNormalize(half3(lightDirection.x, 0.0, lightDirection.z));
    // 计算面部方向与光源方向的点积
    half FDotL = dot(F, L);
    // 计算叉积的Y分量（用于确定左右）
    half FCrossL = cross(F, L).y;
    
    // 准备阴影UV
    half2 shadowUV = input.uv;
    // 根据左右翻转UV（如果FCrossL > 0）
    shadowUV.x = lerp(shadowUV.x, 1.0 - shadowUV.x, step(0.0, FCrossL));
    // 采样面部光照贴图
    half faceShadowMap = SAMPLE_TEXTURE2D(_FaceLightMap, sampler_FaceLightMap, shadowUV).r;
    // 计算面部阴影（基于点积和偏移）
    half faceShadow = step(-0.5 * FDotL + 0.5 + _FaceShadowOffset, faceShadowMap);

    // 采样面部阴影遮罩
    half faceMask = SAMPLE_TEXTURE2D(_FaceShadow, sampler_FaceShadow, input.uv).a;
    // 应用遮罩（遮罩区域不显示阴影）
    half maskedFaceShadow = lerp(faceShadow, 1.0, faceMask);

    return maskedFaceShadow;
}

// 获取阴影颜色函数
half3 GetShadowColor(half shadow, half material, half day)
{
    // 根据材质类型确定阴影渐变纹理的V坐标
    int index = 4; // 默认索引
    index = lerp(index, 1, step(0.2, material)); // 材质>0.2使用索引1
    index = lerp(index, 2, step(0.4, material)); // 材质>0.4使用索引2
    index = lerp(index, 0, step(0.6, material)); // 材质>0.6使用索引0
    index = lerp(index, 3, step(0.8, material)); // 材质>0.8使用索引3

    // 计算阴影渐变纹理的U坐标
    half rangeMin = 0.5 + _ShadowOffset - _ShadowSmoothness;
    half rangeMax = 0.5 + _ShadowOffset;
    // 使用平滑步长计算UV坐标
    half2 rampUV = half2(smoothstep(rangeMin, rangeMax, shadow), index / 10.0 + 0.5 * day + 0.05);
    // 采样阴影渐变纹理
    half3 shadowRamp = SAMPLE_TEXTURE2D(_ShadowRamp, sampler_ShadowRamp, rampUV);

    // 应用阴影颜色
    half3 shadowColor = shadowRamp * lerp(_ShadowColor, 1.0, smoothstep(0.9, 1.0, rampUV.x));
    // 完全照亮区域不使用阴影颜色
    shadowColor = lerp(shadowColor, 1.0, step(rangeMax, shadow));

    return shadowColor;
}

// 计算高光函数
half3 GetSpecular(Varyings input, half3 lightDirection, half3 albedo, half3 lightMap)
{
    // 获取视图方向
    half3 V = GetWorldSpaceNormalizeViewDir(input.positionWS);
    // 计算半角向量（Blinn-Phong模型）
    half3 H = SafeNormalize(lightDirection + V);
    // 计算法线与半角向量的点积
    half NDotH = dot(input.normalWS, H);
    // 使用Blinn-Phong模型计算高光
    half blinnPhong = pow(saturate(NDotH), _SpecularSmoothness);

    // 将法线转换到视图空间
    half3 normalVS = TransformWorldToViewNormal(input.normalWS, true);
    // 计算Matcap UV坐标（基于视图空间法线）
    half2 matcapUV = 0.5 * normalVS.xy + 0.5;
    // 采样金属贴图
    half3 metalMap = SAMPLE_TEXTURE2D(_MetalMap, sampler_MetalMap, matcapUV);

    // 计算非金属高光
    half3 nonMetallic = step(1.1, lightMap.b + blinnPhong) * lightMap.r * _NonmetallicIntensity;
    // 计算金属高光
    half3 metallic = blinnPhong * lightMap.b * albedo * metalMap * _MetallicIntensity;
    // 根据材质类型混合高光（lightMap.r > 0.9时为金属）
    half3 specular = lerp(nonMetallic, metallic, step(0.9, lightMap.r));

    return specular;
}

// 计算边缘光函数
half GetRim(Varyings input)
{
    // 将法线转换到视图空间
    half3 normalVS = TransformWorldToViewNormal(input.normalWS, true);
    // 计算屏幕空间UV
    float2 uv = input.positionNDC.xy / input.positionNDC.w;
    // 计算基于法线的偏移
    float2 offset = float2(_RimOffset * normalVS.x / _ScreenParams.x, 0.0);

    // 采样当前像素深度
    float depth = LinearEyeDepth(SampleSceneDepth(uv), _ZBufferParams);
    // 采样偏移后的深度
    float offsetDepth = LinearEyeDepth(SampleSceneDepth(uv + offset), _ZBufferParams);
    // 计算边缘光强度（基于深度差）
    half rim = smoothstep(0.0, _RimThreshold, offsetDepth - depth) * _RimIntensity;

    // 获取视图方向
    half3 V = GetWorldSpaceNormalizeViewDir(input.positionWS);
    // 计算法线与视图方向的点积
    half NDotV = dot(input.normalWS, V);
    // 使用Fresnel效应增强边缘光
    half fresnel = pow(saturate(1.0 - NDotV), 5.0);

    return rim * fresnel;
}

// 前向渲染顶点着色器
Varyings ForwardPassVertex(Attributes input)
{
    // 获取顶点位置输入
    VertexPositionInputs vertexInput = GetVertexPositionInputs(input.positionOS.xyz);
    // 获取法线输入
    VertexNormalInputs normalInput = GetVertexNormalInputs(input.normalOS, input.tangentOS);

    // 初始化输出结构
    Varyings output = (Varyings)0;
    // 变换主UV坐标
    output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
    // 变换背面UV坐标
    output.backUV = TRANSFORM_TEX(input.backUV, _BaseMap);
    // 存储世界空间位置
    output.positionWS = vertexInput.positionWS;
    // 存储世界空间切线
    output.tangentWS = normalInput.tangentWS;
    // 存储世界空间副切线
    output.bitangentWS = normalInput.bitangentWS;
    // 存储世界空间法线
    output.normalWS = normalInput.normalWS;
    // 传递顶点颜色
    output.color = input.color;
    // 存储标准化设备坐标
    output.positionNDC = vertexInput.positionNDC;
    // 存储裁剪空间位置
    output.positionCS = vertexInput.positionCS;

    // 应用屏幕空间偏移
    output.positionCS.xy += _ScreenOffset.xy * output.positionCS.w;

    return output;
}

// 前向渲染片段着色器
half4 ForwardPassFragment(Varyings input, FRONT_FACE_TYPE facing : FRONT_FACE_SEMANTIC) : SV_TARGET
{
    // 双面渲染处理
    #if _DOUBLE_SIDED
        // 如果是背面，使用背面UV
        input.uv = lerp(input.uv, input.backUV, IS_FRONT_VFACE(facing, 0.0, 1.0));
    #endif

    // 采样基础纹理
    half4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.uv);
    // 应用基础颜色
    half3 albedo = baseMap.rgb * _BaseColor.rgb;
    // 获取透明度
    half alpha = baseMap.a;

    // 面部泛红效果
    #if _IS_FACE
        // 根据面部泛红强度和透明度混合颜色
        albedo = lerp(albedo, _FaceBlushColor.rgb, _FaceBlushStrength * alpha);
    #endif

    // 法线贴图处理
    #if _NORMAL_MAP
        // 构建切线到世界空间的变换矩阵
        half3x3 tangentToWorld = half3x3(input.tangentWS, input.bitangentWS, input.normalWS);
        // 采样法线贴图
        half4 normalMap = SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, input.uv);
        // 解包法线（从[0,1]到[-1,1]）
        half3 normalTS = UnpackNormal(normalMap);
        // 将法线从切线空间转换到世界空间
        half3 normalWS = TransformTangentToWorld(normalTS, tangentToWorld, true);
        // 更新世界空间法线
        input.normalWS = normalWS;
    #endif

    // 获取主光源信息
    Light mainLight = GetMainLight();
    // 应用光源方向乘数
    half3 lightDirection = SafeNormalize(mainLight.direction * _LightDirectionMultiplier);

    // 采样光照贴图
    half4 lightMap = SAMPLE_TEXTURE2D(_LightMap, sampler_LightMap, input.uv);
    // 确定材质类型（使用自定义或贴图中的）
    half material = lerp(lightMap.a, _CustomMaterialType, _UseCustomMaterialType);
    
    // 计算阴影
    #if _IS_FACE
        // 面部特殊阴影计算
        half shadow = GetFaceShadow(input, lightDirection);
    #else
        // 普通阴影计算（考虑环境光遮蔽）
        half aoFactor = lightMap.g * input.color.r;
        half shadow = GetShadow(input, lightDirection, aoFactor);
    #endif
    
    // 获取阴影颜色
    half3 shadowColor = GetShadowColor(shadow, material, _IsDay);

    // 高光计算
    half3 specular = 0.0;
    #if _SPECULAR
        specular = GetSpecular(input, lightDirection, albedo, lightMap.rgb);
    #endif

    // 自发光计算
    half3 emission = 0.0;
    #if _EMISSION
        emission = albedo * _EmissionIntensity * alpha;
    #endif

    // 边缘光计算
    half3 rim = 0.0;
    #if _RIM
        rim = albedo * GetRim(input);
    #endif

    // 最终颜色合成
    half3 finalColor = albedo * shadowColor + specular + rim + emission;
    // 固定透明度为1（不透明）
    half finalAlpha = 1.0;

    // 返回最终颜色
    return half4(finalColor, finalAlpha);
}// 包含URP核心库文件，提供常用的函数和宏定义
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

// 开始定义材质属性常量缓冲区
// UnityPerMaterial缓冲区包含所有在材质面板中定义的属性
CBUFFER_START(UnityPerMaterial)
    // 基础纹理的缩放和偏移参数（Tiling和Offset）
    float4  _BaseMap_ST;
    // 基础颜色，用于乘以基础纹理颜色
    half4   _BaseColor;
    // 是否为白天模式（0或1）
    half    _IsDay;
    // 剔除模式（0=关闭，1=正面，2=背面）
    half    _Cull;
    // 源混合因子（用于透明度混合）
    half    _SrcBlend;
    // 目标混合因子（用于透明度混合）
    half    _DstBlend;

    // 光源方向乘数，用于调整光源方向的影响
    half4   _LightDirectionMultiplier;
    // 阴影偏移量，控制阴影的位置
    half    _ShadowOffset;
    // 阴影平滑度，控制阴影边缘的柔和程度
    half    _ShadowSmoothness;
    // 阴影颜色，HDR颜色支持高动态范围
    half4   _ShadowColor;
    // 是否使用自定义材质类型（0或1）
    half    _UseCustomMaterialType;
    // 自定义材质类型的值
    half    _CustomMaterialType;

    // 自发光强度
    half    _EmissionIntensity;

    // 面部朝向向量
    half4   _FaceDirection;
    // 面部阴影偏移量
    half    _FaceShadowOffset;
    // 面部泛红颜色
    half4   _FaceBlushColor;
    // 面部泛红强度
    half    _FaceBlushStrength;

    // 高光平滑度，控制高光的大小
    half    _SpecularSmoothness;
    // 非金属材质的高光强度
    half    _NonmetallicIntensity;
    // 金属材质的高光强度
    half    _MetallicIntensity;

    // 边缘光偏移量
    half    _RimOffset;
    // 边缘光阈值
    half    _RimThreshold;
    // 边缘光强度
    half    _RimIntensity;

    // 是否使用平滑法线（0或1）
    half    _UseSmoothNormal;
    // 轮廓线宽度
    half    _OutlineWidth;
    // 轮廓线宽度参数（最小距离，最大距离，最小宽度，最大宽度）
    half4   _OutlineWidthParams;
    // 轮廓线Z轴偏移量
    half    _OutlineZOffset;
    // 屏幕空间偏移向量
    half4   _ScreenOffset;
    // 轮廓线颜色1
    half4   _OutlineColor;
    // 轮廓线颜色2
    half4   _OutlineColor2;
    // 轮廓线颜色3
    half4   _OutlineColor3;
    // 轮廓线颜色4
    half4   _OutlineColor4;
    // 轮廓线颜色5
    half4   _OutlineColor5;
// 结束常量缓冲区定义
CBUFFER_END

// 声明纹理和采样器
// 基础纹理和采样器
TEXTURE2D(_BaseMap);            SAMPLER(sampler_BaseMap);
// 光照贴图和采样器
TEXTURE2D(_LightMap);           SAMPLER(sampler_LightMap);
// 阴影渐变纹理和采样器
TEXTURE2D(_ShadowRamp);         SAMPLER(sampler_ShadowRamp);
// 法线贴图和采样器
TEXTURE2D(_NormalMap);          SAMPLER(sampler_NormalMap);
// 面部光照贴图和采样器
TEXTURE2D(_FaceLightMap);       SAMPLER(sampler_FaceLightMap);
// 面部阴影贴图和采样器
TEXTURE2D(_FaceShadow);         SAMPLER(sampler_FaceShadow);
// 金属贴图和采样器
TEXTURE2D(_MetalMap);           SAMPLER(sampler_MetalMap);
// 包含URP核心库文件，提供常用的函数和宏定义
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
// 包含URP光照库文件，提供光照相关函数
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

// 定义顶点着色器输入结构
struct Attributes
{
    float4 positionOS   : POSITION;   // 对象空间位置
    float3 normalOS     : NORMAL;     // 对象空间法线
    float4 tangentOS    : TANGENT;    // 对象空间切线
    float4 color        : COLOR;      // 顶点颜色（用于控制轮廓线宽度）
    float2 uv           : TEXCOORD0;  // 主UV坐标
    float3 smoothNormal : TEXCOORD7;  // 平滑法线（用于生成更平滑的轮廓线）
};

// 定义顶点着色器输出/片段着色器输入结构
struct Varyings
{
    float2 uv           : TEXCOORD0;  // UV坐标
    float4 positionCS   : SV_POSITION; // 裁剪空间位置
};

// 根据远近计算轮廓线宽度函数,在视觉上保持一致
float GetOutlineWidth(float positionVS_Z)
{
    //消除不同相机焦距（FOV）对轮廓线观感的影响
    float fovFactor = 2.414 / UNITY_MATRIX_P[1].y;
    // 计算基于距离的z值（考虑视场角）
    float z = abs(positionVS_Z * fovFactor);

    // 获取轮廓线宽度参数
    float4 params = _OutlineWidthParams;
    // 计算基于距离的插值因子（0到1之间）
    float k = saturate((z - params.x) / (params.y - params.x));
    // 根据k计算轮廓线宽度
    float width = lerp(params.z, params.w, k);
    // 应用全局轮廓线宽度乘数并缩放
    return 0.01 * _OutlineWidth * width;
}

// 获取轮廓线位置函数
float4 GetOutlinePosition(VertexPositionInputs vertexInput, VertexNormalInputs normalInput, half4 vertexColor)
{
    // 获取视图空间Z坐标
    float z = vertexInput.positionVS.z;
    // 计算轮廓线宽度，考虑顶点颜色的alpha通道（可用于控制不同部位的轮廓线宽度）
    float width = GetOutlineWidth(z) * vertexColor.a;

    // 将法线转换到视图空间
    half3 normalVS = TransformWorldToViewNormal(normalInput.normalWS);
    // 确保法线在XY平面上（忽略Z分量），使轮廓线在屏幕平面上扩展
    normalVS = SafeNormalize(half3(normalVS.xy, 0.0));

    // 获取视图空间位置
    float3 positionVS = vertexInput.positionVS;
    // 应用Z偏移（沿视图方向偏移，防止深度冲突）
    positionVS += 0.01 * _OutlineZOffset * SafeNormalize(positionVS);
    // 沿法线方向扩展轮廓线
    positionVS += width * normalVS;

    // 转换到裁剪空间
    float4 positionCS = TransformWViewToHClip(positionVS);
    // 应用屏幕空间偏移
    positionCS.xy += _ScreenOffset.zw * positionCS.w;

    return positionCS;
}

// 轮廓线顶点着色器
Varyings OutlinePassVertex(Attributes input)
{
    // 获取顶点位置输入（对象空间到世界、视图、裁剪空间的转换）
    VertexPositionInputs vertexInput = GetVertexPositionInputs(input.positionOS.xyz);
    // 获取法线输入（对象空间到世界空间的转换）
    VertexNormalInputs normalInput = GetVertexNormalInputs(input.normalOS, input.tangentOS);

    // 构建切线到世界空间的变换矩阵
    half3x3 tangentToWorld = half3x3(normalInput.tangentWS, normalInput.bitangentWS, normalInput.normalWS);
    // 解包平滑法线（从[0,1]范围转换到[-1,1]范围）
    half3 normalTS = 2.0 * (input.smoothNormal - 0.5);
    // 将法线从切线空间转换到世界空间
    half3 normalWS = TransformTangentToWorld(normalTS, tangentToWorld, true);
    // 根据设置选择使用原始法线或平滑法线
    normalInput.normalWS = lerp(normalInput.normalWS, normalWS, _UseSmoothNormal);

    // 计算轮廓线位置
    float4 positionCS = GetOutlinePosition(vertexInput, normalInput, input.color);

    // 初始化输出结构
    Varyings output = (Varyings)0;
    // 变换UV坐标（应用纹理的缩放和偏移）
    output.uv = TRANSFORM_TEX(input.uv, _BaseMap);
    // 设置裁剪空间位置
    output.positionCS = positionCS;

    return output;
}

// 轮廓线片段着色器
half4 OutlinePassFragment(Varyings input) : SV_TARGET
{
    // 采样光照贴图获取材质信息
    half4 lightMap = SAMPLE_TEXTURE2D(_LightMap, sampler_LightMap, input.uv);
    // 获取材质类型（存储在光照贴图的alpha通道中）
    half material = lightMap.a;

    // 根据材质类型选择轮廓线颜色
    // 使用阶梯式插值，根据材质值选择不同的轮廓线颜色
    half4 color = _OutlineColor5; // 默认颜色（材质值最低）
    color = lerp(color, _OutlineColor4, step(0.2, material)); // 材质值>0.2使用颜色4
    color = lerp(color, _OutlineColor3, step(0.4, material)); // 材质值>0.4使用颜色3
    color = lerp(color, _OutlineColor2, step(0.6, material)); // 材质值>0.6使用颜色2
    color = lerp(color, _OutlineColor, step(0.8, material));  // 材质值>0.8使用颜色1

    // 返回轮廓线颜色
    return color;
}
// 定义一个名为"URPGenshinToon"的着色器
Shader "URPGenshinToon"
{
    // Properties块：定义在材质面板中可见和可调节的参数
    Properties
    {
        // [Header(General)] 在材质面板中创建一个名为"General"的标题区域
        [Header(General)]
        // [MainTexture]标记表示这是主纹理，_BaseMap是变量名，"Base Map"是显示名称
        [MainTexture]_BaseMap("Base Map", 2D) = "white" {}
        // [MainColor]标记表示这是主颜色，_BaseColor是变量名，"Base Color"是显示名称
        [MainColor] _BaseColor("Base Color", Color) = (1,1,1,1)
        // [ToggleUI]创建一个开关UI元素，_IsDay是变量名，"Is Day"是显示名称
        [ToggleUI] _IsDay("Is Day", Float) = 1
        // [Toggle(_DOUBLE_SIDED)]创建一个开关，并关联到_DOUBLE_SIDED关键字
        [Toggle(_DOUBLE_SIDED)] _DoubleSided("Double Sided", Float) = 0
        // [Enum(UnityEngine.Rendering.CullMode)]创建一个枚举下拉菜单，选择剔除模式
        [Enum(UnityEngine.Rendering.CullMode)] _Cull("Cull", Float) = 2
        // [Enum(UnityEngine.Rendering.BlendMode)]创建一个枚举下拉菜单，选择源混合模式
        [Enum(UnityEngine.Rendering.BlendMode)] _SrcBlend("Src Blend", Float) = 1
        // [Enum(UnityEngine.Rendering.BlendMode)]创建一个枚举下拉菜单，选择目标混合模式
        [Enum(UnityEngine.Rendering.BlendMode)] _DstBlend("Dst Blend", Float) = 0

        // [Header(Shadow)] 在材质面板中创建一个名为"Shadow"的标题区域
        [Header(Shadow)]
        // 定义光照贴图纹理
        _LightMap("Light Map", 2D) = "white" {}
        // 定义光源方向乘数，用于调整光源方向的影响
        _LightDirectionMultiplier("Light Direction Multiplier", Vector) = (1,1,1,0)
        // 定义阴影偏移量，控制阴影的位置
        _ShadowOffset("Shadow Offset", Float) = 0
        // 定义阴影平滑度，控制阴影边缘的柔和程度
        _ShadowSmoothness("Shadow Smoothness", Float) = 0
        // [HDR]标记表示这是一个HDR颜色，支持高动态范围
        [HDR] _ShadowColor("Shadow Color", Color) = (1,1,1,1)
        // 定义阴影渐变纹理，用于控制阴影的过渡效果
        _ShadowRamp("Shadow Ramp", 2D) = "white" {}
        // 创建一个开关，控制是否使用自定义材质类型
        [ToggleUI] _UseCustomMaterialType("Use Custom Material Type", Float) = 0
        // 定义自定义材质类型的值
        _CustomMaterialType("Custom Material Type", Float) = 1

        // [Header(Emission)] 在材质面板中创建一个名为"Emission"的标题区域
        [Header(Emission)]
        // [Toggle(_EMISSION)]创建一个开关，并关联到_EMISSION关键字
        [Toggle(_EMISSION)] _UseEmission("Use Emission", Float) = 0
        // 定义自发光强度
        _EmissionIntensity("Emission Intensity", Float) = 1

        // [Header(Normal)] 在材质面板中创建一个名为"Normal"的标题区域
        [Header(Normal)]
        // [Toggle(_NORMAL_MAP)]创建一个开关，并关联到_NORMAL_MAP关键字
        [Toggle(_NORMAL_MAP)] _UseNormalMap("Use Normal Map", Float) = 0
        // [Normal]标记表示这是一个法线贴图
        [Normal] _NormalMap("Normal Map", 2D) = "bump" {}

        // [Header(Face)] 在材质面板中创建一个名为"Face"的标题区域
        [Header(Face)]
        // [Toggle(_IS_FACE)]创建一个开关，并关联到_IS_FACE关键字
        [Toggle(_IS_FACE)] _IsFace("Is Face", Float) = 0
        // 定义面部朝向向量
        _FaceDirection("Face Direction", Vector) = (0,0,1,0)
        // 定义面部阴影偏移量
        _FaceShadowOffset("Face Shadow Offset", Float) = 0
        // 定义面部泛红颜色
        _FaceBlushColor("Face Blush Color", Color) = (1,1,1,1)
        // 定义面部泛红强度
        _FaceBlushStrength("Face Blush Strength", Float) = 1
        // 定义面部光照贴图纹理
        _FaceLightMap("Face Light Map", 2D) = "white" {}
        // 定义面部阴影纹理
        _FaceShadow("Face Shadow", 2D) = "white" {}

        // [Header(Specular)] 在材质面板中创建一个名为"Specular"的标题区域
        [Header(Specular)]
        // [Toggle(_SPECULAR)]创建一个开关，并关联到_SPECULAR关键字
        [Toggle(_SPECULAR)] _UseSpecular("Use Specular", Float) = 0
        // 定义高光平滑度
        _SpecularSmoothness("Specular Smoothness", Float) = 1
        // 定义非金属材质的高光强度
        _NonmetallicIntensity("Nonmetallic Intensity", Float) = 1
        // 定义金属材质的高光强度
        _MetallicIntensity("Metallic Intensity", Float) = 1
        // 定义金属贴图纹理
        _MetalMap("Metal Map", 2D) = "white" {}

        // [Header(Rim Light)] 在材质面板中创建一个名为"Rim Light"的标题区域
        [Header(Rim Light)]
        // [Toggle(_RIM)]创建一个开关，并关联到_RIM关键字
        [Toggle(_RIM)] _UseRim("Use Rim", Float) = 0
        // 定义边缘光偏移量
        _RimOffset("Rim Offset", Float) = 1
        // 定义边缘光阈值
        _RimThreshold("Rim Threshold", Float) = 1
        // 定义边缘光强度
        _RimIntensity("Rim Intensity", Float) = 1

        // [Header(Outline)] 在材质面板中创建一个名为"Outline"的标题区域
        [Header(Outline)]
        // 创建一个开关UI元素，控制是否使用平滑法线
        [ToggleUI] _UseSmoothNormal("Use Smooth Normal", Float) = 0
        // 定义轮廓线宽度
        _OutlineWidth("Outline Width", Float) = 1
        // 定义轮廓线宽度参数向量，用于控制不同距离下的轮廓线宽度
        _OutlineWidthParams("Outline Width Params", Vector) = (0,1,0,1)
        // 定义轮廓线Z轴偏移量
        _OutlineZOffset("Outline Z Offset", Float) = 0
        // 定义屏幕空间偏移向量
        _ScreenOffset("Screen Offset", Vector) = (0,0,0,0)
        // 定义轮廓线颜色
        _OutlineColor("Outline Color", Color) = (0,0,0,1)
        // 定义轮廓线颜色2
        _OutlineColor2("Outline Color 2", Color) = (0,0,0,1)
        // 定义轮廓线颜色3
        _OutlineColor3("Outline Color 3", Color) = (0,0,0,1)
        // 定义轮廓线颜色4
        _OutlineColor4("Outline Color 4", Color) = (0,0,0,1)
        // 定义轮廓线颜色5
        _OutlineColor5("Outline Color 5", Color) = (0,0,0,1)
    }

    // Subshader块：包含多个Pass（渲染通道）
    Subshader
    {
        // Tags块：定义渲染管线的标签
        Tags
        {
            "RenderType" = "Opaque" // 渲染类型为不透明
            "RenderPipeline" = "UniversalPipeline" // 使用Universal渲染管线
            "UniversalMaterialType" = "Lit" // 材质类型为受光照影响
            "IgnoreProjector" = "True" // 忽略投影器影响
        }

        // 第一个Pass：前向渲染通道
        Pass
        {
            Name "Forward" // 通道名称
            Tags {"LightMode" = "UniversalForward"} // 光照模式为前向渲染

            // 渲染状态设置
            Cull[_Cull] // 使用属性中定义的剔除模式
            ZWrite On // 开启深度写入
            Blend[_SrcBlend][_DstBlend] // 使用属性中定义的混合模式

            // HLSL程序开始
            HLSLPROGRAM

            // 通用渲染管线关键字编译指令
            #pragma multi_compile _ _MAIN_LIGHT_SHADOWS _MAIN_LIGHT_SHADOWS_CASCADE _MAIN_LIGHT_SHADOWS_SCREEN
            #pragma multi_compile _ _ADDITIONAL_LIGHTS_VERTEX _ADDITIONAL_LIGHTS
            #pragma multi_compile_fragment _ _ADDITIONAL_LIGHT_SHADOWS
            #pragma multi_compile_fragment _ _REFLECTION_PROBE_BLENDING
            #pragma multi_compile_fragment _ _REFLECTION_PROBE_BOX_PROJECTION
            #pragma multi_compile_fragment _ _SHADOWS_SOFT
            #pragma multi_compile_fragment _ _SCREEN_SPACE_OCCLUSION
            #pragma multi_compile_fragment _ _DBUFFER_MRT1 _DBUFFER_MRT2 _DBUFFER_MRT3
            #pragma multi_compile_fragment _ _LIGHT_LAYERS
            #pragma multi_compile_fragment _ _LIGHT_COOKIES
            #pragma multi_compile _ _FORWARD_PLUS
            #pragma multi_compile_fragment _ _WRITE_RENDERING_LAYERS

            // Unity定义的关键字编译指令
            #pragma multi_compile _ LIGHTMAP_SHADOW_MIXING
            #pragma multi_compile _ SHADOWS_SHADOWMASK
            #pragma multi_compile _ DIRLIGHTMAP_COMBINED
            #pragma multi_compile _ LIGHTMAP_ON
            #pragma multi_compile _ DYNAMICLIGHTMAP_ON
            #pragma multi_compile_fragment _ LOD_FADE_CROSSFADE
            #pragma multi_compile_fog
            #pragma multi_compile_fragment _ DEBUG_DISPLAY

            // 本地着色器特性，根据材质设置启用或禁用特定功能
            #pragma shader_feature_local_fragment _DOUBLE_SIDED
            #pragma shader_feature_local_fragment _EMISSION
            #pragma shader_feature_local_fragment _NORMAL_MAP
            #pragma shader_feature_local_fragment _IS_FACE
            #pragma shader_feature_local_fragment _SPECULAR
            #pragma shader_feature_local_fragment _RIM

            // 指定顶点和片段着色器函数
            #pragma vertex ForwardPassVertex
            #pragma fragment ForwardPassFragment

            // 包含自定义HLSL文件
            #include "ToonInput.hlsl"
            #include "ToonForwardPass.hlsl"

            // HLSL程序结束
            ENDHLSL
        }

        // 第二个Pass：阴影投射通道
        Pass
        {
            Name "ShadowCaster" // 通道名称
            Tags{"LightMode" = "ShadowCaster"} // 光照模式为阴影投射

            // 渲染状态设置
            ZWrite On // 开启深度写入
            ZTest LEqual // 深度测试条件：小于等于
            ColorMask 0 // 颜色掩码为0，不写入任何颜色
            Cull[_Cull] // 使用属性中定义的剔除模式

            // HLSL程序开始
            HLSLPROGRAM

            // 指定顶点和片段着色器函数
            #pragma vertex ShadowPassVertex
            #pragma fragment ShadowPassFragment

            // 使用URP内置的阴影投射通道
            #include "Packages/com.unity.render-pipelines.universal/Shaders/LitInput.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/Shaders/ShadowCasterPass.hlsl"

            // HLSL程序结束
            ENDHLSL
        }

        // 第三个Pass：深度Only通道
        Pass
        {
            Name "DepthOnly" // 通道名称
            Tags{"LightMode" = "DepthOnly"} // 光照模式为深度Only

            // 渲染状态设置
            ZWrite On // 开启深度写入
            ColorMask R // 颜色掩码为R，只写入红色通道（深度）
            Cull[_Cull] // 使用属性中定义的剔除模式

            // HLSL程序开始
            HLSLPROGRAM

            // 指定顶点和片段着色器函数
            #pragma vertex DepthOnlyVertex
            #pragma fragment DepthOnlyFragment

            // 使用URP内置的深度Only通道
            #include "Packages/com.unity.render-pipelines.universal/Shaders/LitInput.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/Shaders/DepthOnlyPass.hlsl"

            // HLSL程序结束
            ENDHLSL
        }

        // 第四个Pass：深度法线通道
        Pass
        {
            Name "DepthNormals" // 通道名称
            Tags{"LightMode" = "DepthNormals"} // 光照模式为深度法线

            // 渲染状态设置
            ZWrite On // 开启深度写入
            Cull[_Cull] // 使用属性中定义的剔除模式

            // HLSL程序开始
            HLSLPROGRAM

            // 指定顶点和片段着色器函数
            #pragma vertex DepthNormalsVertex
            #pragma fragment DepthNormalsFragment

            // 使用URP内置的深度法线通道
            #include "Packages/com.unity.render-pipelines.universal/Shaders/LitInput.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/Shaders/LitDepthNormalsPass.hlsl"

            // HLSL程序结束
            ENDHLSL
        }

        // 第五个Pass：轮廓线通道
        Pass
        {
            Name "Outline" // 通道名称
            Tags {"LightMode" = "SRPDefaultUnlit"} // 光照模式为默认无光照

            Cull Front // 正面剔除，只渲染背面（用于轮廓线效果）

            // HLSL程序开始
            HLSLPROGRAM

            // 指定顶点和片段着色器函数
            #pragma vertex OutlinePassVertex
            #pragma fragment OutlinePassFragment

            // 包含自定义HLSL文件
            #include "ToonInput.hlsl"
            #include "ToonOutlinePass.hlsl"

            // HLSL程序结束
            ENDHLSL
        }
    }
}
~~~

