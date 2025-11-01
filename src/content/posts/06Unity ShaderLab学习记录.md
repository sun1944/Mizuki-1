---
title: ShaderLab入门
published: 2024-11-14
description: "ShaderLab基础语法。"
image: "https://typorytp.oss-cn-hangzhou.aliyuncs.com/titleshaderlab01.png"
tags: ["Shader"]
category: Unity Shader
draft: false
---

# ShaderLab语法基础

### ShaderLab组织结构

通常情况下一个Shader的大致结构如下

~~~glsl
//custom是材质选择shader时下拉列表的名称，Test1是shader的名称
Shader "Custom/Test1"
{
   Properties
   {
      //开放到材质面板的属性
   }
   SubShader
   {
      //顶点-片段着色器
      //或者表面着色器
      //或者固定函数着色器
      
      //如果这是一个顶点-片段着色器，则每个SubShader可以嵌套至少一个Pass，每个Pass会依次执行
      //如果是一个表面着色器，则将不会使用Pass，系统在编译过程中会自动生成多个Pass
      Pass
      {
         
      }
      ...
   }
   SubShader
   {
      //其他子着色器，当前面的子着色器没有正常运行时，使用这个子着色器
   }
   ...
   //如果所有子着色器都无法正常运行，则执行回退命令，运行指定的一个基础着色器
   Fallback "Name"
}
~~~

ShaderLab的Properties所有数据类型如下：

~~~glsl
Shader "Custom/Test1"
{
    Properties
    {
        //浮点类型
        _MyFloat("Float Property", Float) = 1.0
        //范围类型
        _MyRange("Range Property", Range(0,1)) = 0.5
        //颜色类型
        _MyColoe("Color Property", Color) = (1,1,1,1)
        //向量类型
        _MyVector("Vector Property", Vector) = (1,1,1,1)
        //2D贴图类型（常用二维图片类型）
        _MyTexture("Texture Property", 2D) = "white" {}
        //立方体贴图类型（主要运用于光照）
        MyCube("Cube Property", Cube) = "" {}
        //3D贴图类型（不常用类型）
        _My3DTexture("3D Texture Property", 3D) = "" {}
    }
    SubShader
    {
        Pass
        {
            
        }
    }
    Fallback "Diffuse"
}
~~~

暴露到编辑器中就是这个样子。

![image-20241112151115725](https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20241112151115725.png)



## 顶点-片段着色器

### 简单案例

我们先定义一个最简单的顶点-片段着色器，然后详细解释每一句的意思。

~~~glsl
Shader "Custom/Test1"
{
   SubShader
   {
      Pass{
         //嵌入指定的CG程序，放在CGPROGRAM和ENDCG之间
         CGPROGRAM
         //定义顶点着色器名称为vert
         #pragma vertex vert
         //定义片元着色器名称为frag
         #pragma  fragment frag
         //顶点着色器函数
         void vert(in float4 vertex : POSITION,out float4 position : SV_POSITION)
         {
            //把模型空间转换到裁切空间
            position=UnityObjectToClipPos(vertex);
         }
         //片段着色器函数
         void frag(in float4 vertex : SV_POSITION,out fixed4 color : SV_TARGET)
         {
            //将color设置为红色
            color=fixed4(1,0,0,1);
         }
         ENDCG
      }
   }
}
~~~

给材质的渲染效果为一个红色的材质。

![image-20241112153953207](https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20241112153953207.png)

#### 无返回值函数

上述案例中使用了无返回值的函数，基本结构如下：

~~~glsl
void name (in 参数, out 参数)
{
    //函数体
}
~~~

in为输入参数，语法为：in + 数据类型 + 名称，可以有多个输入，允许省略in关键词。

out为输出参数，语法为：in + 数据类型 + 名称，可以有多个输出。

#### 有返回值的函数

有返回值的函数不再使用out关键词输出参数，直接通过最后的return关键词返回一个变量，结构如下：

~~~glsl
type name (in 参数)
{
    //函数体
    return 返回值;
}
~~~

我将上面的案例改造成有返回值的函数形式：

~~~glsl
Shader "Custom/Test1"
{
   SubShader
   {
      Pass{
         //嵌入指定的CG程序，放在CGPROGRAM和ENDCG之间
         CGPROGRAM
         //定义顶点着色器名称为vert
         #pragma vertex vert
         //定义片元着色器名称为frag
         #pragma  fragment frag
         //顶点着色器函数
         float4 vert(in float4 vertex : POSITION):SV_POSITION
         {
            //把模型空间转换到裁切空间
           return UnityObjectToClipPos(vertex);
         }
         //片段着色器函数
         fixed4 frag(in float4 vertex : SV_POSITION):SV_TARGET
         {
            //将color设置为红色
            return fixed4(1,0,0,1);
         }
         ENDCG
      }
   }
}
~~~

#### 语义

上述案例中会发现输入和输出参数后会跟着一个冒号，冒号后面带有全为大写字母拼写的关键词，语义用于表示输入参数和输出参数要传递的数据信息。语义是对GPU操作的一层封装。

在顶点着色器中，顶点函数的输入参数就是顶点数据，每个输入参数都需要一个语义，用于表示所传递数据。此案例中，POSITION就是顶点坐标信息，顶点着色器使用顶点数据计算得出顶点在裁切空间中的坐标，得出的坐标就是SV_POSITION。

片段着色器可以自动获取顶点着色器输出的裁切空间顶点坐标，所以片段着色器函数的输入SV_POSITION可以省略。片段着色器通常只会输出一个fixed4类型的颜色信息，输出参数就是使用了SV_TARGET语义进行填充。

语义是附加到着色器输入或输出的字符串，用于传达有关参数预期用途的信息。 在着色器阶段之间传递的所有变量都需要语义。 此处显示了向着色器变量添加语义的语法（[变量语法 (DirectX HLSL)](https://learn.microsoft.com/zh-cn/windows/win32/direct3dhlsl/dx-graphics-hlsl-variable-syntax)）。

通常，在管道阶段之间传递的数据完全通用，并且不被系统唯一解释；允许使用没有特殊含义的任意语义。 包含这些特殊语义的参数（在 Direct3D 10 及更高版本中）称为[系统值语义](https://learn.microsoft.com/zh-cn/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics?redirectedfrom=MSDN#system-value-semantics)。

更多语义相关信息请查阅[语义 - Win32 apps | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics?redirectedfrom=MSDN)。

#### 在shader中使用颜色和纹理贴图

~~~glsl
Shader "Custom/Test1"
{
   Properties
   {
      // 定义一个名为_MainColor的颜色属性，用于设置主颜色，默认值为RGBA(1,0,1,1)
      _MainColor("MainColor",Color)=(1,0,1,1)
      // 定义一个名为_MainTex的2D纹理属性，用于设置主纹理，默认为白色纹理
      _MainTex("MainTex",2D)="white"{}
   }

   // 定义SubShader，用于描述渲染流程
   SubShader
   {
      // 定义一个Pass，用于描述一次渲染通过
      Pass{
         // 开始CG程序部分，使用CG语言编写顶点和片段着色器
         CGPROGRAM
         // 指定顶点着色器的函数名为vert
         #pragma vertex vert
         // 指定片段着色器的函数名为frag
         #pragma fragment frag

         // 定义一个全局变量_MainColor，用于在CG程序中访问主颜色
         fixed4 _MainColor;

         // 定义一个全局变量_MainTex，用于在CG程序中访问主纹理
         sampler2D _MainTex;
         //用于存储纹理的缩放和平移参数，它会根据Properties中定义的_MainTex自动生成
         float4 _MainTex_ST;

         // 顶点着色器函数，用于转换顶点位置和计算纹理坐标
         void vert (in float4 vertex:POSITION,in float2 uv:TEXCOORD0,
            out float4 position :SV_POSITION,out float2 texcoord:TEXCOORD0)
         {
            // 将顶点从模型空间转换到裁剪空间
            position=UnityObjectToClipPos(vertex);
            // 计算纹理坐标，应用纹理的缩放和平移
            texcoord=uv*_MainTex_ST.xy+_MainTex_ST.zw;
         }

         // 片段着色器函数，用于计算像素颜色
         void frag (in float4 position:SV_POSITION,in float2 texcoord:TEXCOORD0,
            out fixed4 color :SV_TARGET)
         {
            // 从纹理中采样颜色，并与主颜色相乘，得到最终的像素颜色
            color=tex2D(_MainTex,texcoord)*_MainColor;
         }
         
         // 结束CG程序部分
         ENDCG
      }
   }
}

~~~

在属性面板中选择颜色和纹理后，效果如下：

![image-20241113113341511](https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20241113113341511.png)

#### 使用立方体贴图

~~~GLSL
Shader "Custom/Cubemap Property"
{
    Properties
    {
        _MainTex ("MainTex", 2D) = "white" {}
        _MainColor ("MainColor", Color) = (1, 1, 1, 1)

        // 添加Cubemap属性和反射强度
        _Cubemap ("Cubemap", Cube) = "" {}
        _Reflection ("Reflection", Range(0, 1)) = 0
    }
    SubShader
    {
        Pass
        {
            CGPROGRAM
            #include "UnityCG.cginc"
            #pragma vertex vert
            #pragma fragment frag

            sampler _MainTex;
            float4 _MainTex_ST;
            fixed4 _MainColor;

            // 声明Cubemap和反射属性变量
            samplerCUBE _Cubemap;
            fixed _Reflection;

            void vert (in float4 vertex : POSITION,
                        in float3 normal : NORMAL,
                        in float4 uv : TEXCOORD0,
                        out float4 position : SV_POSITION,
                        out float4 worldPos : TEXCOORD0,
                        out float3 worldNormal : TEXCOORD1,
                        out float2 texcoord : TEXCOORD2)
            {
                position = UnityObjectToClipPos(vertex);

                // 将顶点坐标变换到世界空间
                worldPos = mul(unity_ObjectToWorld, vertex);

                // 将法线向量变换到世界空间
                worldNormal = mul(normal, (float3x3)unity_WorldToObject);
                worldNormal = normalize(worldNormal);

                texcoord = uv * _MainTex_ST.xy + _MainTex_ST.zy;
            }

            void frag (in float4 position : SV_POSITION,
                        in float4 worldPos : TEXCOORD0,
                        in float3 worldNormal : TEXCOORD1,
                        in float2 texcoord : TEXCOORD2,
                        out fixed4 color : SV_Target)
            {
                fixed4 main = tex2D(_MainTex, texcoord) * _MainColor;

                // 计算世界空间中从摄像机指向顶点的方向向量
                float3 viewDir = worldPos.xyz - _WorldSpaceCameraPos;
                viewDir = normalize(viewDir);

                // 套用公式计算反射向量
                float3 refDir = 2 * dot(-viewDir, worldNormal)
                                * worldNormal + viewDir;
                refDir = normalize(refDir);

                // 对Cubemap采样
                fixed4 reflection = texCUBE(_Cubemap, refDir);

                // 使用_Reflection对颜色和反射进行线性插值计算
                color = lerp(main, reflection, _Reflection);
            }
            ENDCG
        }
    }
}

~~~

渲染效果如下：

![image-20241114153151208](https://typorytp.oss-cn-hangzhou.aliyuncs.com/image-20241114153151208.png)

#### 结构体

结构体概念和C语言基本一致，就是将多个变量封装在一起，进行一个整体的输入输出。

现在对之前写过的颜色和纹理材质使用结构体进行改造。

~~~glsl
Shader "Custom/In Out Struct"
{
    Properties
    {
        _MainColor("MainColor",Color)=(1,1,1,1)
        _MainTex("MainTex",2D)="white"{}
    }
    SubShader
    {
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            //定义顶点着色器输入结构体
            struct appdata
            {
                float4 vertex:POSITION;
                float2 uv:TEXCOORD0;
            };
            //定义顶点着色器输出结构体
            struct v2f
            {
                float4 position:SV_POSITION;
                float2 texcoord:TEXCOORD0;
            };
            fixed4 _MainColor;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            //使用结构体传入传出参数
            void vert (in appdata v,out v2f o)
            {
                o.position=UnityObjectToClipPos(v.vertex);
                o.texcoord=v.uv*_MainTex_ST.xy+_MainTex_ST.zw;
            }
            void frag (in v2f i,out fixed4 color: SV_Target)
            {
                color=tex2D(_MainTex,i.texcoord)*_MainColor;
            }
            ENDCG
        }
    }
}
~~~

结构体也可以作为函数返回类型使用，具体用法如下：

~~~glsl
Shader "Custom/In Out Struct"
{
    Properties
    {
        _MainColor("MainColor",Color)=(1,1,1,1)
        _MainTex("MainTex",2D)="white"{}
    }
    SubShader
    {
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            //定义顶点着色器输入结构体
            struct appdata
            {
                float4 vertex:POSITION;
                float2 uv:TEXCOORD0;
            };
            //定义顶点着色器输出结构体
            struct v2f
            {
                float4 position:SV_POSITION;
                float2 texcoord:TEXCOORD0;
            };
            fixed4 _MainColor;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            //使用结构体传入传出参数
            v2f vert (in appdata v)
            {
                v2f o;
                o.position=UnityObjectToClipPos(v.vertex);
                o.texcoord=v.uv*_MainTex_ST.xy+_MainTex_ST.zw;
                return o;
            }
            fixed4 frag (in v2f i):SV_Target
            {
                return tex2D(_MainTex,i.texcoord)*_MainColor;
            }
            ENDCG
        }
    }
}
~~~

#### 使用unity的包含文件

在实际编写shader中常常会使用同一组数据来使用，为了简化和提高代码复用率，使用unity提供的一系列包含文件，里面有预先定义的变量，各种辅助函数和空间变换矩阵。

~~~glsl
Shader "Custom/In Out Struct"
{
    Properties
    {
        _MainColor("MainColor",Color)=(1,1,1,1)
        _MainTex("MainTex",2D)="white"{}
    }
    SubShader
    {
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            //声明包含文件
            #include "UnityCG.cginc"
            fixed4 _MainColor;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            //使用结构体传入传出参数
            v2f_img vert (in appdata_img v)
            {
                v2f_img o;
                o.pos=UnityObjectToClipPos(v.vertex);
                o.uv=TRANSFORM_TEX(v.texcoord,_MainTex);
                return o;
            }
            fixed4 frag (in v2f_img i):SV_Target
            {
                return tex2D(_MainTex,i.uv)*_MainColor;
            }
            ENDCG
        }
    }
}
~~~



