---
title: "GameFramework|14 UI源码分析"
categories: GameFramework 
tags: 框架
---

# 主要流程解析

**UIManager.OpenUIForm** 是打开UI的入口

![image-20221014164934152](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210141649226.png)

界面资源很明显就是对应UI的Prefab，界面组应该是一种逻辑上的结构，后面三个参数比较好理解

**m_Serial **应该是界面编号，随着界面的打开会不断的增加

**m_InstancePool **很明显是一个对象池，首先尝试从对象池中获取UI资源（GF中，ObjectPool是具体资源对象池，ReferencePool是逻辑类对象池）

假设我们从来没有加载过这个UI资源，那么自然对象池里也没有对象，那么会进入 **If** 逻辑

**m_UIFormsBeingLoaded** 是一个字典，仅仅用于记录当前那些资源界面正在被加载中，加载结束后会被从字典中删除

随后调用资源管理器加载资源，资源管理器部分的逻辑在这里暂且忽略

加载相关的回调函数在 **UIManager** 初始化时已经设置过了

![image-20221014165937327](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210141659358.png)

我们只看资源加载成功的回调

![image-20221014171949225](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210141719274.png)

回调比较简单，分两步

一是把加载出来的资源放到对象池，方便之后加载时调用

二是调用内部真正的UI打开逻辑

看看打开UI的逻辑

![image-20221014172216491](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210141722572.png)

关键是 **CreateUIForm**

![image-20221014173338198](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210141733238.png)

很简单，单纯只是把实例化的 **UI** 资源转成 **Go**，然后设置它的父节点为目标 **UIGroup**

有一个小的注意点，我们可以发现这个 **Helper** 是一个接口类型，为什么可以转为 **Mono** 呢？

![image-20221014173855439](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210141738458.png)

原因也很简单，因为实现类继承了这个接口同时也继承了 **Mono**

![image-20221014174005117](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210141740138.png)

所有的 **Group** 类，都会继承这个基类，自然也就可以转成 **Mono** 了

然后调用 **UIFrom** 的初始化方法

**uiForm.OnInit()**

![image-20221014193552979](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210141935060.png)

主要是会获取一个 **Mono** 组件 **UIFormLogic** ，然后进一步调用初始化方法

这个 **UIFormLogic** 是挂载到 **Prefab** 上的，真正 **UI** 的逻辑类

以菜单 **UI** 为例

![image-20221015221746079](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210152217112.png)

![image-20221015221803871](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210152218888.png)

![image-20221015221815591](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210152218606.png)

观察一个运行时的 **UI** 就可以发现一个 **Prefab** 上实际会有2个组件用于相关操作

一个是 **UIForm** 主要用于系统相关的逻辑，一个就是 **UIFormLogic** 用于自身业务逻辑

![image-20221015232809936](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210152328960.png)

**UIFormLogic** 底层的初始化

没啥东西，缓存了一些数据

![image-20221015234942495](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210152349523.png)

然后是 **UGuiForm** 层的初始化

主要是一些初始化设置，与系统无关，这边进行了一次所有 **Text** 组件的本地化操作

![image-20221015235142252](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210152351302.png)

# 细节解析

这是编辑器时的组件列表

![image-20221017151137351](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210171511381.png)

这是运行时，打开第一个UI后的组件列表

![image-20221017151303393](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210171513421.png)

## UI Form Helper

**GF** 的所有模块都遵循这个思路，每个模块都需要有一个逻辑中心（**Helper**），进行真正的逻辑处理，而对应的 **Manager** 只不过是对于 **Helper** 的调用而已

所有模块都提供了一个默认的实现（**Default Helper**），如果你想，你也可以实现自己的逻辑

在 **UIManager** 初始化的时候，会设置用于进行逻辑处理的 **Helper**

![image-20221017152012969](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210171520001.png)

具体配置是可以在编辑器进行的

![image-20221017152037754](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210171520774.png)

处理器需要继承接口 **IUIFormHelper** ，核心方法也只有三个

![image-20221017152131638](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210171521688.png)

## UI Group

**UIGroup** 是在什么时候创建的呢？

在编辑器上可以配置默认的分组

![image-20221017171550748](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210171715784.png)

然后在 **UIComponent** 的 **Start** 生命周期中，会依次添加

![image-20221017171639838](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210171716862.png)

![image-20221017171750161](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202210171717220.png)

主要是在 **Helper.CreateHelper** 方法中，会返回一个 **GameObject**，上面附带了 **UIGroup**，也就是我们默认的一个 **Default Group**

# 实现

## UIForm

大多数都是些常见的生命周期，**OnDepthChanged**需要注意一下

主要用于控制UI的系统生命周期，具体的UI业务逻辑放到具体的UI类中，在这里是没有业务逻辑的

大多数接口都只是转发一下消息，没啥作用

```c#
/// <summary>
/// 界面。
/// </summary>
public sealed class UIForm : MonoBehaviour, IUIForm
{
    private int m_SerialId;
    private string m_UIFormAssetName;
    private IUIGroup m_UIGroup;
    private int m_DepthInUIGroup;
    private bool m_PauseCoveredUIForm;
    private UIFormLogic m_UIFormLogic;

    public int SerialId => m_SerialId;
    public string UIFormAssetName => m_UIFormAssetName;
    public object Handle => gameObject;
    public IUIGroup UIGroup => m_UIGroup;
    public int DepthInUIGroup => m_DepthInUIGroup;
    public bool PauseCoveredUIForm => m_PauseCoveredUIForm;
    public UIFormLogic Logic => m_UIFormLogic;

    /// <summary>
    /// 初始化界面。
    /// </summary>
    /// <param name="serialId">界面序列编号。</param>
    /// <param name="uiFormAssetName">界面资源名称。</param>
    /// <param name="uiGroup">界面所处的界面组。</param>
    /// <param name="pauseCoveredUIForm">是否暂停被覆盖的界面。</param>
    /// <param name="isNewInstance">是否是新实例。</param>
    /// <param name="userData">用户自定义数据。</param>
    public void OnInit(int serialId, string uiFormAssetName, IUIGroup uiGroup, bool pauseCoveredUIForm,
        bool isNewInstance, object userData)
    {
        m_SerialId = serialId;
        m_UIFormAssetName = uiFormAssetName;
        m_UIGroup = uiGroup;
        m_DepthInUIGroup = 0;
        m_PauseCoveredUIForm = pauseCoveredUIForm;

        if (!isNewInstance)
            return;

        m_UIFormLogic = GetComponent<UIFormLogic>();
        if (m_UIFormLogic == null)
            return;

        m_UIFormLogic.OnInit(userData);
    }

    /// <summary>
    /// 界面回收。
    /// </summary>
    public void OnRecycle()
    {
        m_UIFormLogic.OnRecycle();
        m_SerialId = 0;
        m_DepthInUIGroup = 0;
        m_PauseCoveredUIForm = true;
    }
    
    public void OnOpen(object userData){m_UIFormLogic.OnOpen(userData);}
    public void OnClose(bool isShutdown, object userData){m_UIFormLogic.OnClose(isShutdown, userData);}
    public void OnPause(){m_UIFormLogic.OnPause();}
    public void OnResume(){m_UIFormLogic.OnResume();}
    public void OnCover(){m_UIFormLogic.OnCover();}
    public void OnReveal(){m_UIFormLogic.OnReveal();}
    public void OnRefocus(object userData){m_UIFormLogic.OnRefocus(userData);}
    public void OnUpdate(float elapseSeconds, float realElapseSeconds)
    {m_UIFormLogic.OnUpdate(elapseSeconds, realElapseSeconds);}

    /// <summary>
    /// 界面深度改变。
    /// </summary>
    /// <param name="uiGroupDepth">界面组深度。</param>
    /// <param name="depthInUIGroup">界面在界面组中的深度。</param>
    public void OnDepthChanged(int uiGroupDepth, int depthInUIGroup)
    {
        m_DepthInUIGroup = depthInUIGroup;
        m_UIFormLogic.OnDepthChanged(uiGroupDepth, depthInUIGroup);
    }
}
```

## UIFormInfo

UI的一些信息，打开UI的时候会和UI生成到一起

```c#
public sealed class UIFormInfo
{
    private IUIForm m_UIForm;
    private bool m_Paused;
    private bool m_Covered;

    public UIFormInfo()
    {
        m_UIForm = null;
        m_Paused = false;
        m_Covered = false;
    }

    public IUIForm UIForm
    {
        get
        {
            return m_UIForm;
        }
    }

    public bool Paused
    {
        get
        {
            return m_Paused;
        }
        set
        {
            m_Paused = value;
        }
    }

    public bool Covered
    {
        get
        {
            return m_Covered;
        }
        set
        {
            m_Covered = value;
        }
    }

    public static UIFormInfo Create(IUIForm uiForm)
    {
        if (uiForm == null)
            return null;

        UIFormInfo uiFormInfo = new UIFormInfo();
        uiFormInfo.m_UIForm = uiForm;
        uiFormInfo.m_Paused = true;
        uiFormInfo.m_Covered = true;
        return uiFormInfo;
    }
}
```

## UIGroup

界面组，每一个UI都需要放到一个UI组里，UI管理器不会直接控制UI，是通过UI组间接控制的

```c#
/// <summary>
/// 界面组。
/// </summary>
public class UIGroup : IUIGroup
{
    private readonly string m_Name;
    private int m_Depth;
    private bool m_Pause;
    private readonly IUIGroupHelper m_UIGroupHelper;
    private readonly LinkedList<UIFormInfo> m_UIFormInfos;
    private LinkedListNode<UIFormInfo> m_CachedNode;

    /// <summary>
    /// 初始化界面组的新实例。
    /// </summary>
    /// <param name="name">界面组名称。</param>
    /// <param name="depth">界面组深度。</param>
    /// <param name="uiGroupHelper">界面组辅助器。</param>
    public UIGroup(string name, int depth, IUIGroupHelper uiGroupHelper)
    {
        m_Name = name;
        m_Pause = false;
        m_UIGroupHelper = uiGroupHelper;
        m_UIFormInfos = new LinkedList<UIFormInfo>();
        m_CachedNode = null;
        Depth = depth;
    }

    /// <summary>
    /// 获取界面组名称。
    /// </summary>
    public string Name => m_Name;

    /// <summary>
    /// 获取或设置界面组深度。
    /// </summary>
    public int Depth
    {
        get => m_Depth;
        set
        {
            if (m_Depth == value)
            {
                return;
            }

            m_Depth = value;
            m_UIGroupHelper.SetDepth(m_Depth);
            Refresh();
        }
    }

    /// <summary>
    /// 获取或设置界面组是否暂停。
    /// </summary>
    public bool Pause
    {
        get => m_Pause;
        set
        {
            if (m_Pause == value)
            {
                return;
            }

            m_Pause = value;
            Refresh();
        }
    }

    /// <summary>
    /// 获取界面组中界面数量。
    /// </summary>
    public int UIFormCount => m_UIFormInfos.Count;

    /// <summary>
    /// 获取当前界面。
    /// </summary>
    public IUIForm CurrentUIForm => m_UIFormInfos.First?.Value.UIForm;

    /// <summary>
    /// 获取界面组辅助器。
    /// </summary>
    public IUIGroupHelper Helper => m_UIGroupHelper;

    /// <summary>
    /// 界面组轮询。
    /// </summary>
    /// <param name="elapseSeconds">逻辑流逝时间，以秒为单位。</param>
    /// <param name="realElapseSeconds">真实流逝时间，以秒为单位。</param>
    public void Update(float elapseSeconds, float realElapseSeconds)
    {
        LinkedListNode<UIFormInfo> current = m_UIFormInfos.First;
        while (current != null)
        {
            if (current.Value.Paused)
            {
                break;
            }

            m_CachedNode = current.Next;
            current.Value.UIForm.OnUpdate(elapseSeconds, realElapseSeconds);
            current = m_CachedNode;
            m_CachedNode = null;
        }
    }

    /// <summary>
    /// 界面组中是否存在界面。
    /// </summary>
    /// <param name="uiFormAssetName">界面资源名称。</param>
    /// <returns>界面组中是否存在界面。</returns>
    public bool HasUIForm(string uiFormAssetName)
    {
        foreach (UIFormInfo uiFormInfo in m_UIFormInfos)
        {
            if (uiFormInfo.UIForm.UIFormAssetName == uiFormAssetName)
            {
                return true;
            }
        }

        return false;
    }

    /// <summary>
    /// 从界面组中获取界面。
    /// </summary>
    /// <param name="uiFormAssetName">界面资源名称。</param>
    /// <returns>要获取的界面。</returns>
    public IUIForm GetUIForm(string uiFormAssetName)
    {
        foreach (UIFormInfo uiFormInfo in m_UIFormInfos)
        {
            if (uiFormInfo.UIForm.UIFormAssetName == uiFormAssetName)
            {
                return uiFormInfo.UIForm;
            }
        }

        return null;
    }

    /// <summary>
    /// 往界面组增加界面。
    /// </summary>
    /// <param name="uiForm">要增加的界面。</param>
    public void AddUIForm(IUIForm uiForm)
    {
        m_UIFormInfos.AddFirst(UIFormInfo.Create(uiForm));
    }

    /// <summary>
    /// 从界面组移除界面。
    /// </summary>
    /// <param name="uiForm">要移除的界面。</param>
    public void RemoveUIForm(IUIForm uiForm)
    {
        UIFormInfo uiFormInfo = GetUIFormInfo(uiForm);
 
        if (!uiFormInfo.Covered)
        {
            uiFormInfo.Covered = true;
            uiForm.OnCover();
        }

        if (!uiFormInfo.Paused)
        {
            uiFormInfo.Paused = true;
            uiForm.OnPause();
        }

        if (m_CachedNode != null && m_CachedNode.Value.UIForm == uiForm)
        {
            m_CachedNode = m_CachedNode.Next;
        }
    }

    /// <summary>
    /// 激活界面。
    /// </summary>
    /// <param name="uiForm">要激活的界面。</param>
    /// <param name="userData">用户自定义数据。</param>
    public void RefocusUIForm(IUIForm uiForm, object userData)
    {
        UIFormInfo uiFormInfo = GetUIFormInfo(uiForm);

        m_UIFormInfos.Remove(uiFormInfo);
        m_UIFormInfos.AddFirst(uiFormInfo);
    }

    /// <summary>
    /// 刷新界面组。
    /// </summary>
    public void Refresh()
    {
        ...
    }

    private UIFormInfo GetUIFormInfo(IUIForm uiForm)
    {
        foreach (UIFormInfo uiFormInfo in m_UIFormInfos)
        {
            if (uiFormInfo.UIForm == uiForm)
            {
                return uiFormInfo;
            }
        }
        return null;
    }
}
```

大多数方法都比较简单，重点关注一下这个刷新方法

```c#
/// <summary>
/// 刷新界面组。
/// </summary>
public void Refresh()
{
    // First是最新打开的UI
    LinkedListNode<UIFormInfo> current = m_UIFormInfos.First;
    bool pause = m_Pause;
    bool cover = false;
    // 深度，最先打开的UI深度最低，因为是最后打开的UI会被添加到链表的头部
    int depth = UIFormCount;
    while (current != null && current.Value != null)
    {
        LinkedListNode<UIFormInfo> next = current.Next;
        current.Value.UIForm.OnDepthChanged(Depth, depth--);
        if (current.Value == null)
            return;

        if (pause)
        {
            if (!current.Value.Covered)
            {
                current.Value.Covered = true;
                current.Value.UIForm.OnCover();
                if (current.Value == null)
                    return;
            }

            if (!current.Value.Paused)
            {
                current.Value.Paused = true;
                current.Value.UIForm.OnPause();
                if (current.Value == null)
                    return;
            }
        }
        else
        {
            if (current.Value.Paused)
            {
                current.Value.Paused = false;
                current.Value.UIForm.OnResume();
                if (current.Value == null)
                    return;
            }

            // 下一个UI开始都会被暂停
            if (current.Value.UIForm.PauseCoveredUIForm)
            {
                pause = true;
            }

            // 如果当前UI是被覆盖的UI
            if (cover)
            {
                if (!current.Value.Covered)
                {
                    current.Value.Covered = true;
                    current.Value.UIForm.OnCover();
                    if (current.Value == null)
                        return;
                }
            }
            else
            {
                if (current.Value.Covered)
                {
                    current.Value.Covered = false;
                    current.Value.UIForm.OnReveal();
                    if (current.Value == null)
                        return;
                }

                cover = true;
            }
        }

        current = next;
    }
}
```

## IUIGroupHelper

界面组逻辑类需要实现的接口，只有一个改变界面组深度的功能

```c#
/// <summary>
/// 界面组辅助器接口。
/// </summary>
public interface IUIGroupHelper
{
    /// <summary>
    /// 设置界面组深度。
    /// </summary>
    /// <param name="depth">界面组深度。</param>
    void SetDepth(int depth);
}
```

## UIGroupHelperBase

具体的UI组逻辑类继承自这个Base类，原因在于，我们可能采用不同的UI，比如UGUI，FGUI，NGUI，针对不同的UI，你可以实现自己的UI组

```c#
public abstract class UIGroupHelperBase : MonoBehaviour, IUIGroupHelper
{
    /// <summary>
    /// 设置界面组深度。
    /// </summary>
    /// <param name="depth">界面组深度。</param>
    public abstract void SetDepth(int depth);
}
```

## UGUIGroupHelper

```c#
public class UGUIGroupHelper : UIGroupHelperBase
{
    // 两个UI组之间的深度间隔
    public const int DepthFactor = 10000;

    private int m_Depth = 0;
    private Canvas m_CachedCanvas = null;

    /// <summary>
    /// 设置界面组深度。
    /// </summary>
    /// <param name="depth">界面组深度。</param>
    public override void SetDepth(int depth)
    {
        m_Depth = depth;
        m_CachedCanvas.overrideSorting = true;
        m_CachedCanvas.sortingOrder = DepthFactor * depth;
    }

    private void Awake()
    {
        m_CachedCanvas = gameObject.GetOrAddComponent<Canvas>();
        gameObject.GetOrAddComponent<GraphicRaycaster>();
    }

    private void Start()
    {
        m_CachedCanvas.overrideSorting = true;
        m_CachedCanvas.sortingOrder = DepthFactor * m_Depth;

        RectTransform transform = GetComponent<RectTransform>();
        transform.anchorMin = Vector2.zero;
        transform.anchorMax = Vector2.one;
        transform.anchoredPosition = Vector2.zero;
        transform.sizeDelta = Vector2.zero;
    }
}
```

## UIExtension

一个辅助工具类

```c#
public static class UIExtension
{
    private static readonly List<Transform> s_CachedTransforms = new List<Transform>();
    
    /// <summary>
    /// 递归设置游戏对象的层次。
    /// </summary>
    /// <param name="gameObject"><see cref="GameObject" /> 对象。</param>
    /// <param name="layer">目标层次的编号。</param>
    public static void SetLayerRecursively(this GameObject gameObject, int layer)
    {
        gameObject.GetComponentsInChildren(true, s_CachedTransforms);
        for (int i = 0; i < s_CachedTransforms.Count; i++)
        {
            s_CachedTransforms[i].gameObject.layer = layer;
        }

        s_CachedTransforms.Clear();
    }
    
    /// <summary>
    /// 获取或增加组件。
    /// </summary>
    /// <typeparam name="T">要获取或增加的组件。</typeparam>
    /// <param name="gameObject">目标对象。</param>
    /// <returns>获取或增加的组件。</returns>
    public static T GetOrAddComponent<T>(this GameObject gameObject) where T : Component
    {
        T component = gameObject.GetComponent<T>();
        if (component == null)
        {
            component = gameObject.AddComponent<T>();
        }

        return component;
    }
}
```

## UIFormLogic

这是UI的业务逻辑基类，是一个抽象类，具体的UI需要有自身对应的具体业务逻辑类

大部分内容都是一些对应的生命周期

```c#
public abstract class UIFormLogic : MonoBehaviour
{
    private bool m_Available = false;
    private bool m_Visible = false;
    private UIForm m_UIForm = null;
    private Transform m_CachedTransform = null;
    private int m_OriginalLayer = 0;

    /// <summary>
    /// 获取界面。
    /// </summary>
    public UIForm UIForm
    {
        get { return m_UIForm; }
    }

    /// <summary>
    /// 获取或设置界面名称。
    /// </summary>
    public string Name
    {
        get { return gameObject.name; }
        set { gameObject.name = value; }
    }

    /// <summary>
    /// 获取界面是否可用。
    /// </summary>
    public bool Available
    {
        get { return m_Available; }
    }

    /// <summary>
    /// 获取或设置界面是否可见。
    /// </summary>
    public bool Visible
    {
        get { return m_Available && m_Visible; }
        set
        {
            if (!m_Available)
            {
                return;
            }

            if (m_Visible == value)
            {
                return;
            }

            m_Visible = value;
            InternalSetVisible(value);
        }
    }

    /// <summary>
    /// 获取已缓存的 Transform。
    /// </summary>
    public Transform CachedTransform
    {
        get { return m_CachedTransform; }
    }

    protected virtual void OnInit(object userData)
    {
        if (m_CachedTransform == null)
        {
            m_CachedTransform = transform;
        }

        m_UIForm = GetComponent<UIForm>();
        m_OriginalLayer = gameObject.layer;
    }
    protected virtual void OnRecycle(){}
    protected virtual void OnOpen(object userData)
    {
        m_Available = true;
        Visible = true;
    }
    protected virtual void OnClose(bool isShutdown, object userData)
    {
        gameObject.SetLayerRecursively(m_OriginalLayer);
        Visible = false;
        m_Available = false;
    }
    protected virtual void OnPause(){Visible = false;}
    protected virtual void OnResume(){Visible = true;}
    protected virtual void OnCover(){}
    protected virtual void OnReveal(){}
    protected virtual void OnRefocus(object userData){}
    protected virtual void OnUpdate(float elapseSeconds, float realElapseSeconds){}

    /// <summary>
    /// 界面深度改变。
    /// </summary>
    /// <param name="uiGroupDepth">界面组深度。</param>
    /// <param name="depthInUIGroup">界面在界面组中的深度。</param>
    protected virtual void OnDepthChanged(int uiGroupDepth, int depthInUIGroup){}

    /// <summary>
    /// 设置界面的可见性。
    /// </summary>
    /// <param name="visible">界面的可见性。</param>
    protected virtual void InternalSetVisible(bool visible){gameObject.SetActive(visible);}
}
```

## UGUIForm

大部分内容还是只是一些生命周期，没什么特殊操作，除了深度改变时需要重写

为什么会有这一层呢，因为UIFormLogic层是通用的业务逻辑层，而这一层封装则是针对不同的UI来说的

比如也可以有NGUIForm，FGUIForm等

```c#
public abstract class UGUIForm : UIFormLogic
{
    public const int DepthFactor = 100;
    private const float FadeTime = 0.3f;

    private static Font s_MainFont = null;
    private Canvas m_CachedCanvas = null;
    private CanvasGroup m_CanvasGroup = null;
    private List<Canvas> m_CachedCanvasContainer = new List<Canvas>();

    public int OriginalDepth { get; private set; }

    public int Depth
    {
        get { return m_CachedCanvas.sortingOrder; }
    }

    public void Close(bool ignoreFade = false)
    {
        UIComponent.instance.CloseUIForm(UIForm);
    }

    protected override void OnInit(object userData)
    {
        base.OnInit(userData);

        m_CachedCanvas = gameObject.GetOrAddComponent<Canvas>();
        m_CachedCanvas.overrideSorting = true;
        OriginalDepth = m_CachedCanvas.sortingOrder;

        m_CanvasGroup = gameObject.GetOrAddComponent<CanvasGroup>();

        RectTransform transform = GetComponent<RectTransform>();
        transform.anchorMin = Vector2.zero;
        transform.anchorMax = Vector2.one;
        transform.anchoredPosition = Vector2.zero;
        transform.sizeDelta = Vector2.zero;

        gameObject.GetOrAddComponent<GraphicRaycaster>();
    }

    protected override void OnRecycle()
    {base.OnRecycle();}
    protected override void OnOpen(object userData)
    {base.OnOpen(userData);}
    protected override void OnClose(bool isShutdown, object userData)
    {base.OnClose(isShutdown, userData);}
    protected override void OnPause()
    {base.OnPause();}
    protected override void OnResume()
    {base.OnResume();}
    protected override void OnCover(){base.OnCover();}
    protected override void OnReveal()
    {base.OnReveal();}
    protected override void OnRefocus(object userData)
    {base.OnRefocus(userData);}
    protected override void OnUpdate(float elapseSeconds, float realElapseSeconds)
    { base.OnUpdate(elapseSeconds, realElapseSeconds);}

    protected override void OnDepthChanged(int uiGroupDepth, int depthInUIGroup)
    {
        int oldDepth = Depth;
        base.OnDepthChanged(uiGroupDepth, depthInUIGroup);
        int deltaDepth = UGUIGroupHelper.DepthFactor * uiGroupDepth + DepthFactor * depthInUIGroup - oldDepth + OriginalDepth;
        GetComponentsInChildren(true, m_CachedCanvasContainer);
        for (int i = 0; i < m_CachedCanvasContainer.Count; i++)
        {
            m_CachedCanvasContainer[i].sortingOrder += deltaDepth;
        }

        m_CachedCanvasContainer.Clear();
    }
}
```

## IUIFormHelper

UI操作的核心逻辑，UIManager是对于Helper的代理

```c#
/// <summary>
/// 界面辅助器接口。
/// </summary>
public interface IUIFormHelper
{
    /// <summary>
    /// 实例化界面。
    /// </summary>
    /// <param name="uiFormAsset">要实例化的界面资源。</param>
    /// <returns>实例化后的界面。</returns>
    object InstantiateUIForm(object uiFormAsset);

    /// <summary>
    /// 创建界面。
    /// </summary>
    /// <param name="uiFormInstance">界面实例。</param>
    /// <param name="uiGroup">界面所属的界面组。</param>
    /// <param name="userData">用户自定义数据。</param>
    /// <returns>界面。</returns>
    IUIForm CreateUIForm(object uiFormInstance, IUIGroup uiGroup, object userData);

    /// <summary>
    /// 释放界面。
    /// </summary>
    /// <param name="uiFormAsset">要释放的界面资源。</param>
    /// <param name="uiFormInstance">要释放的界面实例。</param>
    void ReleaseUIForm(object uiFormAsset, object uiFormInstance);
}
```

## UIFormHelperBase

界面操作器基类，在封装一层的目的是单纯只是继承Mono

```c#
public abstract class UIFormHelperBase : MonoBehaviour,IUIFormHelper
{
    public abstract object InstantiateUIForm(object uiFormAsset);
    public abstract IUIForm CreateUIForm(object uiFormInstance, IUIGroup uiGroup, object userData);
    public abstract void ReleaseUIForm(object uiFormAsset, object uiFormInstance);
}
```

## DefaultUIFormHelper

默认的界面操作器，只要继承接口，就可以实现我们自己的逻辑

主要就是加载和释放资源，创建UI

创建UI的时候会在Go上添加一个UIForm组件

```c#
public class DefaultUIFormHelper : UIFormHelperBase
{
    /// <summary>
    /// 实例化界面。
    /// </summary>
    /// <param name="uiFormAsset">要实例化的界面资源。</param>
    /// <returns>实例化后的界面。</returns>
    public override object InstantiateUIForm(object uiFormAsset)
    {
        return Instantiate((Object)uiFormAsset);
    }

    /// <summary>
    /// 创建界面。
    /// </summary>
    /// <param name="uiFormInstance">界面实例。</param>
    /// <param name="uiGroup">界面所属的界面组。</param>
    /// <param name="userData">用户自定义数据。</param>
    /// <returns>界面。</returns>
    public override IUIForm CreateUIForm(object uiFormInstance, IUIGroup uiGroup, object userData)
    {
        GameObject gameObject = uiFormInstance as GameObject;
        if (gameObject == null)
            return null;

        Transform transform = gameObject.transform;
        transform.SetParent(((MonoBehaviour)uiGroup.Helper).transform);
        transform.localScale = Vector3.one;

        return gameObject.GetOrAddComponent<UIForm>();
    }

    /// <summary>
    /// 释放界面。
    /// </summary>
    /// <param name="uiFormAsset">要释放的界面资源。</param>
    /// <param name="uiFormInstance">要释放的界面实例。</param>
    public override void ReleaseUIForm(object uiFormAsset, object uiFormInstance)
    {
        Resources.UnloadAsset((Object)uiFormAsset);
        Destroy((Object)uiFormInstance);
    }
}
```

## Event

一些事件参数

### OpenUIFormSuccessEventArgs

```c#
public class OpenUIFormSuccessEventArgs
{
    /// <summary>
    /// 获取打开成功的界面。
    /// </summary>
    public IUIForm UIForm
    {
        get;
        private set;
    }

    /// <summary>
    /// 获取加载持续时间。
    /// </summary>
    public float Duration
    {
        get;
        private set;
    }

    /// <summary>
    /// 获取用户自定义数据。
    /// </summary>
    public object UserData
    {
        get;
        private set;
    }

    /// <summary>
    /// 创建打开界面成功事件。
    /// </summary>
    /// <param name="uiForm">加载成功的界面。</param>
    /// <param name="duration">加载持续时间。</param>
    /// <param name="userData">用户自定义数据。</param>
    /// <returns>创建的打开界面成功事件。</returns>
    public OpenUIFormSuccessEventArgs (IUIForm uiForm, float duration, object userData)
    {
        UIForm = uiForm;
        Duration = duration;
        UserData = userData;
    }
}
```

### CloseUIFormCompleteEventArgs

```c#
public class CloseUIFormCompleteEventArgs
{
    /// <summary>
    /// 获取界面序列编号。
    /// </summary>
    public int SerialId { get; private set; }

    /// <summary>
    /// 获取界面资源名称。
    /// </summary>
    public string UIFormAssetName { get; private set; }

    /// <summary>
    /// 获取界面所属的界面组。
    /// </summary>
    public IUIGroup UIGroup { get; private set; }

    /// <summary>
    /// 获取用户自定义数据。
    /// </summary>
    public object UserData { get; private set; }

    /// <summary>
    /// 创建关闭界面完成事件。
    /// </summary>
    /// <param name="serialId">界面序列编号。</param>
    /// <param name="uiFormAssetName">界面资源名称。</param>
    /// <param name="uiGroup">界面所属的界面组。</param>
    /// <param name="userData">用户自定义数据。</param>
    /// <returns>创建的关闭界面完成事件。</returns>
    public CloseUIFormCompleteEventArgs(int serialId, string uiFormAssetName, IUIGroup uiGroup, object userData)
    {
        SerialId = serialId;
        UIFormAssetName = uiFormAssetName;
        UIGroup = uiGroup;
        UserData = userData;
    }
}
```

## UIManager

界面管理器，删减掉了很多接口，目前是一些很基础的接口

```c#
/// <summary>
/// 界面管理器。
/// </summary>
internal sealed partial class UIManager
{
    private readonly Dictionary<string, UIGroup> m_UIGroups;
    private IUIFormHelper m_UIFormHelper;
    private int m_Serial;
    
    private EventHandler<OpenUIFormSuccessEventArgs> m_OpenUIFormSuccessEventHandler;
    private EventHandler<CloseUIFormCompleteEventArgs> m_CloseUIFormCompleteEventHandler;

    /// <summary>
    /// 初始化界面管理器的新实例。
    /// </summary>
    public UIManager()
    {
        m_UIGroups = new Dictionary<string, UIGroup>(StringComparer.Ordinal);
        m_UIFormHelper = null;
        m_Serial = 0;
        m_OpenUIFormSuccessEventHandler = null;
        m_CloseUIFormCompleteEventHandler = null;
    }


    /// <summary>
    /// 打开界面成功事件。
    /// </summary>
    public event EventHandler<OpenUIFormSuccessEventArgs> OpenUIFormSuccess
    {
        add => m_OpenUIFormSuccessEventHandler += value;
        remove => m_OpenUIFormSuccessEventHandler -= value;
    }

    /// <summary>
    /// 关闭界面完成事件。
    /// </summary>
    public event EventHandler<CloseUIFormCompleteEventArgs> CloseUIFormComplete
    {
        add => m_CloseUIFormCompleteEventHandler += value;
        remove => m_CloseUIFormCompleteEventHandler -= value;
    }

    /// <summary>
    /// 界面管理器轮询。
    /// </summary>
    /// <param name="elapseSeconds">逻辑流逝时间，以秒为单位。</param>
    /// <param name="realElapseSeconds">真实流逝时间，以秒为单位。</param>
    internal void Update(float elapseSeconds, float realElapseSeconds)
    {
        foreach (KeyValuePair<string, UIGroup> uiGroup in m_UIGroups)
        {
            uiGroup.Value.Update(elapseSeconds, realElapseSeconds);
        }
    }

    /// <summary>
    /// 设置界面辅助器。
    /// </summary>
    /// <param name="uiFormHelper">界面辅助器。</param>
    public void SetUIFormHelper(IUIFormHelper uiFormHelper)
    {
        m_UIFormHelper = uiFormHelper;
    }

    /// <summary>
    /// 是否存在界面组。
    /// </summary>
    /// <param name="uiGroupName">界面组名称。</param>
    /// <returns>是否存在界面组。</returns>
    public bool HasUIGroup(string uiGroupName)
    {
        return m_UIGroups.ContainsKey(uiGroupName);
    }

    /// <summary>
    /// 获取界面组。
    /// </summary>
    /// <param name="uiGroupName">界面组名称。</param>
    /// <returns>要获取的界面组。</returns>
    public IUIGroup GetUIGroup(string uiGroupName)
    {
        UIGroup uiGroup = null;
        if (m_UIGroups.TryGetValue(uiGroupName, out uiGroup))
        {
            return uiGroup;
        }
        return null;
    }


    /// <summary>
    /// 增加界面组。
    /// </summary>
    /// <param name="uiGroupName">界面组名称。</param>
    /// <param name="uiGroupHelper">界面组辅助器。</param>
    /// <returns>是否增加界面组成功。</returns>
    public bool AddUIGroup(string uiGroupName, IUIGroupHelper uiGroupHelper)
    {
        return AddUIGroup(uiGroupName, 0, uiGroupHelper);
    }

    /// <summary>
    /// 增加界面组。
    /// </summary>
    /// <param name="uiGroupName">界面组名称。</param>
    /// <param name="uiGroupDepth">界面组深度。</param>
    /// <param name="uiGroupHelper">界面组辅助器。</param>
    /// <returns>是否增加界面组成功。</returns>
    public bool AddUIGroup(string uiGroupName, int uiGroupDepth, IUIGroupHelper uiGroupHelper)
    {
        if (HasUIGroup(uiGroupName))
        {
            return false;
        }

        m_UIGroups.Add(uiGroupName, new UIGroup(uiGroupName, uiGroupDepth, uiGroupHelper));

        return true;
    }

    /// <summary>
    /// 是否存在界面。
    /// </summary>
    /// <param name="uiFormAssetName">界面资源名称。</param>
    /// <returns>是否存在界面。</returns>
    public bool HasUIForm(string uiFormAssetName)
    {
        foreach (KeyValuePair<string, UIGroup> uiGroup in m_UIGroups)
        {
            if (uiGroup.Value.HasUIForm(uiFormAssetName))
            {
                return true;
            }
        }

        return false;
    }


    /// <summary>
    /// 获取界面。
    /// </summary>
    /// <param name="uiFormAssetName">界面资源名称。</param>
    /// <returns>要获取的界面。</returns>
    public IUIForm GetUIForm(string uiFormAssetName)
    {
        foreach (KeyValuePair<string, UIGroup> uiGroup in m_UIGroups)
        {
            IUIForm uiForm = uiGroup.Value.GetUIForm(uiFormAssetName);
            if (uiForm != null)
            {
                return uiForm;
            }
        }

        return null;
    }

    /// <summary>
    /// 打开界面。
    /// </summary>
    /// <param name="uiFormAssetName">界面资源名称。</param>
    /// <param name="uiGroupName">界面组名称。</param>
    /// <param name="priority">加载界面资源的优先级。</param>
    /// <param name="pauseCoveredUIForm">是否暂停被覆盖的界面。</param>
    /// <param name="userData">用户自定义数据。</param>
    /// <returns>界面的序列编号。</returns>
    public int OpenUIForm(string uiFormAssetName, string uiGroupName, int priority, bool pauseCoveredUIForm, object userData)
    {
        if (m_UIFormHelper == null)
            throw new Exception("You must set UI form helper first.");

        UIGroup uiGroup = (UIGroup) GetUIGroup(uiGroupName);
        if (uiGroup == null)
            throw new Exception($"UI group '{uiGroupName}' is not exist.");

        int serialId = ++m_Serial;
        var asset = Resources.Load(uiFormAssetName);
        if (asset == null)
            throw new Exception($"Load '{uiGroupName}' Fail.");
        var instance = Object.Instantiate(asset);
        InternalOpenUIForm(serialId, uiFormAssetName, uiGroup, instance, pauseCoveredUIForm, true, 0f, userData);

        return serialId;
    }

    /// <summary>
    /// 关闭界面。
    /// </summary>
    /// <param name="uiForm">要关闭的界面。</param>
    /// <param name="userData">用户自定义数据。</param>
    public void CloseUIForm(IUIForm uiForm, object userData)
    {
        if (uiForm == null)
            throw new Exception("UI form is invalid.");

        UIGroup uiGroup = (UIGroup) uiForm.UIGroup;
        if (uiGroup == null)
            throw new Exception("UI group is invalid.");

        uiGroup.RemoveUIForm(uiForm);
        uiForm.OnClose(false, userData);
        uiGroup.Refresh();

        if (m_CloseUIFormCompleteEventHandler != null)
        {
            CloseUIFormCompleteEventArgs closeUIFormCompleteEventArgs = new CloseUIFormCompleteEventArgs(uiForm.SerialId, uiForm.UIFormAssetName, uiGroup, userData);
            m_CloseUIFormCompleteEventHandler(this, closeUIFormCompleteEventArgs);
        }
    }


    /// <summary>
    /// 激活界面。
    /// </summary>
    /// <param name="uiForm">要激活的界面。</param>
    public void RefocusUIForm(IUIForm uiForm)
    {
        RefocusUIForm(uiForm, null);
    }

    /// <summary>
    /// 激活界面。
    /// </summary>
    /// <param name="uiForm">要激活的界面。</param>
    /// <param name="userData">用户自定义数据。</param>
    public void RefocusUIForm(IUIForm uiForm, object userData)
    {
        UIGroup uiGroup = (UIGroup) uiForm.UIGroup;
        uiGroup.RefocusUIForm(uiForm, userData);
        uiGroup.Refresh();
        uiForm.OnRefocus(userData);
    }

    private void InternalOpenUIForm(int serialId, string uiFormAssetName, UIGroup uiGroup, object uiFormInstance, bool pauseCoveredUIForm, bool isNewInstance, float duration, object userData)
    {
        IUIForm uiForm = m_UIFormHelper.CreateUIForm(uiFormInstance, uiGroup, userData);
        if (uiForm == null)
            throw new Exception("Can not create UI form in UI form helper.");

        uiForm.OnInit(serialId, uiFormAssetName, uiGroup, pauseCoveredUIForm, isNewInstance, userData);
        uiGroup.AddUIForm(uiForm);
        uiForm.OnOpen(userData);
        uiGroup.Refresh();

        if (m_OpenUIFormSuccessEventHandler != null)
        {
            OpenUIFormSuccessEventArgs openUIFormSuccessEventArgs = new OpenUIFormSuccessEventArgs(uiForm, duration, userData);
            m_OpenUIFormSuccessEventHandler(this, openUIFormSuccessEventArgs);
        }
    }
}
```

## UICompoent

最后是一个代理类，Mono类，用于控制UI系统的生命周期，以及提供对外接口，在GF中，所有最上层的接口都通过Component进行，所有组件都会注册到一起

但我们现在单独写一个UI框架，就跳过这一步，简单写一个单例类

```c#
public class UIComponent : MonoBehaviour
{
    public static UIComponent instance;
    private UIManager m_UIManager;
    private const int DefaultPriority = 0;

    [SerializeField] 
    private Transform m_InstanceRoot = null;


    private void Awake()
    {
        if (instance != null)
        {
            Destroy(this);
            return;
        }

        instance = this;

        m_UIManager = new UIManager();
        
        var uiFormHelper = new GameObject().AddComponent<DefaultUIFormHelper>();
        uiFormHelper.name = "UI Form Helper";
        var trans = uiFormHelper.transform;
        trans.SetParent(transform);
        trans.localScale = Vector3.one;

        m_UIManager.SetUIFormHelper(uiFormHelper);

        if (m_InstanceRoot == null)
        {
            m_InstanceRoot = new GameObject("UI Form Instances").transform;
            m_InstanceRoot.SetParent(gameObject.transform);
            m_InstanceRoot.localScale = Vector3.one;
        }

        m_InstanceRoot.gameObject.layer = LayerMask.NameToLayer("UI");

        AddUIGroup("Default", 999);
    }
    
    private void Update()
    {
        m_UIManager.Update(0, 0);
    }
    
    /// <summary>
    /// 是否存在界面组。
    /// </summary>
    /// <param name="uiGroupName">界面组名称。</param>
    /// <returns>是否存在界面组。</returns>
    public bool HasUIGroup(string uiGroupName)
    {
        return m_UIManager.HasUIGroup(uiGroupName);
    }

    /// <summary>
    /// 获取界面组。
    /// </summary>
    /// <param name="uiGroupName">界面组名称。</param>
    /// <returns>要获取的界面组。</returns>
    public IUIGroup GetUIGroup(string uiGroupName)
    {
        return m_UIManager.GetUIGroup(uiGroupName);
    }

    /// <summary>
    /// 增加界面组。
    /// </summary>
    /// <param name="uiGroupName">界面组名称。</param>
    /// <param name="depth">界面组深度。</param>
    /// <returns>是否增加界面组成功。</returns>
    public bool AddUIGroup(string uiGroupName, int depth)
    {
        if (m_UIManager.HasUIGroup(uiGroupName))
        {
            return false;
        }

        UIGroupHelperBase uiGroupHelper = new GameObject().AddComponent<UGUIGroupHelper>();

        uiGroupHelper.name = $"UI Group - {uiGroupName}";
        uiGroupHelper.gameObject.layer = LayerMask.NameToLayer("UI");
        Transform trans = uiGroupHelper.transform;
        trans.SetParent(m_InstanceRoot);
        trans.localScale = Vector3.one;

        return m_UIManager.AddUIGroup(uiGroupName, depth, uiGroupHelper);
    }

    /// <summary>
    /// 是否存在界面。
    /// </summary>
    /// <param name="uiFormAssetName">界面资源名称。</param>
    /// <returns>是否存在界面。</returns>
    public bool HasUIForm(string uiFormAssetName)
    {
        return m_UIManager.HasUIForm(uiFormAssetName);
    }

    /// <summary>
    /// 获取界面。
    /// </summary>
    /// <param name="uiFormAssetName">界面资源名称。</param>
    /// <returns>要获取的界面。</returns>
    public UIForm GetUIForm(string uiFormAssetName)
    {
        return (UIForm) m_UIManager.GetUIForm(uiFormAssetName);
    }


    /// <summary>
    /// 打开界面。
    /// </summary>
    /// <param name="uiFormAssetName">界面资源名称。</param>
    /// <param name="uiGroupName">界面组名称。</param>
    /// <param name="priority">加载界面资源的优先级。</param>
    /// <param name="pauseCoveredUIForm">是否暂停被覆盖的界面。</param>
    /// <param name="userData">用户自定义数据。</param>
    /// <returns>界面的序列编号。</returns>
    public int OpenUIForm(string uiFormAssetName, string uiGroupName, int priority = DefaultPriority, bool pauseCoveredUIForm = false, object userData = null)
    {
        return m_UIManager.OpenUIForm(uiFormAssetName, uiGroupName, priority, pauseCoveredUIForm, userData);
    }

    /// <summary>
    /// 关闭界面。
    /// </summary>
    /// <param name="uiForm">要关闭的界面。</param>
    /// <param name="userData">用户自定义数据。</param>
    public void CloseUIForm(UIForm uiForm, object userData = null)
    {
        m_UIManager.CloseUIForm(uiForm, userData);
    }

    /// <summary>
    /// 激活界面。
    /// </summary>
    /// <param name="uiForm">要激活的界面。</param>
    /// <param name="userData">用户自定义数据。</param>
    public void RefocusUIForm(UIForm uiForm, object userData)
    {
        m_UIManager.RefocusUIForm(uiForm, userData);
    }


    public void AddOpenUIFormSuccessEvent(object sender, EventHandler<OpenUIFormSuccessEventArgs> e)
    {
        m_UIManager.OpenUIFormSuccess += e;
    }
    
    public void RemoveOpenUIFormSuccessEvent(object sender, EventHandler<OpenUIFormSuccessEventArgs> e)
    {
        m_UIManager.OpenUIFormSuccess -= e;
    }
    
    public void AddCloseUIFormCompleteEvent(object sender, EventHandler<CloseUIFormCompleteEventArgs> e)
    {
        m_UIManager.CloseUIFormComplete += e;
    }
    
    public void RemoveCloseUIFormCompleteEvent(object sender, EventHandler<CloseUIFormCompleteEventArgs> e)
    {
        m_UIManager.CloseUIFormComplete -= e;
    }
}
```

# 测试

添加组件到一个GO上面

![image-20221105215511639](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202211052155691.png)

制作两个测试UI

![image-20221105215548803](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202211052155828.png)

分别写两个脚本挂载到对应的Prefab上，名称随意，但要继承UGUIForm

```c#
public class Test1Logic : UGUIForm
{
    public Button open;

    private int data;

    protected override void OnOpen(object userData)
    {
        base.OnOpen(userData);

        data = (int) userData;
        
        UIComponent.instance.AddOpenUIFormSuccessEvent(((sender, args) =>
        {
            Debug.Log($"open {(int) args.UserData}");
        }));
        
        open.onClick.RemoveAllListeners();
        open.onClick.AddListener((() =>
        {
            UIComponent.instance.OpenUIForm("Test2", "Default", 0, false, 999);
        }));
    }

    protected override void OnUpdate(float elapseSeconds, float realElapseSeconds)
    {
        base.OnUpdate(elapseSeconds, realElapseSeconds);
        Debug.Log(data);
    }
}
```

```c#
public class Test2Logic : UGUIForm
{
    public Button close;
    private int data;
    
    protected override void OnInit(object userData)
    {
        base.OnInit(userData);
        data = (int) userData;
    }

    protected override void OnOpen(object userData)
    {
        base.OnOpen(userData);
        close.onClick.RemoveAllListeners();
        close.onClick.AddListener((() =>
        {
            UIComponent.instance.CloseUIForm(UIForm);
        }));
    }

    protected override void OnUpdate(float elapseSeconds, float realElapseSeconds)
    {
        base.OnUpdate(elapseSeconds, realElapseSeconds);
        Debug.Log(data);
    }
}
```

![1](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202211052308560.gif)

两个Update一起跑是因为打开UI的时候没有设置暂停上一个UI









































