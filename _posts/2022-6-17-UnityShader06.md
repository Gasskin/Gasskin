---
title: "入门精要|06 更复杂的光照"
tags: 图形学
---
在此前的章节中，我们存在两个限制：
1. 场景中仅存在一个平行光
2. 不存在阴影 

在这一章我们会对这两个效果进行处理

# Unity渲染路径
目前主要有两种渲染路径
1. 向前渲染
2. 延迟渲染

我们可以通过Project面板设置，这里设置的是项目统一渲染路径
![](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202206172153261.png)

当然我们也可以为每一个相机单独设置渲染路径
![](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202206172154940.png)

设置完成之后，可以在shader中通过**LightMode**标签对Pass的渲染路径进行设置

```hlsl
Pass
{
	Tags { "LightMode" = "ForwardBase" }
}
```

**LightMode**支持的渲染路径有：
- **Always** 该Pass总会渲染，但不计算光照
- **ForwardBase** 用于**向前渲染**，该Pass会计算环境光、最重要的平行光、逐顶点/SH光源，以及Lightmaps
- **ForwardAdd** 用于**向前渲染**，会额外计算逐像素光源，每个PASS一个光源
- **Deferred** 用于**延迟渲染**，该PASS会渲染G-Buffer
- **ShadowCaster** 把物体的深度信息渲染到阴影映射纹理中

## 向前渲染
向前渲染主要有3种处理光照的方式，逐顶点、逐像素、球谐函数(SH)，而使用哪种模式来处理光源，则取决于**光源的类型**和**渲染方式**
光源的类型是指该光源是**平行光**还是**其他类型**
渲染方式则是指**该光源是否重要**

我们可以在光源中设置该光源的渲染方式
![](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202206172213384.png)

当场景中存在多个光源时，Unity会使用如下规则进行排布
- 场景中最亮的平行光总是逐像素的
- 被设置为Not Important的光源会按照逐顶点或者SH的方式处理
- 被设置为Important的光源会按照逐像素的方式处理
- 如果按照以上规则得到的逐像素光源数少于Quality Setting中的限制，那么会有更多光源被进行逐像素处理

光照会在PASS中进行计算，有两种，**Base Pass**以及**Additional Pass**

- **BasePass** 一个逐像素的平行光，以及所有顶点光和SH光

  ```c#
  Tags{"LightMode"="ForwardBase"}
  #pragma multi_compile_fwdbase
  ```

- **AdditionalPass** 其他所有逐像素光源，每个光源都会执行一次PASS

  ```c#
  Tags{"LightMode"="ForwardAdd"}
  #pragma multi_compile_fwdadd
  ```

## 延迟渲染

向前渲染的问题是，每一个光源影响的物体，都会进行一个PASS，如果有M个光源，N个物体，那么PASS数量会急剧上升，说白了就是性能问题很大

因此延迟渲染重新提上了日常，正常的向前渲染中，我们只会有颜色缓冲和深度缓冲，但在延迟渲染中，需要一个G-Buffer缓冲，此缓冲区会存储某一个像素区域的某些信息，比如法线，位置，等

### 1.原理

在延迟渲染中，我们需要两个PASS，在第一个PASS里，我们不进行光照计算，仅仅计算哪一些片元是可见的（这通过深度缓冲就可以简单实现），而如果一个片元是可见的，那我们就把他的相关信息存储到G-Buffer中（法线，系数等）

然后再第二个PASS中，利用G-Buffer中的信息，进行光照计算，只需要计算一次

# 阴影





















