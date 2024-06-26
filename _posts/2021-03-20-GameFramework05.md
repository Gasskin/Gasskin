---
title: "GameFramework|05 UI管理器03"
categories: GameFramework 
tags: 框架
---

### 数据结构设计

我们还不能直接开始设计UI的开闭接口，必须先确定好，应该用什么样的一种格式来储存所有的UI脚本，其实很简单

我们能确定一共会有多少个UI吗？这不一定。

我们需要随机访问UI吗？一般没这个需求。

我们可能会频繁创建和删除UI吗？这倒是有可能的。

注意另一个情况，一般来说，我们游戏的UI打开是递进的，关闭UI同样如此，比如说，我们先打开商城，然后购买时弹出确认提示，此时我们应该先关闭确认提示，然后才能关闭商城，是不是很像先进后出的数据结构？

那么很明显了，其实没啥特殊的，我们用栈来维护所有UI就行了，但不能只有一个栈，因为3种UI基本上是独立的，所以在这里我设计了3个栈，分别维护三种界面UI，同时，还需要一个字典类型，记录了所有已经被载入的UI。

```c#
public class UIManager : ManagerBase
{
    // 分别缓存已经加载的三种UI，被加载过不代表正在显示
    private Stack<UIBase> allFixedUI;
    private Stack<UIBase> allNormalUI;
    private Stack<UIBase> allPopUpUI;
    
    /// 这里记录了所有已经被加载的UI，其实就是上面三个之和
    private Dictionary<string, UIBase> uiLoaded;

    /// 当前打开的UI
    private UIBase currentUI;
    

    public override void Init()
    {
        // 初始化
        allFixedUI = new Stack<UIBase>();
        allNormalUI = new Stack<UIBase>();
        allPopUpUI = new Stack<UIBase>();
        
        uiLoaded = new Dictionary<string, UIBase>();
        
        currentUI = null;
        
        // 保存各个节点的信息
        if (null == uiRoot)
        {
            // ...
        }
    }
}

```

### 打开UI

好了，这次是真的要写打开UI的逻辑了......

首先，我们判断当前想要打开的这个UI有没有加载过，如果没有，那就先加载，否则就直接打开

```c#
public class UIManager : ManagerBase
{
    #region 接口方法

    public void OpenUI(UIStruct data)
    {
        // 如果这个UI还没被加载，那需要先加载
        if (!uiLoaded.TryGetValue(data.name,out var ui)) 
        {
            LoadUI(data,out ui);
        }
        // 显示UI
        ShowUI(ui);
    }

    #endregion
}
```

加载UI

```c#
public class UIManager : ManagerBase
{
    #region 工具方法

    /// <summary>
    /// 加载UI
    /// </summary>
    private void LoadUI(UIStruct data, out UIBase ui)
    {
        GameObject prefab = null;

        if (String.IsNullOrEmpty(data.path))
        {
            throw new Exception("非法的UI路径");
        }

        // 加载预制体
        prefab = Resources.Load<GameObject>(data.path);
        prefab = GameObject.Instantiate(prefab);

        if (prefab == null)
        {
            throw new Exception($"加载UI Prefab失败，请检查是否存在预制体：{data.path}");
        }

        // 利用反射来挂在脚本，这样我们就不用手动把脚本拖放到预制体上了
        Type script = Type.GetType(data.name);

        if (script == null)
        {
            throw new Exception($"加载脚本失败，请检查是否存在脚本：{data.name}");
        }
        
        // 挂载脚本
        ui = prefab.AddComponent(script) as UIBase;
        uiLoaded.Add(data.name, ui);

        // 根据UI类型，存放到不同的节点中
        switch (ui.type)
        {
            case UIType.Fixed:
                prefab.transform.SetParent(fixedRoot,false);
                break;
            case UIType.Normal:
                prefab.transform.SetParent(normalRoot,false);
                break;
            case UIType.PopUp:
                prefab.transform.SetParent(popUpRoot,false);
                break;
        }
    }
    
    #endregion
}
```

显示UI

```c#
public class UIManager : ManagerBase
{
    #region 工具方法

    /// <summary>
    /// 显示UI 
    /// </summary>
    private void ShowUI(UIBase ui)
    {
        switch (ui.type)
        {
            case UIType.Fixed:
                PushToStack(ui,allFixedUI);
                break;
            case UIType.Normal:
                PushToStack(ui,allNormalUI);
                break;
            case UIType.PopUp:
                PushToStack(ui,allPopUpUI);
                break;
        }
    }

    /// <summary>
    /// 将UI加入对应栈，并真正的控制当前UI的显示隐藏逻辑
    /// </summary>
    private void PushToStack(UIBase ui, Stack<UIBase> stack)
    {
        // 打开一个新的UI，当前UI必然被冻结，但未必会隐藏，打开Normal会隐藏Normal但不会隐藏Fixed
        if (currentUI != null) 
        {
            // 冻结当前的界面
            currentUI.Freeze();
            // 如果打开的不是Pop界面，且打开的界面和当前界面是同类型的，那才需要关闭当前的界面
            if (ui.type != UIType.PopUp && ui.type == currentUI.type)  
            {
                currentUI.Close();
                currentUI.gameObject.SetActive(false);
            }
        }
        if (!ui.gameObject.activeSelf)
        {
            ui.gameObject.SetActive(true);
        }
        ui.Show();
        stack.Push(ui);
        currentUI = ui;
    }

    #endregion
}
```

当然我们还需要创建一个和TestUI Prefab对应的UI 脚本，注意两点，一是需要继承自UIBase，二是脚本名和预制体名一定要相同

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/15.png)

```c#
public class TestUI : UIBase
{
    public override void Load()
    {
        
    }

    public override void UnLoad()
    {
        
    }

    public override void Show()
    {
        Debug.Log("Test Show!");
    }
}

```

编写一个脚本测试一下，应该是没有问题的~

```c#
public class test : MonoBehaviour
{
    private UIManager uiManager;
    void Start()
    {
        uiManager = SGFEntry.Instance.GetManager<UIManager>();
        uiManager.OpenUI(UIs.TestUI);
    }
}
```

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/16.png)

### 关闭UI

```c#
public class UIManager : ManagerBase
{
    #region 接口方法
        
    /// <summary>
    /// 关闭当前UI，注意此方法只关闭当前打开的最顶层UI
    /// </summary>
    public void CloseCurrent()
    {
        if (currentUI == null) 
        {
            Debug.LogError("没有可以关闭的界面");
            return;
        }

        switch (currentUI.type)
        {
            case UIType.Fixed:
                PopFromStack(allFixedUI);
                break;
            case UIType.Normal:
                PopFromStack(allNormalUI);
                break;
            case UIType.PopUp:
                PopFromStack(allPopUpUI);
                break;
        }
    }

    #endregion

    #region 工具方法
        
    /// <summary>
    /// 将UI从对应链表中移除，并真正的控制当前UI的关闭 
    /// </summary>
    private void PopFromStack(Stack<UIBase> stack)
    {
        // 首先关闭并冻结当前的UI
        currentUI.Close();
        currentUI.Freeze();
        currentUI.gameObject.SetActive(false);
        // 栈顶UI出栈，其实就是currentUI
        stack.Pop();
        // 然后尝试获取当前UI栈中的上一个UI
        if (stack.Count >= 1)
        {
            currentUI = stack.Peek();
            // 注意，如果当前栈是Pop栈，那么我们关闭顶层的弹窗是不需要重新激活上一个弹窗的，因为Pop类型UI之间不会相互关闭，我们只需要解冻就好
            if (stack != allPopUpUI) 
            {
                currentUI.Show();
                currentUI.gameObject.SetActive(true);
            }
            currentUI.UnFreeze();
        }
        // 如果当前栈里没有UI了，那就获取上一个栈中的UI
        else
        {
            // 如果当前栈是Pop，那么currentUI就是Noraml栈中的最后一个元素，其余同理
            // 另外，因为不同类型的UI之间也不会相互关闭，所以我们也只需要解冻就好
            if (stack == allPopUpUI)
            {
                currentUI = allNormalUI.Peek();
                currentUI.UnFreeze();
            }
            else if (stack == allNormalUI)
            {
                currentUI = allFixedUI.Peek();
                currentUI.UnFreeze();
            }
            else
            {
                currentUI = null;
            }
        }
    }
    
    #endregion
}
```

编写几个UI来测试一下~

<img src="https://www.logarius996.icu/images/SimpleGameFramework/UI/17.png" alt="1" style="zoom:50%;" />

<img src="https://www.logarius996.icu/images/SimpleGameFramework/UI/18.png" alt="1" style="zoom:50%;" />

<img src="https://www.logarius996.icu/images/SimpleGameFramework/UI/19.png" alt="1" style="zoom:50%;" />

<img src="https://www.logarius996.icu/images/SimpleGameFramework/UI/20.png" alt="1" style="zoom:50%;" />

<img src="https://www.logarius996.icu/images/SimpleGameFramework/UI/21.png" alt="1" style="zoom:50%;" />

<img src="https://www.logarius996.icu/images/SimpleGameFramework/UI/22.png" alt="1" style="zoom:50%;" />

再编写对应的脚本，我就不全截出来了，注意一定要在Load方法里设置UI的默认类型，否则就是Normal了

```c#
public class Fixed1 : UIBase
{
    public override void Load()
    {
        // 这里一定要加上
        type = UIType.Fixed;
        Debug.Log("加载 Fixed1");
    }

    public override void UnLoad()
    {
        Debug.Log("卸载 Fixed1");
    }

    public override void Show()
    {
        Debug.Log("显示 Fixed1");
    }

    public override void Close()
    {
        Debug.Log("关闭 Fixed1");
    }

    public override void Freeze()
    {
        Debug.Log("冻结 Fixed1");
    }

    public override void UnFreeze()
    {
        Debug.Log("解冻 Fixed1");
    }
}
```



![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/24.png)

记得注册，然后重新Generate

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/25.png)

然后编写一个测试脚本

```c#
public class test : MonoBehaviour
{
    private UIManager uiManager;
    
    void Start()
    {
        uiManager = SGFEntry.Instance.GetManager<UIManager>();
        uiManager.Open(UIs.Fixed1);
    }
}
```

然后写一个类，把所有的Button方法都放到这，并拖到相关联的Button上

```c#
public class buttons : MonoBehaviour
{
    
    private UIManager uiManager;
    void Start()
    {
        uiManager = SGFEntry.Instance.GetManager<UIManager>();
    }
    
    public void OpenFixed2()
    {
        uiManager.Open(UIs.Fixed2);
    }

    public void OpenNormal1()
    {
        uiManager.Open(UIs.Normal1);
    }

    public void OpenNormal2()
    {
        uiManager.Open(UIs.Normal2);
    }

    public void OpenPop1()
    {
        uiManager.Open(UIs.Pop1);
    }

    public void OpenPop2()
    {
        uiManager.Open(UIs.Pop2);
    }

    public void Close()
    {
        uiManager.CloseCurrent();
    }
}
```

运行测试一下吧，我就不放截图了，太多了~
