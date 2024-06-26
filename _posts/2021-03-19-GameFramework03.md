---
title: "GameFramework|03 UI管理器01"
categories: GameFramework 
tags: 框架
---

### 理论

首先，为什么需要一个UI框架？抛开所有的想法， 假设我们用最最最基础的做法来做一个UI，我们会怎么操作？很简单，作为入门的一个小白，我可能会在场景里拖一个canvas，然后设置他的active为false，并在需要的时候激活canvas，那样就能显示和隐藏

这没有问题，毕竟我的第一个UI界面就是这么写的。。。

咳咳，但是！但是我们的游戏不可能只有一个界面啊，比如商店、背包、人物属性等等，UI的数量绝对是难以预计的多。。难不成给每一个UI都拖一个canvas？当然理论上这是可行的。。就像你可以把整个游戏的逻辑都写在一个Update()里，理论上，应该也是可行的。。

好了，说认真点，框架还是为了提高维护性和拓展性，当UI越来越多的时候，如果我们不进行统一的管理，那会造成很多不必要的麻烦，比如会有这样几个非常明显的问题：

1. 很多UI可以在多个场景出现，比如背包，我们不可能在每个场景都建一个canvas然后把背包UI摆放一遍，所以UI一定要是可复用的
2. UI之间经常会相互的传递数值，典型的就是鼠标放到背包某个物体上，显示对应的属性
3. 当同时打开多个UI时，怎么维护他们之间的先后关系，一般来说，游戏是不允许跨UI操作的，即必须先处理最顶层的UI
4. ...

当然可能还有其他很多问题，不多赘述了，如果是萌新可能确实没太多的感觉（比如当初我接触框架就完全一脸蒙），但认证看完整个框架的设计后，想必诸位都能有自己的想法！

### 窗口类型

我们首先当然需要确定，一个游戏会有哪几种类型的窗口，不同的窗口对应不同的显示方式！当然，目前我觉得就三种，背景类型、一般类型、弹窗类型，让我们在框架下新建一个模块文件夹，就叫UI吧，并新建一个脚本，作为UI的配置信息！

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/1.png)

```c#
public enum UIType
{
    /// <summary>
    /// 固定类型，一般作为背景
    /// </summary>
    Fixed,
    /// <summary>
    /// 普通窗口，一般窗口都用这个
    /// </summary>
    Normal,
    /// <summary>
    /// 弹窗，可以在某些窗口上弹出的窗口
    /// </summary>
    PopUp
}
```

在这里明确几点，一般来说，打开一个新的UI是会关闭上一个UI的，当然具体看你需求，但是Fixed和Normal一般来说各自只允许打开一个，比如你打开新的Fixed就会关闭上一个Fixed，你打开新的Normal就会关闭上一个Normal，但你打开Normal是不会关闭Fixed的，PopUp的逻辑和另外两个不一样，弹窗那是随便弹的，并不会关闭其他UI

其实这三个类型区别还是挺大的，多熟悉一下就明白了！

### 窗口基类

所谓窗口基类，那自然是所有窗口脚本都必须继承的类，以便我们之后用管理器统一管理，那么让我们思考一下一个窗口需要哪些基本的生命周期？首先，打开和关闭，这是必须的，毋庸置疑。其次需要考虑的是初始化和卸载，类似于Awake()和OnDestroy()，用于加载和卸载资源。

这就好了吗？其实还缺一部分！

思考一下，如果我们现在打开了一个商城UI购买东西，然后发现金币不足，弹出了一个弹窗提示，按照正常逻辑，在我们点掉弹窗之前，我们是不能够操作商城UI的，这是很符合逻辑的，我们必须先处理最顶层的弹窗，也就是说，我们的商城UI会被冻结，等弹窗被点掉了，再解冻。当然还有其他很多时候我们需要冻结我们的UI，因此我们还需要另外两个生命周期，Freeze()、UnFreeze()，即冻结和解冻

EMM，目前就想到这些，那就先这么写一个基类出来先！让我们新建一个脚本叫做UIBase

```c#
public abstract class UIBase : MonoBehaviour
{
    #region Public

    /// 窗口类型，默认是Normal
    public UIType type = UIType.Normal;

    #endregion
        
    #region MonoBehaviour

    private void Awake()
    {
        Load();
    }

    private void OnDestroy()
    {
        UnLoad();
    }

    #endregion
    
    #region 生命周期

    /// <summary>
    /// 加载UI，类似于Awake()，多用于初始化
    /// </summary>
    public abstract void Load();

    /// <summary>
    /// 卸载UI，类似于OnDestroy()，多用于卸载
    /// </summary>
    public abstract void UnLoad();

    /// <summary>
    /// 会在UI管理器中被统一调用，用于更新
    /// </summary>
    public virtual void OnUpdate()
    {
        
    }

    /// <summary>
    /// 打开UI
    /// </summary>
    public virtual void Show()
    {
        
    }

    /// <summary>
    /// 关闭UI
    /// </summary>
    public virtual void Close()
    {
        
    }

    /// <summary>
    /// 冻结UI
    /// </summary>
    public virtual void Freeze()
    {
        
    }

    /// <summary>
    /// 解冻UI
    /// </summary>
    public virtual void UnFreeze()
    {
        
    }

    #endregion
}

```

再解释一下，首先，很明显，Load和UnLoad分别会在Awake和OnDestroy中调用，很明显就是为了加载或者卸载资源，这两个方法是abstract，也就是说子类必须重写

另外四个是Show、Close、Freeze、UnFreeze，这几个就是普通的生命周期方法了，所以是Virtual的，因为子类不一定需要

还需要强调的是OnUpdate，虽然我们的脚本继承自MonoBehaviour，但我们不想要他自己更新，因为我们会有很多UI，如果每一个UI都自动Update，那会有很多不必要的逻辑，这个OnUpdate会被UI管理器所调用，用于更新UI

### 如何加载？

在写UI管理器之前，我们需要先考虑几个问题，最终我们的UI应该如何被加载？被加载到哪？加载方式目前来看，没什么更多的选择，无非是把所有UI做成Prefab，然后需要的时候直接加载Prefab就好，但问题是，应该把UI放到哪儿呢？考虑这样一个问题，如果我们有很多的UI Prefab，是不是每一个预制体都需要带根节点，也就是Canvas？很明显是不需要的， 因为所有UI都可以放到同一个Cancas下。但如果只是如此，应该如何维护UI之间的层级关系呢？所以我们会在Cancas下再分几个层级。

可能听起来比较迷糊，看一下图就很明白了

首先新建一个Canvas，把EventSystem放到它下面，并制作成预制体，改名为UIRoot，也就是作为我们所有UI的根节点

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/2.png)

然后，我们再给它多增加三个节点，分别作为对应UI的挂在节点

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/3.png)

现在，加载方式就很明确了，首先，我们会加载一个UIRoot作为根节点，其次，再加载想要的UI，根据UI的类型，我们把他们放入不同的子节点中

注意两点，首先，此时的UI预制体不需要带Canvas，只需要是UI本身就好，其次，目前并没有什么资源加载框架，所以我们是直接放在Resources下的，一般来说不会把所有资源都放到Resources下，但目前先这么用~~

### UI管理器？

好了！总算到这一步了！但还是不行。。

再管理UI之前。。我们先把目前的加载方式整理一下。。给定好预制体加载后的存放位置，如果一股脑全部放在Hierarchy中，那会很乱的。。

好了，让我们在UI模块下新建一个脚本叫UIManager，记得继承自ManagrBase

```c#
public class UIManager : ManagerBase
{
    public override int Priority { get; }

    #region 管理器生命周期

    public override void Init()
    {
        
    }

    public override void Update(float time)
    {
        
    }

    public override void ShutDown()
    {
        
    }

    #endregion

    #region 接口方法

    public void OpenUI()
    {
        GameObject go = Resources.Load<GameObject>("UI/UIRoot");
        if (go)
        {
            GameObject.Instantiate(go);
        }
    }

    #endregion
}
```

然后写一个测试脚本，并随便拖到场景中运行

```c#
public class test : MonoBehaviour
{
    private UIManager uiManager;
    void Start()
    {
        uiManager = SGFEntry.Instance.GetManager<UIManager>();
        uiManager.OpenUI();
    }
}
```

结果如下

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/4.png)

我们的UIRoot确实被加载出来了，但这样的结果真是我们想要的吗？

其实还是能改进一点的。。比如。。我们就想要UIRoot放到咱们的SGFEntry下面，这是很符合正常逻辑的，当然应该把框架相关的的东西放到一起！

不仅如此，更进一步想，SGFEntry之下有很多管理器，如果所有管理器的东西我们都丢到SGFEntry之下，那随着管理器变多，依旧会变成非常凌乱，因此我们再生成管理器的时候，还应该在SGFEntry下生成一个对应的Empty GameObject，用来存放管理器相关的东西

### 修改SGFEntry

现在我们的目标是，我们应该在SGFEntry下生成很多的空节点，对应于不同的管理器，并且，理所定然的，他们顺序应该和管理器的优先级一样，有很多方法可以实现这个目标啦，在这里我只分享一下我目前的思路

首先我们在Core里新建一个脚本，用来储存管理器的优先级，当然你也可以在每个管理器里自定义，但想一下，如果某一天你需要修改管理器的优先级，那是不是会很麻烦呢？所以我们在这里统一记录。

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/5.png)

```c#
/// 用于记录管理器的优先级，每一个管理器的Proprity属性需要返回对应enum的HashCode
public enum ManagerPriority
{
    UIManager,
    EventManager,
    ReferenceManager,
}
```

那么很自然的，每一个管理器的Priority都要做对应的修改，比如UI管理器就会写成这样

```c#
public override int Priority
{
    get { return ManagerPriority.UIManager.GetHashCode(); }
}
```

如果以后我们需要修改管理器的优先级，那么只要修改这个enum就好啦！

好了，第二步我们就应该让SGFEntry根据这个enum，来生成对应的根节点

```c#
public class SGFEntry : Singleton<SGFEntry>
{
    /// 新增Awake()，初始化根节点
    private void Awake()
    {
        var names = Enum.GetNames(typeof(ManagerPriority));
        for (int i = 0, imax = names.Length; i < imax; i++)
        {
            GameObject go = new GameObject(names[i]);
            go.transform.SetParent(transform);
            go.SetActive(false);// 注意这里我们设置了active为false，应该在获取管理器的时候才真正的打开
        }
    }
    
    /// 从管理器链表中获取指定的管理器，如果没有，那会创建一个对应的管理器，并加入管理器链表 
    public TManager GetManager<TManager>() where TManager : ManagerBase
    {
        Type managerType = typeof(TManager);

        // 打开SGFEntry下对应的节点
        transform.Find(managerType.Name).gameObject.SetActive(true);            

        ///...
    }
}
```

因为一般来说，管理器打开了是不会主动关闭的，除非游戏都停了，所以暂时我们不写关闭的逻辑，好啦运行一下test看看现在的情况

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/6.png)

what！为什么UIRoot还在外面！！

废话。。因为我们压根还没改加载UI的逻辑。。

但是咱们SGFEntry就初见成效了！看起来还不错，强迫症表示满足（嗯，暂时满足）

### UI管理器！

好了别急。。这次我们是真的开始写UI管理器了。。

首先是初始化逻辑，保存我们的几个节点的信息

```c#
public class UIManager : ManagerBase
{
    /// 根节点，即UIRoot 
    private Transform uiRoot;
    /// fixed界面的节点 
    private Transform fixedRoot;
    /// normal界面的节点
    private Transform normalRoot;
    /// popUp界面的节点
    private Transform popUpRoot;

    public override void Init()
    {
        // 保存各个节点的信息
        if (null == uiRoot)
        {
            // 加载UIRoot Prefab
            GameObject go = Resources.Load<GameObject>("UI/UIRoot");
            var prefab = GameObject.Instantiate(go);

            // 初始化节点信息
            var uiNode = SGFEntry.Instance.transform.Find("UIManager");
            prefab.transform.SetParent(uiNode);
            uiRoot = prefab.transform;
            fixedRoot = uiRoot.Find("Fixed");
            normalRoot = uiRoot.Find("Normal");
            popUpRoot = uiRoot.Find("PopUp");
            if (fixedRoot == null || normalRoot == null || popUpRoot == null)
            {
                throw new Exception("==== UI节点初始化失败 ===");
            }
        }
    }
}
```

然后我们只做一个简单的UI Prefab来测试一下先

随便做做

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/8.png)

**强调一下，制作UI的时候我们不需要再次设置一个Canvas！因为已经有统一的UIRoot了！如图，我们只需要在对应的节点下面制作就行，比如现在，我们就可以把TestUI做成一个预制体！！然后把他从UIRoot中删除！！**

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/9.png)

简单写一个测试方法测试一下

```c#
public class UIManager : ManagerBase
{
    public void OpenUI()
    {
        GameObject go = Resources.Load<GameObject>("UI/TestUI");
        var prefab = GameObject.Instantiate(go);
        /// 第二个参数一定要是false
        prefab.transform.SetParent(normalRoot,false);
    }
}
```

运行游戏！

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/10.png)

完美符合预期！





