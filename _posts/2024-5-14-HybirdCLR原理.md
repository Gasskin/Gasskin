---
title: "HybirdCLR原理"
categories: Huatuo
tags: 热更新
---

先约定一些名词

- **hotfix.dll/hotfix.dll.bytes**  需要热更新的dll
- **build ab** 打ab包
- **build apk** 打安卓apk
- **HotFix层/热更新层** 执行热更新代码的地方
- **AOT层/原生层** 无法热更的部分

# IL2CPP

你的C#代码会被转成CPP代码，同时，由于你目标系统的不同，编译成的CPP代码是不一样的

# AOT/JIT/热更新

![image-20240514000124514](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img202405140035087.png)

# 泛型共享

List<A>和List<B>这是2个不同的类，这是显而易见的，如果你的项目里使用了这两个List，那么编译后大概会有这样一串代码

```c#
public class List
{
    public void Add(A o){...}
    public void Add(B o){...}
}
```

这样就很明显能看出来问题了，如果你有一万个不同的List<XXX>，那么你编译后就会有一万个方法，要知道我们项目里泛型是很随便用的，又不是单单只有List

这样的后果就是编译出来的DLL可能很爆炸，非常大！

所以IL2CPP为了解决这个问题，提出了一种泛型共享方法，大概就是提供一个通用接口，比如上面的List可能就会变成

```c#
public class List
{
    public void Add(object o){...}
}
```

当然泛型是多种多样的。。毕竟泛型还是可以是一个泛型。。所以规则也比较多，具体如下

![image-20240514003015018](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img202405140035445.png)

# AOT泛型问题

由上可知，我们的泛型共享类是IL2CPP编译好的，但是我们的热更新DLL是自己编译的啊！如果我们热更新层使用了AOT层没有用过的一个泛型，那不是寄了？

比较蠢的办法就是，把你所有需要的泛型都提前注册一下（在AOT层使用一下），但是这也太弱智了，而且万一漏了呢？

所以HCLR提供了一个叫补充元数据DLL的技术！

那么首先，啥是元数据DLL...

比如我们随便编写了一些代码，编译成了一个testDll，我们反编译他，就可以看到很多meta信息，这些就是元数据，这个DLL就是元数据DLL

![image-20240514003024146](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img202405140035130.png)

但是很可惜，元数据会在IL2CPP的时候被丢掉...

所以，我们需要把这部分元数据补充给HCLR，怎么做呢？

HCLR要求元数据DLL必须和build apk过程中生成的裁剪后的AOT DLL保持一致，在这个基础上，通过LoadMetadataForAOTAssembly函数加载即可

所以一般打APK都需要两次，第一次先打出元数据，把元数据打成ab，第二次再真的打APK...

一般加载元数据应该在加载DLL之后立刻跟着

# 代码裁剪

为了让DLL小一点，Unity会把AOT层没有用到的函数注释掉，不参与编译，如果你不需要热更新的话，其实无所谓，因为你所有代码都在AOT层

但是你有热更新DLL就不一样了，编译时HCLR会把热更新DLL排除在外，所以你热更新里用到，但AOT里没用的代码，其实是会被裁掉的。。导致热更新层跑起来的时候调用不到代码了

这个一般需要用到link.xml，官方文档：https://docs.unity3d.com/Manual/ManagedCodeStripping.html

# 桥接函数

热更新层现在由我们的HCLR解释器执行，所以IL2CPP与热更新层交互，需要一个中间媒介









































