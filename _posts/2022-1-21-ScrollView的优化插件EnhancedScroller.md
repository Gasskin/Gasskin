---
layout: post
title: "ScrollView的优化插件EnhancedScroller"
categories: [UI]
tags: [Unity,学习记录]  
---

# 基础使用

典型的MVC架构，我们需要数据对象(Model)，视图对象(View)，控制器(Controller)

建立如下结构

![image-20220121134421195](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220121134421195.png)

给我们的目标Scroll添加如下脚本

![image-20220121134736873](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220121134736873.png)

设定Scroll的Content

![image-20220121134823712](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220121134823712.png)

创建一个Item，这将作为我们Scroll内的基础对象

并调整一下布局大小

![image-20220121135022411](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220121135022411.png)

![image-20220121135027298](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220121135027298.png)

基础准备工作完成

## 数据 Model

数据脚本存储每一个Item所需要的所有数据，只是一个普通C#类

假如我们目前的Item需要一个背景色和一个名字

```c#
public class ScrollData
{
    public string name;
    public Color  color;
}
```

## 视图 View

视图里记录了所有实际显示组件的引用，比如Image，比如Text

```c#
public class ScrollView : EnhancedScrollerCellView
{
    public Image image;
    public Text  text;
}
```

必须继承EnhancedScrollerCellView

```c#
public class EnhancedScrollerCellView : MonoBehaviour
{
    ....
}
```

因为Controll里需要获取当前的Item，但是我们可能有非常多的Scroll，比如商店Scroll和背包Scroll，很明显这两个Scroll里面的Item是不一样的，但是Controller并不不关心，他只获取当前Item的EnhancedScrollerCellView，而具体组件继承自它，自然可以任意转换

## 控制器 Controller

实际进行逻辑处理以及视图刷新的地方

```c#
public class Controller : MonoBehaviour, IEnhancedScrollerDelegate
{
    public  EnhancedScroller scroller;
    public  ScrollView         cellView;
    private List<ScrollData>   datas = new();
    ...
}
```

EnHancedScroller是全局控制器，我们这个Controller只是负责刷新数据也显示，很显然Scroll并不只有这么一些东西，比如还有滑动速度啊，水平滑动还是垂直滑动啊，等等等等。

ScrollView是我们的视图节点，在这里其实是一个Prefab

最后这个List很明显就是存储了所有的数据，并不是必须的，可以按照自己的想法处理逻辑

```c#
public class Controller : MonoBehaviour, IEnhancedScrollerDelegate
{
	...
    public int GetNumberOfCells (EnhancedScroller scroller)
    {
        return datas.Count;
    }

    public float GetCellViewSize (EnhancedScroller scroller, int dataIndex)
    {
        return 300;
    }

    public EnhancedScrollerCellView GetCellView (EnhancedScroller scroller, int dataIndex, int cellIndex)
    {
        var view = scroller.GetCellView (cellView) as ScrollView;
        view.image.color = datas[dataIndex].color;
        view.text.text   = datas[dataIndex].name;

        return view;
    }
    ...
}
```

这是接口提供的三个方法

- **`GetNumberOfCells`** 

  当前有多少个Item

- **`GetCellViewSize`**

  返回每一个Item的大小（水平就是宽度，垂直就是高度），这影响两个Item之间的距离，应该和实际UI高度/宽度一致

- **`GetCellView`**

  实例化Item，因为这个插件的Item是循环利用的，如果不够，那会新创建，否则会优先从池子里拿，此时会告诉我们这是第几个Item，然后我们通过index从datas中获取数据，自行更新Item

最后我们进行调用

```c#
public class Controller : MonoBehaviour, IEnhancedScrollerDelegate
{
    private void Start ()
    {
        datas.Add (new ScrollData () { color = Color.red, name     = "AAA" });
        datas.Add (new ScrollData () { color = Color.black, name   = "BBB" });
        datas.Add (new ScrollData () { color = Color.blue, name    = "CCC" });
        datas.Add (new ScrollData () { color = Color.clear, name   = "DDD" });
        datas.Add (new ScrollData () { color = Color.cyan, name    = "EEE" });
        datas.Add (new ScrollData () { color = Color.gray, name    = "FFF" });
        datas.Add (new ScrollData () { color = Color.green, name   = "GGG" });
        datas.Add (new ScrollData () { color = Color.grey, name    = "HHH" });
        datas.Add (new ScrollData () { color = Color.magenta, name = "III" });
        datas.Add (new ScrollData () { color = Color.red, name     = "JJJ" });
        datas.Add (new ScrollData () { color = Color.white, name   = "KKK" });
        datas.Add (new ScrollData () { color = Color.yellow, name  = "LLL" });
        datas.Add (new ScrollData () { color = Color.red, name     = "MMM" });
        datas.Add (new ScrollData () { color = Color.magenta, name = "NNN" });

        scroller.Delegate = this;
        scroller.ReloadData();
    }
}
```

前面都是往datas里添加数据，最后两行才是刷新

## 具体使用

首先我们绑定Item的视图组件

![image-20220121140725210](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220121140725210.png)

拖为Prefab

![image-20220121140806490](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220121140806490.png)

创建一个控制器

![image-20220121141000055](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220121141000055.png)

CellView里拖入我们的Item预制体，Scroller则是全局控制器

运行游戏

![image-20220121141126456](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220121141126456.png)

虽然脚本有点多，但实际写起来并不麻烦

# 接入ILRuntime

如果不热更，那么上面的已经足够使用了，但UI肯定是要热更滴

需要将MVC三层架构都放到热更工程中去

我们看看这三个类的类型

- **Model** 

  普通C#类型，可以无痛放入热更

- **View** 

   继承自EnhancedScrollerCellView，间接继承了MonoBehaviour，继承了两次而且间接继承了Mono，放入热更会比较麻烦

- **Controller**

  继承了Mono，同时实现了IEnhancedScrollerDelegate接口，也继承了Mono，比较麻烦，接口只是一个普通接口，好处理

先处理比较简单的Controller，写个转换器

```c#
public class IEnhancedScrollerDelegateAdaptor : CrossBindingAdaptor
{
    public override Type   BaseCLRType                                                      { get; }
    public override Type   AdaptorType                                                      { get; }
    public override object CreateCLRInstance (AppDomain appdomain, ILTypeInstance instance)
    {
        return new Adaptor (appdomain, instance);
    }
    
    internal class Adaptor : IEnhancedScrollerDelegate, CrossBindingAdaptorType
    {
        private AppDomain      appdomain;
        private ILTypeInstance instance;

        private static CrossBindingFunctionInfo<EnhancedScroller, int>                                
            getNumberOfCells = new ("GetNumberOfCells");
        private static CrossBindingFunctionInfo<EnhancedScroller, int, float>                         
            getCellViewSize  = new("GetCellViewSize");
        private static CrossBindingFunctionInfo<EnhancedScroller, int, int, EnhancedScrollerCellView> 
            getCellView      = new("GetCellView");

        public ILTypeInstance ILInstance => instance;

        public Adaptor () { }

        public Adaptor (AppDomain appdomain, ILTypeInstance instance)
        {
            this.appdomain = appdomain;
            this.instance  = instance;
        }

        public int                      GetNumberOfCells (EnhancedScroller scroller)
        {
            return getNumberOfCells.Invoke (instance, scroller);
        }

        public float                    GetCellViewSize (EnhancedScroller scroller, int dataIndex)
        {
            return getCellViewSize.Invoke (instance, scroller, dataIndex);
        }

        public EnhancedScrollerCellView GetCellView (EnhancedScroller scroller, int dataIndex, int cellIndex)
        {
            return getCellView.Invoke (instance, scroller, dataIndex, cellIndex);
        }
    }
}
```

