---
title: "入门精要|03 基础光照"
categories: UnityShader入门精要
tags: 图形学
---

### 环境光 

可以认为就是给物体加一层底色，因为即使在黑暗的情况下，世界上通常也仍然有一些光亮（月亮、远处的光），所以物体几乎永远不会是完全黑暗的。为了模拟这个，我们会使用一个环境光照常量，它永远会给物体一些颜色。

在Windows/Lighting/Settings中的AmbientOcclusion中设置，在Shader中通过UNITY_LIGHTMODEL_AMBIENT得到环境光的颜色和强度

### 自发光

物体自身的颜色，一般来说需要乘以一个很小的系数，也算是一种底色

$c_{self}=k_{self} * c_{base}$

$k_{self}$：自发光系数，一般来说很小，当然得看具体物体

$c_{base}$：物体自身颜色

### 漫反射

物体的漫反射取决于物体表演面色、光照颜色以及光照方向，主要有两个特点：

**(1)反射强度与观察者的角度没有关系；(2)反射强度与光线的入射角度有关系**

很好理解，我从正面打一束光，那物体背面无光照的地方肯定是黑色的，如何描述光照方向与颜色之间的关系呢？

由Lambert 光照模型定义，某一个点的漫反射强度取决于光线方向与该点的法线方向	

<img src="https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202110161629928.png" alt="image-20211016162938885"  />

如图，该点的漫反射强度等于光线方向点成法线方向，注意必须是单位向量

$m_{diffuse}=\hat{L}\cdot\hat{N}=cos\theta$

其范围是[-1,1]，当然我们不可能希望某一个点的漫反射强度会是负数，所以再加上限制

$m_{diffuse}=\max(0,\hat{L}\cdot\hat{N})$

上述只是漫反射强度，想要计算漫反射颜色还需要乘以当前光照颜色以及物体自身的漫反射系数，即

$c_{diffuse}=c_{light} * c_{diffuse} * m_{diffuse}$

$c_{light}$：光照颜色

$c_{diffuse}$：漫反射颜色

$m_{diffuse}$：漫反射强度

### 高光反射（Blinn Phone 模型）

<img src="https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202110161752650.png" alt="image-20211016175234604" style="zoom:50%;" />

高光反射模拟的是一种镜面反射，即如果如果光线方向和观察方向关于当前这个点的法线方向对称，那么是完全的镜面反射，即高光最强烈

我们假设向量H在向量V和向量L的中间

那么我们可以很容易的得到这么一个结论：H越靠近N，则该点的高光越强烈

**问：如何描述H和N之间的距离关系？**

答：角度！Cos！点乘！

**问：如何求H？**

如果V和L不是单位向量，那么根据向量的平行四边形法则，V+L等于对应平行四边形的中线，而此时V和L都是单位向量，那么这就成了一个正方形，正方形的中线可不就是角平分线嘛？

但注意结果不是单位向量，还需要进行一次归一化，即

$\hat{H}=\frac{\vec{V}+\vec{L}}{\lVert{\vec{V}+\vec{L}\rVert}}$

那么高光强度也很容易计算了，同样我们不希望高光强度会小于0，并且为了描述“H和N”足够接近，我们加上一个指数

$m_{specular}=\max(0,\hat{N}\cdot\hat{H})^{m_{gloss}}$

总结，高光颜色公式如下

$c_{specular}=c_{light} * c_{specular} * m_{specular}$

$c_{light}$：光照颜色

$c_{specular}$：高光颜色

$m_{specular}$：高光强度

### 一个基础光照

```hlsl
Shader "Custom/BaseLit"
{
    Properties
    {
        _BaseColor("自发光颜色",Color) = (1,1,1,1)
        _Base("自发光系数",Range(0,1))=0.01
        _Diffuse("漫反射系数",Range(0,1))=1
        _Specular("高光系数1",Range(0,1))=1
        _Gloss("高光系数2",float)=1
    }
    SubShader
    {
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag
            
            #include "Lighting.cginc"

            
            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                fixed3 worldNormal : TEXTURE0;
                fixed3 worldPos : TEXTURE1;
            };

            fixed4 _BaseColor;
            float _Base;
            float _Diffuse;
            float _Specular;
            float _Gloss;

            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = normalize(UnityObjectToWorldNormal(v.normal));
                o.worldPos = mul(unity_ObjectToWorld,v.vertex);
                return o;
            }

            fixed4 frag(v2f i):SV_Target
            {
                // 环境光
                fixed4 environment = UNITY_LIGHTMODEL_AMBIENT;
                // 自发光
                fixed4 base = _Base*_BaseColor;
                // 漫反射
                fixed3 worldLight = normalize(_WorldSpaceLightPos0);
                fixed3 diffuse = _LightColor0 * _Diffuse * saturate(dot(i.worldNormal,worldLight));
                // 高光
                fixed3  viewDir = normalize(_WorldSpaceCameraPos-i.worldPos);
                fixed3 halfDir = normalize(_WorldSpaceLightPos0+viewDir);
                fixed3 specular = _LightColor0*_Specular*pow(max(0,dot(i.worldNormal,halfDir)),_Gloss);                
                
                return fixed4(environment.xyz+base.xyz+diffuse+specular,1.0);
            }
            
            ENDCG
        }
    }
    FallBack "Diffuse"
}

```

































