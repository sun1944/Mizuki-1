---
title: Socket网络通信
published: 2025-07-14
description: "Socket网络通信知识讲解。"
image: "https://typorytp.oss-cn-hangzhou.aliyuncs.com/titleshader01.png"
tags: ["Shader"]
category: Unity Shader
draft: false
---

# Unity Shader基础

## 光照基础

游戏中的光照通过光照的数学模型去模拟光照的效果。

### Lambert光照模型

当光线照射到表面粗糙的物体，就会向各个方向反射等强度的光，称之为漫反射现象。

漫反射 = 光源颜色 x 模型基础颜色 x max（0，物体表面法线 · 指向光源的单位方向）。
$$
I 
diffuse
​
 =k 
d
​
 ⋅I 
light
​
 ⋅max(0,n⋅l)
$$

#### 逐顶点光照

~~~glsl
Shader "Unlit/Lambert_01"
{
    Properties
    {
        _MainColor("MainColor",Color) = (1,1,1,1)
    }
    
    SubShader
    {
        Pass
        {
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"
            #include "UnityLightingCommon.cginc"

            struct v2f
            {
                float4 pos: SV_POSITION;
                fixed4 dif : COLOR0;
            };

            fixed4 _MainColor;

            v2f vert(appdata_base v)
            {
                v2f o;
                //变换到裁切空间
                o.pos = UnityObjectToClipPos(v.vertex);
                //计算表面法线
                float3 n = UnityObjectToWorldNormal(v.normal);
                //指向光源单位向量
                fixed3 l = normalize(_WorldSpaceLightPos0.xyz);
                //公式计算
                fixed ndotl = dot(n,l);
                o.dif = _LightColor0 * _MainColor * saturate(ndotl);

                return o;
            }
            
            fixed4 frag(v2f i) : SV_Target
            {
                return i.dif;
            }
            ENDCG
        }
    }
    
    
}

~~~

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20250721163422256.png" alt="image-20250721163422256" style="zoom:50%;" />

很明显发现没有光照的部分是完全黑的。

#### 逐像素光照

~~~glsl
Shader "Unlit/Lambert_02"
{
    Properties  // 材质属性面板中显示的参数
    {
        // 定义一个名为"Main Color"的颜色属性，默认值为白色
        _MainColor("Main Color", Color) = (1,1,1,1)
    }
    
    SubShader  // 子着色器块
    {
        
        Pass  // 渲染通道
        {
            CGPROGRAM  // 开始CG代码块
            
            // 声明顶点着色器函数名为vert
            #pragma vertex vert
            // 声明片元着色器函数名为frag
            #pragma fragment frag
            
            // 包含Unity内置的CG函数库
            #include "UnityCG.cginc"
            // 包含Unity内置的光照相关变量（如_LightColor0）
            #include "UnityLightingCommon.cginc"

            // 定义顶点着色器输入结构
            struct appdata
            {
                float4 vertex : POSITION;  // 顶点位置（模型空间）
                float3 normal : NORMAL;    // 顶点法线（模型空间）
            };

            // 定义顶点到片元的数据传递结构
            struct v2f
            {
                float4 pos : SV_POSITION;       // 裁剪空间位置（必须）
                float3 worldNormal : TEXCOORD0;  // 世界空间法线（自定义语义）
                float3 worldPos : TEXCOORD1;     // 世界空间位置（自定义语义）
            };

            // 声明在Properties中定义的_MainColor变量
            fixed4 _MainColor;

            // 顶点着色器函数
            v2f vert (appdata v)
            {
                v2f o;  // 创建输出结构实例
                
                // 将顶点位置从模型空间转换到裁剪空间
                o.pos = UnityObjectToClipPos(v.vertex);
                
                // 将法线从模型空间转换到世界空间并归一化
                // UnityObjectToWorldNormal内部处理了非统一缩放
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                
                // 将顶点位置从模型空间转换到世界空间
                // unity_ObjectToWorld是模型到世界的变换矩阵
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                
                return o;  // 返回输出结构
            }
            
            // 片元着色器函数
            fixed4 frag (v2f i) : SV_Target  // 输出到渲染目标
            {
                // 对插值后的世界法线进行重新归一化（重要！）
                // 因为顶点到片元的插值可能导致法线长度不为1
                float3 n = normalize(i.worldNormal);
                
                // 获取光源方向（对于平行光，_WorldSpaceLightPos0.xyz就是方向）
                // 对于点光源需要计算：光源方向 = 光源位置 - 当前片元位置
                float3 l = normalize(_WorldSpaceLightPos0.xyz);
                
                // 计算法线和光源方向的点积（余弦值）
                // saturate函数将结果限制在0-1范围内
                half ndotl = saturate(dot(n, l));
                
                // 计算最终颜色：
                // 光源颜色 * 材质基础颜色 * 光照强度
                fixed4 col = _LightColor0 * _MainColor * ndotl;
                
                return col;  // 返回最终片元颜色
            }
            ENDCG  // 结束CG代码块
        }
    }
}
~~~

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20250721220802308.png" alt="image-20250721220802308" style="zoom:50%;" />

### Half Lambert光照模型

$$
I 
HalfLambert
​
 =k 
d
​
 ⋅I 
light
​
 ⋅(0.5⋅(n⋅l)+0.5)
$$

为了解决兰伯特光照导致的背光面黑色问题，valve公司在《半条命》中对Lambert光照模型简单修改，相当于将点积结果从[-1,1]映射到[0,1]。

~~~glsl
Shader "Custom/Half-Lambert"
{
    Properties
    {
        _MainColor ("Main Color", Color) = (1, 1, 1, 1)
    }
    SubShader
    {
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"
            #include "Lighting.cginc"

            struct v2f
            {
                float4 pos : SV_POSITION;
                fixed4 dif : COLOR0;
            };

            fixed4 _MainColor;

            v2f vert (appdata_base v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);

                float3 n = UnityObjectToWorldNormal(v.normal);
                n= normalize(n);
                fixed3 l = normalize(_WorldSpaceLightPos0).xyz;
                
                fixed ndotl = dot(n, l);
                o.dif = _LightColor0 * _MainColor * (0.5 * ndotl + 0.5);

                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                return i.dif;
            }
            ENDCG
        }
    }
}

~~~

可以发现背光面已经不是完全黑色了。

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20250721210233237.png" alt="image-20250721210233237" style="zoom:50%;" />

### Phong光照模型

Phong光照模型适合模拟表面光滑的物体。例如金属、陶瓷、塑料等等。
$$
SurfaceColor
​
 =C
 Ambient 
 ​
 +
 C
 Diffuse
 +
 C
 Specular
$$
即环境光、漫反射光、镜面反射光组成。镜面反射光的计算公式为
$$
C
specular
=
​
(
C
light
 ⋅
 M
 specular
​
)
saturate
(
v
 ⋅
 r
)
^
M
shininess
$$
镜面反射 =（灯光亮度 ⋅ 物体材质镜面反射颜色）⋅（视角方向 ⋅光线反射方向）^（材质光泽度）

#### 逐顶点光照

~~~glsl
Shader "Unlit/Phong"
{
    Properties
    {
        _MainColor("Main Color",Color) = (1,1,1,1) 
        //高光色
        _SpecularColor("Specular Color",Color) = (0,0,0,0)
        //光泽度 
        _Shininess("Shininess",Range(1,100)) = 1
        
    }
    
    SubShader
    {
        Pass
        {
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"
            #include "Lighting.cginc"

            struct v2f
            {
                float4 pos: SV_POSITION;
                fixed4 color : COLOR0;
            };

            fixed4 _MainColor;
            fixed4 _SpecularColor;
            half _Shininess;

            v2f vert (appdata_base v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                
                //计算公式各个变量
                // 1. 将顶点法线从对象空间转换到世界空间，并归一化
                float3 n = UnityObjectToWorldNormal(v.normal); // 使用Unity内置函数转换法线到世界空间
                n = normalize(n); 
                // 2. 获取光源方向（世界空间下），并归一化
                fixed3 l = normalize(_WorldSpaceLightPos0.xyz); // _WorldSpaceLightPos0是Unity内置变量，表示当前主光源方向
                // 3. 计算视线方向（从顶点指向摄像机）并归一化
                fixed3 view = normalize(WorldSpaceViewDir(v.vertex)); // WorldSpaceViewDir用于获取世界空间下的视图方向

                //漫反射
                fixed ndotl = saturate(dot(n,l));
                fixed4 dif = _LightColor0 * _MainColor * ndotl;

                //镜面反射
                float3 ref = reflect(-l,n);//计算
                ref = normalize(ref);
                fixed rdotv = saturate(dot(ref,view));
                fixed4 spec = _LightColor0 * _SpecularColor * pow(rdotv, _Shininess);

                //环境光+漫反射+镜面反射
                o.color = unity_AmbientSky + dif + spec;

                return o;
                
            }

            fixed4 frag (v2f i) : SV_Target
            {
                return i.color;
            }
            ENDCG
        }
        
        
    }
    
    
    
    
}

~~~

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20250722143531745.png" alt="image-20250722143531745" style="zoom:50%;" />

#### 逐像素光照

~~~glsl
Shader "Unlit/Phong_02"
{
    Properties
    {
        _MainColor("Main Color",Color) = (1,1,1,1) 
        //高光色
        _SpecularColor("Specular Color",Color) = (0,0,0,0)
        //光泽度 
        _Shininess("Shininess",Range(1,100)) = 1
    }
    
    SubShader
    {
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"
            #include "Lighting.cginc"

            struct v2f
            {
                float4 pos : SV_POSITION;
                float3 normal : TEXCOORD0;
                float4 vertex : TEXCOORD1;
            };

            fixed4 _MainColor;
            fixed4 _SpecularColor;
            half _Shininess;

            v2f vert (appdata_base v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.normal = v.normal;
                o.vertex = v.vertex;
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                //计算公式各个变量
                float3 n = UnityObjectToWorldNormal(i.normal);
                n = normalize(n);
                fixed3 l = normalize(_WorldSpaceLightPos0.xyz);
                fixed3 view = normalize(WorldSpaceViewDir(i.vertex));
                
                //漫反射
                fixed ndotl = saturate(dot(n,l));
                fixed dif = _LightColor0 * _MainColor * ndotl;

                //镜面反射
                float3 ref = reflect(-l,n);
                ref = normalize(ref);
                fixed rdotv = saturate(dot(ref,view));
                fixed4 spec = _LightColor0 * _SpecularColor * pow(rdotv,_Shininess);
                
                //环境光+漫反射+镜面反射
                return unity_AmbientSky + dif + spec;
            }
            
            ENDCG
        }
    }
}

~~~

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20250722145601868.png" alt="image-20250722145601868" style="zoom:50%;" />

### Blinn-Phong光照模型

相比于Phong光照模型，只是使用半角向量h替代r，h为视角方向v和灯光方向l的角平分线方向,n是物体表面法线。
$$
h
=
normalize
(
v
+
l
)
$$

$$
C
specular
=
​
(
C
light
 ⋅
 M
 specular
​
)
saturate
(
 n
 ⋅
 h
)
^
M
shininess
$$

~~~glsl
Shader "Unlit/Blinn-Phong"
{
    Properties
    {
        _MainColor("Main Color",Color) = (1,1,1,1) 
        //高光色
        _SpecularColor("Specular Color",Color) = (0,0,0,0)
        //光泽度 
        _Shininess("Shininess",Range(1,100)) = 1
    }
    
    SubShader
    {
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"
            #include "Lighting.cginc"

            struct v2f
            {
                float4 pos : SV_POSITION;
                float3 normal : TEXCOORD0;
                float4 vertex : TEXCOORD1;
            };

            fixed4 _MainColor;
            fixed4 _SpecularColor;
            half _Shininess;

            v2f vert (appdata_base v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.normal = v.normal;
                o.vertex = v.vertex;
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                //计算公式各个变量
                float3 n = UnityObjectToWorldNormal(i.normal);
                n = normalize(n);
                fixed3 l = normalize(_WorldSpaceLightPos0.xyz);
                fixed3 view = normalize(WorldSpaceViewDir(i.vertex));
                
                //漫反射
                fixed ndotl = saturate(dot(n,l));
                fixed dif = _LightColor0 * _MainColor * ndotl;

                //镜面反射
                fixed3 h = normalize(l + view);
                fixed ndoth = saturate(dot(n,h));
                fixed4 spec = _LightColor0 * _SpecularColor * pow(ndoth,_Shininess);
                
                //环境光+漫反射+镜面反射
                return unity_AmbientSky + dif + spec;
            }
            
            ENDCG
        }
    }
}

~~~

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20250722151916521.png" alt="image-20250722151916521" style="zoom:50%;" />

## 纹理基础





## 透明效果

### 透明度测试

简单来说就是比较纹理alpha和阈值，不符合要求的片元直接丢掉不渲染。

~~~glsl
// 定义Shader名称和路径
Shader "Unlit/AlphaTest"
{
    // 属性块：在材质面板中显示的参数
    Properties
	{
		_MainTex("MainTex", 2D) = "white" {}          // 主纹理属性
		_AlphaTest("Alpha Test",Range(0,1)) = 0        // Alpha测试阈值（0-1范围）
	}
    
    // 子着色器块（可包含多个Pass）
    SubShader
    {
    	// 标签：设置渲染队列和类型
    	Tags
    	{
    		"Queue" = "AlphaTest"                  // 使用Alpha测试渲染队列
    		"RenderType" = "TransparentCutout"      // 渲染类型为透明镂空
    		"IgnoreProjector" = "True"             // 忽略投影器影响
    	}
    	
    	// 单个渲染通道
    	Pass
    	{
    		Tags{"LightMode" = "ForwardBase"}      // 使用前向渲染基础光照模式
    		
    		// 关闭几何体剔除（显示双面）
    		Cull Off
    		//开启多重采样抗锯齿
    		AlphaToMask On
    		
    		// 开始CG代码块
    		CGPROGRAM
    		// 声明顶点和片段着色器函数
    		#pragma vertex vert
    		#pragma fragment frag
    		
    		// 包含Unity内置CG文件
    		#include "UnityCG.cginc"
    		#include "UnityLightingCommon.cginc"  // 包含光照相关函数

		    // 定义顶点着色器到片段着色器的数据结构
		    struct v2f
		    {
			    float4 pos : SV_POSITION;       // 裁剪空间位置（必须）
    			float4 worldPos : TEXCOORD0;     // 世界空间位置（寄存器0）
    			float2 texcoord : TEXCOORD1;     // 纹理坐标（寄存器1）
    			float3 worldNormal : TEXCOORD2;  // 世界空间法线（寄存器2）
		    };

    		// 声明属性变量
    		sampler2D _MainTex;                  // 主纹理采样器
    		float4 _MainTex_ST;                  // 纹理的缩放和平移参数
    		fixed _AlphaTest;                   // Alpha测试阈值

    		// 顶点着色器函数
    		v2f vert (appdata_base v)            // 使用基础顶点数据结构
    		{
    			v2f o;                           // 声明输出结构
    			
    			// 将顶点位置从对象空间转换到裁剪空间
    			o.pos = UnityObjectToClipPos(v.vertex);
    			
    			// 将顶点位置转换到世界空间
    			o.worldPos = mul(unity_ObjectToWorld, v.vertex);
    			
    			// 应用纹理缩放和平移变换后存储UV坐标
    			o.texcoord = TRANSFORM_TEX(v.texcoord, _MainTex);

    			// 将法线从对象空间转换到世界空间并归一化
    			float3 worldNormal = UnityObjectToWorldNormal(v.normal);
    			o.worldNormal = normalize(worldNormal);

    			return o;  // 返回处理后的顶点数据
    		}

    		// 片段着色器函数
    		fixed4 frag (v2f i) : SV_Target      // 输出到渲染目标
    		{
    			// 计算当前片段指向光源的方向（世界空间）
    			float3 worldLight = UnityWorldSpaceLightDir(i.worldPos.xyz);
    			worldLight = normalize(worldLight);  // 归一化光源方向
    			
    			// 计算法线与光源方向的点积（漫反射系数）
    			fixed NdotL = saturate(dot(i.worldNormal, worldLight));
    			
    			// 采样主纹理获取颜色
    			fixed4 color =  tex2D(_MainTex, i.texcoord);
    			
    			// Alpha测试：当纹理alpha值小于阈值时丢弃片段
    			clip(color.a - _AlphaTest);
    			
    			// 应用漫反射光照：颜色 = 纹理颜色 × 光照强度 × 光源颜色
    			color.rgb *= NdotL * _LightColor0.rgb;
    			
    			// 添加环境光（天空盒环境光）
    			color.rgb += unity_AmbientSky.rgb;

    			return color;  // 返回最终颜色
    		}
    		
    		ENDCG  // 结束CG代码块
    	}
    }
}
~~~

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20250725142607216.png" alt="image-20250725142607216" style="zoom:50%;" />





### 透明度混合

#### 半透明物体正面渲染

透明物体通过深度测试之后禁止把深度值写入到深度缓存中，之后使用混合指令。

~~~glsl
Shader "Custom/Blend01"
{
    // 属性块：在材质面板中显示的参数
    Properties
    {
        // 主纹理：用于表面颜色和图案
        _MainTex ("MainTex", 2D) = "white" {}
        // 主颜色：RGBA颜色值，用于调整整体颜色和透明度
        _MainColor ("MainColor(RGB_A)", Color) = (1, 1, 1, 1)
    }
    
    SubShader
    {
        // 设置渲染标签
        Tags
        {
            "Queue" = "Transparent"       // 使用透明渲染队列（在几何体之后渲染）
            "RenderType" = "Transparent"  // 声明为透明物体类型
            "IgnoreProjector" = "True"    // 忽略投影器影响（透明物体通常不需要投影）
        }

        Pass
        {
            // 光照模式标签：前向渲染基础通道
            Tags{"LightMode" = "ForwardBase"}

            // 设置渲染状态
            ZWrite Off  // 关闭深度写入，允许透明物体相互混合
            // 设置混合模式：源(当前片元)使用自身Alpha，目标(已存在颜色)使用1-源Alpha
            Blend SrcAlpha OneMinusSrcAlpha

            CGPROGRAM
            // 声明顶点着色器函数
            #pragma vertex vert
            // 声明片元着色器函数
            #pragma fragment frag
            // 编译前向渲染基础通道所需的多重编译变体
            #pragma multi_compile_fwdbasse
            // 包含UnityCG核心库（矩阵变换、基本函数）
            #include "UnityCG.cginc"
            // 包含光照计算相关库
            #include "UnityLightingCommon.cginc"

            // 定义顶点着色器输出/片元着色器输入的结构体
            struct v2f
            {
                float4 pos : SV_POSITION;     // 裁剪空间位置（必须）
                float4 worldPos : TEXCOORD0;  // 世界空间位置（用于光照计算）
                float2 texcoord : TEXCOORD1;   // 纹理坐标（用于采样贴图）
                float3 worldNromal : TEXCOORD2; // 世界空间法线（用于光照计算）
            };

            // 声明属性对应的变量
            sampler2D _MainTex;     // 主纹理采样器
            float4 _MainTex_ST;     // 纹理的缩放(Scale)和偏移(Translation)值
            fixed4 _MainColor;      // 主颜色属性（RGBA）

            // 顶点着色器：处理每个顶点
            v2f vert (appdata_base v)
            {
                v2f o;
                // 将顶点位置从对象空间转换到裁剪空间
                o.pos = UnityObjectToClipPos(v.vertex);
                // 将顶点位置转换到世界空间（用于光照计算）
                o.worldPos = mul(unity_ObjectToWorld, v.vertex);
                // 应用纹理缩放/偏移到纹理坐标
                o.texcoord = TRANSFORM_TEX(v.texcoord, _MainTex);

                // 将法线从对象空间转换到世界空间
                float3 worldNromal = UnityObjectToWorldNormal(v.normal);
                // 归一化法线（确保长度为1）
                o.worldNromal = normalize(worldNromal);

                return o;
            }

            // 片元着色器：处理每个像素
            fixed4 frag (v2f i) : SV_Target
            {
                // 获取当前片元指向光源的方向向量（世界空间）
                float3 worldLight = UnityWorldSpaceLightDir(i.worldPos.xyz);
                // 归一化光源方向向量（确保长度为1）
                worldLight = normalize(worldLight);

                // 计算法线与光源方向的点积（漫反射强度），并限制在[0,1]范围
                fixed NdotL = saturate(dot(i.worldNromal, worldLight));

                // 采样主纹理获取颜色
                fixed4 color = tex2D(_MainTex, i.texcoord);

                // 光照计算：
                // 1. 纹理颜色 × 主颜色(RGB) 
                // 2. 乘以漫反射强度(NdotL)
                // 3. 乘以光源颜色
                color.rgb *= _MainColor.rgb * NdotL * _LightColor0;
                
                // 添加环境光（天空盒环境光）
                color.rgb += unity_AmbientSky;

                // 通过_MainColor的alpha分量控制整体透明度
                // 将纹理的alpha与主颜色的alpha相乘
                color.a *= _MainColor.a;

                // 返回最终颜色（包含透明度）
                return color;
            }
            ENDCG
        }
    }
    // 备选着色器（当主着色器不支持时使用）
    FallBack "Diffuse"
}
~~~

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20250724223652183.png" alt="image-20250724223652183" style="zoom:50%;" />

#### 半透明物体双面渲染

使用两个Pass，第一个Pass使用Cull Front剔除正面，先渲染物体的背面，在第二个Pass中使用Cull Back剔除背面，只渲染正面。因为使用了两个Pass，实际执行了两次渲染。

~~~glsl
Shader "Custom/Blend01"
{
    // 属性块：在材质面板中显示的参数
    Properties
    {
        // 主纹理：用于表面颜色和图案
        _MainTex ("MainTex", 2D) = "white" {}
        // 主颜色：RGBA颜色值，用于调整整体颜色和透明度
        _MainColor ("MainColor(RGB_A)", Color) = (1, 1, 1, 1)
    }
    
    SubShader
    {
        // 设置渲染标签
        Tags
        {
            "Queue" = "Transparent"       // 使用透明渲染队列（在几何体之后渲染）
            "RenderType" = "Transparent"  // 声明为透明物体类型
            "IgnoreProjector" = "True"    // 忽略投影器影响（透明物体通常不需要投影）
        }

        //--------------渲染背面-----------------
        Pass
        {
            // 光照模式标签：前向渲染基础通道
            Tags{"LightMode" = "ForwardBase"}

            //开启正面剔除
            Cull Front
            ZWrite Off  // 关闭深度写入，允许透明物体相互混合
            // 设置混合模式：源(当前片元)使用自身Alpha，目标(已存在颜色)使用1-源Alpha
            Blend SrcAlpha OneMinusSrcAlpha

            CGPROGRAM
            // 声明顶点着色器函数
            #pragma vertex vert
            // 声明片元着色器函数
            #pragma fragment frag
            // 编译前向渲染基础通道所需的多重编译变体
            #pragma multi_compile_fwdbasse
            // 包含UnityCG核心库（矩阵变换、基本函数）
            #include "UnityCG.cginc"
            // 包含光照计算相关库
            #include "UnityLightingCommon.cginc"

            // 定义顶点着色器输出/片元着色器输入的结构体
            struct v2f
            {
                float4 pos : SV_POSITION;     // 裁剪空间位置（必须）
                float4 worldPos : TEXCOORD0;  // 世界空间位置（用于光照计算）
                float2 texcoord : TEXCOORD1;   // 纹理坐标（用于采样贴图）
                float3 worldNromal : TEXCOORD2; // 世界空间法线（用于光照计算）
            };

            // 声明属性对应的变量
            sampler2D _MainTex;     // 主纹理采样器
            float4 _MainTex_ST;     // 纹理的缩放(Scale)和偏移(Translation)值
            fixed4 _MainColor;      // 主颜色属性（RGBA）

            // 顶点着色器：处理每个顶点
            v2f vert (appdata_base v)
            {
                v2f o;
                // 将顶点位置从对象空间转换到裁剪空间
                o.pos = UnityObjectToClipPos(v.vertex);
                // 将顶点位置转换到世界空间（用于光照计算）
                o.worldPos = mul(unity_ObjectToWorld, v.vertex);
                // 应用纹理缩放/偏移到纹理坐标
                o.texcoord = TRANSFORM_TEX(v.texcoord, _MainTex);

                // 将法线从对象空间转换到世界空间
                float3 worldNromal = UnityObjectToWorldNormal(v.normal);
                // 归一化法线（确保长度为1）
                o.worldNromal = normalize(worldNromal);

                return o;
            }

            // 片元着色器：处理每个像素
            fixed4 frag (v2f i) : SV_Target
            {
                // 获取当前片元指向光源的方向向量（世界空间）
                float3 worldLight = UnityWorldSpaceLightDir(i.worldPos.xyz);
                // 归一化光源方向向量（确保长度为1）
                worldLight = normalize(worldLight);

                // 计算法线与光源方向的点积（漫反射强度），并限制在[0,1]范围
                fixed NdotL = saturate(dot(i.worldNromal, worldLight));

                // 采样主纹理获取颜色
                fixed4 color = tex2D(_MainTex, i.texcoord);

                // 光照计算：
                // 1. 纹理颜色 × 主颜色(RGB) 
                // 2. 乘以漫反射强度(NdotL)
                // 3. 乘以光源颜色
                color.rgb *= _MainColor.rgb * NdotL * _LightColor0;
                
                // 添加环境光（天空盒环境光）
                color.rgb += unity_AmbientSky;

                // 通过_MainColor的alpha分量控制整体透明度
                // 将纹理的alpha与主颜色的alpha相乘
                color.a *= _MainColor.a;

                // 返回最终颜色（包含透明度）
                return color;
            }
            ENDCG
            
        }

        //--------------渲染正面-----------------
        Pass
        { 
            Tags{"LightMode" = "ForwardBase"}
            //开启背面剔除
            Cull Back
            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha
            
            CGPROGRAM
            // 声明顶点着色器函数
            #pragma vertex vert
            // 声明片元着色器函数
            #pragma fragment frag
            // 编译前向渲染基础通道所需的多重编译变体
            #pragma multi_compile_fwdbasse
            // 包含UnityCG核心库（矩阵变换、基本函数）
            #include "UnityCG.cginc"
            // 包含光照计算相关库
            #include "UnityLightingCommon.cginc"

            // 定义顶点着色器输出/片元着色器输入的结构体
            struct v2f
            {
                float4 pos : SV_POSITION;     // 裁剪空间位置（必须）
                float4 worldPos : TEXCOORD0;  // 世界空间位置（用于光照计算）
                float2 texcoord : TEXCOORD1;   // 纹理坐标（用于采样贴图）
                float3 worldNromal : TEXCOORD2; // 世界空间法线（用于光照计算）
            };

            // 声明属性对应的变量
            sampler2D _MainTex;     // 主纹理采样器
            float4 _MainTex_ST;     // 纹理的缩放(Scale)和偏移(Translation)值
            fixed4 _MainColor;      // 主颜色属性（RGBA）

            // 顶点着色器：处理每个顶点
            v2f vert (appdata_base v)
            {
                v2f o;
                // 将顶点位置从对象空间转换到裁剪空间
                o.pos = UnityObjectToClipPos(v.vertex);
                // 将顶点位置转换到世界空间（用于光照计算）
                o.worldPos = mul(unity_ObjectToWorld, v.vertex);
                // 应用纹理缩放/偏移到纹理坐标
                o.texcoord = TRANSFORM_TEX(v.texcoord, _MainTex);

                // 将法线从对象空间转换到世界空间
                float3 worldNromal = UnityObjectToWorldNormal(v.normal);
                // 归一化法线（确保长度为1）
                o.worldNromal = normalize(worldNromal);

                return o;
            }

            // 片元着色器：处理每个像素
            fixed4 frag (v2f i) : SV_Target
            {
                // 获取当前片元指向光源的方向向量（世界空间）
                float3 worldLight = UnityWorldSpaceLightDir(i.worldPos.xyz);
                // 归一化光源方向向量（确保长度为1）
                worldLight = normalize(worldLight);

                // 计算法线与光源方向的点积（漫反射强度），并限制在[0,1]范围
                fixed NdotL = saturate(dot(i.worldNromal, worldLight));

                // 采样主纹理获取颜色
                fixed4 color = tex2D(_MainTex, i.texcoord);

                // 光照计算：
                // 1. 纹理颜色 × 主颜色(RGB) 
                // 2. 乘以漫反射强度(NdotL)
                // 3. 乘以光源颜色
                color.rgb *= _MainColor.rgb * NdotL * _LightColor0;
                
                // 添加环境光（天空盒环境光）
                color.rgb += unity_AmbientSky;

                // 通过_MainColor的alpha分量控制整体透明度
                // 将纹理的alpha与主颜色的alpha相乘
                color.a *= _MainColor.a;

                // 返回最终颜色（包含透明度）
                return color;
            }
            ENDCG
        }
    }
    // 备选着色器（当主着色器不支持时使用）
    FallBack "Diffuse"
}
~~~

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20250724224049226.png" alt="image-20250724224049226" style="zoom:50%;" />

### 模板测试透明效果

~~~glsl
Shader "Unlit/Stencil Test A"
{
    SubShader
    {
        // 在渲染队列中提前渲染 (比普通几何体早)
        Tags {"Queue" = "Geometry -1"}
        
        Pass
        {
            // 模板缓冲配置
            Stencil
            {
                Ref 1          // 参考值：1
                Comp Always    // 总是通过模板测试
                Pass Replace   // 通过后用参考值(1)替换当前模板值
            }
            
            // 禁止写入任何颜色通道
            ColorMask 0
            // 关闭深度写入 (不影响深度缓冲)
            ZWrite Off
            
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            // 顶点着色器：仅处理位置变换
            float4 vert(in float4 vertex: POSITION) : SV_POSITION
            {
                // 将顶点从对象空间变换到裁剪空间
                float4 pos = UnityObjectToClipPos(vertex);
                return pos;
            }

            // 片段着色器：输出透明黑色
            void frag (out fixed4 color : SV_Target)
            {
                color = fixed4(0,0,0,0); // RGBA全0 = 完全透明
            }
            
            ENDCG
        }
    }
}
~~~

~~~glsl
Shader "Unlit/Stencil Test B"
{
    // 定义可在材质面板调整的属性
    Properties
    {
        _MainColor ("Main Color" , Color) = (1,1,1,1)  // 基础颜色
        _MainTex("Main Tex",2D) =  "white"{}           // 主纹理
    }
    
    SubShader
    {
        // 使用标准几何体渲染队列
        Tags{"Queue" = "Geometry"}
        
        Pass
        {
            // 使用前向渲染基础通道
            Tags{"LightMode" = "ForwardBase"}
            
        // 模板缓冲配置
        Stencil
        {
            Ref 1             // 参考值：1
            Comp NotEqual     // 仅当模板值 ≠ 1时通过测试
            Pass Replace      // 通过后用参考值(1)替换当前值
        }
        
        CGPROGRAM
        #pragma vertex vert
        #pragma fragment frag
        #include "UnityCG.cginc"          // 包含Unity CG函数
        #include "UnityLightingCommon.cginc" // 包含光照函数

        // 顶点到片段的数据结构
        struct v2f
        {
            float4 pos : SV_POSITION;     // 裁剪空间位置
            float4 worldPos : TEXCOORD0;   // 世界空间位置
            float3 worldNormal : TEXCOORD1; // 世界空间法线
            float2 texcoord : TEXCOORD2;   // 纹理坐标
        };

        // 声明属性对应的变量
        sampler2D _MainTex;         // 主纹理采样器
        float4 _MainTex_ST;         // 纹理缩放偏移参数
        fixed4 _MainColor;          // 基础颜色

        // 顶点着色器
        v2f vert (appdata_base v)  // 使用基础顶点数据结构
        {
            v2f o;
            // 变换顶点位置到裁剪空间
            o.pos = UnityObjectToClipPos(v.vertex);
            // 变换法线到世界空间 (注意：这里应是v.vertex而非v.normal)
            o.worldPos = mul(unity_ObjectToWorld,v.vertex);
            
            // 计算世界空间法线并归一化
            float3 worldNormal = UnityObjectToWorldNormal(v.normal);
            o.worldNormal =normalize(worldNormal);
            // 应用纹理缩放偏移
            o.texcoord = TRANSFORM_TEX(v.texcoord,_MainTex);

            return o;
        }

        // 片段着色器
        fixed4 frag(v2f i) : SV_Target
        {
            // 获取世界空间光源方向
            float3 worldLight = UnityWorldSpaceLightDir(i.worldPos.xyz);
            worldLight = normalize(worldLight);

            // 计算法线与光源的点积 (漫反射系数)
            fixed NdotL = saturate(dot(i.worldNormal,worldLight));

            // 采样纹理
            fixed4 color = tex2D(_MainTex,i.texcoord);
            // 组合颜色：纹理色 × 基础色 × 漫反射 × 光源色
            color.rgb *= _MainColor * NdotL * _LightColor0.rgb;
            // 添加环境光
            color.rgb += unity_AmbientSky.rgb;

            return color;
        }
        ENDCG
        }
    }
}
~~~

可以透过物体A看见物体B后面的东西

<img src="https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20250725162626339.png" alt="image-20250725162626339" style="zoom:50%;" />