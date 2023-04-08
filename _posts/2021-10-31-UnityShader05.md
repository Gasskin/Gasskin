---
title: "05 透明效果"
tags: UnityShader
---

- **深度测试**

  根据深度缓冲，比较两个片元距离摄像机的距离，如果当前片元比起深度缓冲中的值来说，离摄像机更远，那么当前片元不会被写入。

- **透明度测试**

  如果片元的透明度不满足条件，则被舍去，不进行任何处理。如果满足条件，那就当做不透明物体进行处理。不需要关闭深度写入。

- **透明度混合**

  根据当前片元的透明度，与颜色缓冲中的颜色进行混合，得到新的颜色。需要关闭深度写入。为什么？因为透明物体不会遮挡后面的物体，如果开启了深度缓冲，那么根据深度测试，在该透明物体后方的物体，不会被写入深度缓冲那种。这不是我们所需要的。

# 渲染顺序

<img src="https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202110311736852.png" alt="image-20211031173619808" style="zoom:50%;" />

考虑上图，假设A是透明物体，B是不透明物体，此时我们开启了透明度混合，**关闭了深度写入功能**

如果我们先渲染B，当我们渲染A时，会发现深度缓冲中已经有更远的值了，所以A会根据透明度，混合到颜色缓冲中

但如果我们先渲染A，由于A关闭了深度写入功能，他不会忘缓冲中写入任何数据，当我们渲染B时，B的颜色会直接覆盖A

### Unity内置渲染队列

|    名称     | 队列索引号 |                             描述                             |
| :---------: | :--------: | :----------------------------------------------------------: |
| Background  |    1000    |                 最先被渲染，通常用于渲染背景                 |
|  Geometry   |    2000    |            默认渲染队列，不透明物体需使用这个队列            |
|  AlphaTest  |    2450    |                        透明度测试队列                        |
| Transparent |    3000    | 会在Geometry和AlphaTest之后，按照从后往前的顺序进行渲染，任何使用了透明度混合的队物体应该使用此队列 |
|   Overlay   |    4000    |               最后被渲染，通常用于一些叠加效果               |

### 透明度测试

``` c#
Shader "Custom/AlphaTest"
{
    Properties
    {
        _MainTex ("MainTex", 2D) = "white" { }
        _CutOff ("CutOff", Range(0, 1)) = 0.5
    }
    SubShader
    {
        // 渲染队列：透明度测试
        // RenderType将Shader归入提前定义的组中，即TransparentCutout
        // IgnoreProjector=true，忽略投影
        Tags { "Queue" = "AlphaTest" "IgnoreProjector" = "True" "RenderType" = "TransparentCutout" }
        
        Pass
        {
            Tags { "LightMode" = "ForwardBase" }
            
            CGPROGRAM
            
            #pragma vertex vert
            #pragma fragment frag
            
            #include "Lighting.cginc"
            
            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed _CutOff;
            
            struct a2v
            {
                float4 vertex: POSITION;
                float3 normal: NORMAL;
                float4 texcoord: TEXCOORD0;
            };
            
            struct v2f
            {
                float4 pos: SV_POSITION;
                float3 worldNormal: TEXCOORD0;
                float3 worldPos: TEXCOORD1;
                float2 uv: TEXCOORD2;
            };
            
            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex);
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
                
                return o;
            }
            
            fixed4 frag(v2f i): SV_TARGET
            {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLight = normalize(UnityWorldSpaceLightDir(i.worldPos));
                
                fixed4 texColor = tex2D(_MainTex, i.uv);
                
                // clip(texColor - _CutOff);
                if (texColor.a < _CutOff)
                    discard;

                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.rgb * texColor.rgb;
                fixed3 diffuse = _LightColor0.rgb * texColor.rgb * max(0, dot(worldNormal, worldLight));
                
                return fixed4(ambient + diffuse, 1);
            }
            
            
            ENDCG
            
        }
    }
}
```

<img src="https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202110311825022.png" alt="image-20211031182508965" style="zoom:50%;" />

### 透明度混合

混合语义，将当前颜色与颜色缓冲中的颜色进行混合

|          语义          |                             描述                             |
| :--------------------: | :----------------------------------------------------------: |
|       Belend Off       |                           关闭混合                           |
|       Blend A B        | 开启混合并设置混合因子，源颜色会乘以A，已存在颜色会乘以B，并相加混合 |
|     Blend A B C D      |                  同上，但采用更多的混合因子                  |
| BlendOp BlendOperation | 开启混合，但不只是对颜色的简单相加，而是进行BlendOperation操作 |

```c#
Shader "Custom/AlphaBend"
{
    Properties
    {
        _MainTex ("MainTex", 2D) = "white" { }
        _AlphaScale ("Alpha Scale", Range(0, 1)) = 0.5
    }
    SubShader
    {
        // 忽略投影，队列和渲染类型都是透明类型
        Tags { "Queue" = "Transparent" "IgnoreProjector" = "True" "RenderType" = "Transparent" }
        
        pass
        {
            Tags { "LightMode" = "ForwardBase" }
            
            // 关闭深度写入，设置混合
            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha
            CGPROGRAM
            
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed _AlphaScale;
            struct a2v
            {
                float4 vertex: POSITION;
                float3 normal: NORMAL;
                float4 texcoord: TEXCOORD0;
            };
            
            struct v2f
            {
                float4 pos: SV_POSITION;
                float3 worldNormal: TEXCOORD0;
                float3 worldPos: TEXCOORD1;
                float2 uv: TEXCOORD2;
            };
            
            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex);
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
                
                return o;
            }
            
            fixed4 frag(v2f i): SV_TARGET
            {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLight = normalize(UnityWorldSpaceLightDir(i.worldPos));
                
                fixed4 texColor = tex2D(_MainTex, i.uv);
                
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.rgb * texColor.rgb;
                fixed3 diffuse = _LightColor0.rgb * texColor.rgb * max(0, dot(worldNormal, worldLight));
                
                return fixed4(ambient + diffuse, texColor.a * _AlphaScale);
            }
            
            ENDCG
            
        }
    }
    FallBack "Transparent/VertexLit"
}
```

<img src="https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202110312121671.png" alt="image-20211031212137595" style="zoom:50%;" />

注意是没有阴影的，因为最后的FallBack是VertexLit

但当上述材质应用于一个复杂模型时，会出现明显的问题

<img src="https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202110312223281.png" alt="image-20211031222327184" style="zoom:50%;" />

因为我们关闭了深度写入，片元之间不会有任何排序和剔除，所有的片元颜色都被渲染了

解决方法是采用两个Pass，第一个Pass仅仅用来写入深度信息

```c#
Shader "Custom/AlphaBend"
{
    Properties
    {
        _MainTex ("MainTex", 2D) = "white" { }
        _AlphaScale ("Alpha Scale", Range(0, 1)) = 0.5
    }
    SubShader
    {
        // 忽略投影，队列和渲染类型都是透明类型
        Tags { "Queue" = "Transparent" "IgnoreProjector" = "True" "RenderType" = "Transparent" }
        pass
        {
            ZWrite On
            ColorMask 0
        }
        pass
        {
			// 内容同上            
        }
    }
    FallBack "Transparent/VertexLit"
}
```

- **ColorMaks** 用于设置颜色通道的写入掩码，即该PASS会写入哪一个颜色通道，0代表不输出任何颜色

<img src="https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202110312225919.png" alt="image-20211031222542817" style="zoom:50%;" />

# 混合命令

称当前片元着色器产生的颜色为源颜色（Source Color），称颜色缓冲中的颜色为目标颜色（Destination Color）

当开启混合指令后，会混合以上两种颜色，包含RGBA四个通道

### 混合指令

混合是一个逐片元的操作，并且不可编程，但在Unity中的可配置性很高

RGB通道和A通道需要不同的混合因子，所以一般而言我们需要4个因子来混合源颜色与目标颜色

|          语义          |                             描述                             |
| :--------------------: | :----------------------------------------------------------: |
|       Belend Off       |                           关闭混合                           |
|       Blend A B        | 开启混合并设置混合因子，源颜色会乘以A，已存在颜色会乘以B，并相加混合 |
|     Blend A B C D      |                  同上，但采用更多的混合因子                  |
| BlendOp BlendOperation | 开启混合，但不只是对颜色的简单相加，而是进行BlendOperation操作 |

当使用语义2时，只提供了两个参数，这意味会使用相同的混合因子来混合RGB通道和A通道

$O_{rgb}=A*S_{rgb}+B*D_{rgb}$

$O_{a}=A*S_{a}+B*D_{a}$

目前Unity支持以下混合因子

|       参数       |                             描述                             |
| :--------------: | :----------------------------------------------------------: |
|       One        |                              1                               |
|       Zero       |                              0                               |
|     ScrColor     | 源颜色值，混合RGB通道时使用$S_{rgb}$，混合A通道时使用$S_{a}$ |
|     ScrAlpha     |                        源颜色的透明度                        |
|     DstColor     |                     目标颜色值，其余同上                     |
|     DstAlpha     |                             同上                             |
| OneMinusScrColor |                          1-源颜色值                          |
| OneMinusScrAlpha |                        1-源颜色透明度                        |
| OneMinusDstColor |                             同上                             |
| OneMinusDstAlpha |                             同上                             |

默认混合操作是对源颜色和目标颜色进行相加操作，当然也可以使用指令**BlendOp BlendOperation**来设置对应操作

|  操作  |            描述             |
| :----: | :-------------------------: |
|  Add   | 混合后的当前颜色 + 缓存颜色 |
|  Sub   | 混合后的当前颜色 - 缓存颜色 |
| RevSub | 缓存颜色 - 混合后的当前颜色 |
|  Min   |  min（当前颜色，缓存颜色）  |
|  Max   |  max (当前颜色，缓存颜色)   |

# 双面渲染

在现实中，如果某一个物体的透明的，我们通常能够看到它的内部结构，但前面的正方体却没有这样的效果，因为默认情况下，Unity剔除了模型的背面

可以使用**Cull**指令来控制需要剔除的图元

- **Back** 背面剔除
- **Front** 正面剔除
- **Off** 关闭

### 透明度测试下的双面渲染

```c#
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader "Custom/AlphaTest"
{
    Properties
    {
        _MainTex ("MainTex", 2D) = "white" { }
        _CutOff ("CutOff", Range(0, 1)) = 0.5
    }
    SubShader
    {
        // 渲染队列：透明度测试
        // RenderType将Shader归入提前定义的组中，即TransparentCutout
        // IgnoreProjector=true，忽略投影
        Tags { "Queue" = "AlphaTest" "IgnoreProjector" = "True" "RenderType" = "TransparentCutout" }
        
        Pass
        {
            Tags { "LightMode" = "ForwardBase" }
            
            // 关闭剔除
            Cull Off
            
            ...
        }
    }
}
```

<img src="https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202111012309537.png" alt="image-20211101230935468" style="zoom:50%;" />

### 透明度测试下的双面渲染

 透明度混合需要关闭深度写入，此时如果还采用透明度测试中的方法，那么无法保证正面和背面的渲染顺序

可以利用两个PASS，第一个PASS只渲染背面，第二个PASS只渲染正面

```c#
Shader "Custom/AlphaBlendBothSide"
{
    Properties
    {
        _MainTex ("MainTex", 2D) = "white" { }
        _AlphaScale ("Alpha Scale", Range(0, 1)) = 0.5
    }
    SubShader
    {
        // 忽略投影，队列和渲染类型都是透明类型
        Tags { "Queue" = "Transparent" "IgnoreProjector" = "True" "RenderType" = "Transparent" }
        pass
        {
            Tags { "LightMode" = "ForwardBase" }
            ZWrite Off
            Cull Front
            Blend  SrcAlpha OneMinusSrcAlpha
            
            ...
        }
        pass
        {
            Tags { "LightMode" = "ForwardBase" }
            
            Cull Back
            ZWrite Off
            Blend  SrcAlpha OneMinusSrcAlpha
            CGPROGRAM
            
            ...
        }
    }
    FallBack "Transparent/VertexLit"
}

```

<img src="https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202111012316070.png" alt="image-20211101231632006" style="zoom:50%;" />

















