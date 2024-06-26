---
title: "GameFramework|06 UI管理器04"
categories: GameFramework 
tags: 框架
---

### 工具类

上一节的末尾，我们需要创建很多UI来测试，因为需要很多Button事件，所以我们写了很多函数，然后...然后一个一个拖到了Button上...很明显的能够感到，这个过程是真的很痛苦...这才五六个Button而已，而实际上我们可能需要几十个几百个Button事件，如果我们都采用这样直接拖放的方式...那是真的痛苦...

在UI下新建一个静态工具类，就叫UIUtility吧~

```c#
/// <summary>
/// UI工具类
/// </summary>
public static class UIUtility
{
    /// <summary>
    /// 根据路径获取某个child的某个组件 
    /// </summary>
    private static T Find<T>(Transform trans, string path)
    {
        var child = trans.Find(path);
        return child.GetComponent<T>();
    }


    /// <summary>
    /// 根据路径给按钮添加一个OnClick方法
    /// </summary>
    public static void RegisterButton(this Transform trans, string path, UnityAction action)
    {
        var btn = Find<Button>(trans, path);
        btn.onClick.AddListener(action);
    }

    /// <summary>
    /// 根据路径获取一个Text组件
    /// </summary>
    public static Text FindText(this Transform trans, string path)
    {
        return Find<Text>(trans, path);
    }
}
```

可以根据需要自己添加其他方法，比如查找一个Image组件或者另外啥东西

顺带一提，其实这些方法可以写在**UIBase**里，那样的话用起来会更加方便，但是我还是觉得这部分只是作为工具方法，哪怕没有这部分方法我们的UI也是能够运行的，作为扩充，我就没有放到**UIBase**里，而是直接写了一个工具类，大家可以根据自己的想法自行处理~

然后我们新建两个UI

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/26.png)

注意，这两个UI不要相互遮挡，其次必须是Pop类型，等下就明白为啥了~

然后在Config中注册，并Genertated

然后编写咱们的UI脚本，此时就可以用上咱们写的工具方法了~

```c#
public class Update1 : UIBase
{
    private UIManager manager;

    private Text text;
    
    public override void Load()
    {
        /// 一定要把UI类型设置为PopUp
        UIType = UIType.PopUp;

        manager = SGFEntry.Instance.GetManager<UIManager>();

        text = transform.FindText("Image/Text");
        
        transform.RegisterButton("Image/Button",(() =>
        {
            manager.Open(UIs.Update2);
        }));
    }

    public override void UnLoad()
    {
        
    }
}
```

```c#
public class Update2 : UIBase
{
    private UIManager manager;

    private Text text;
    
    public override void Load()
    {
        UIType = UIType.PopUp;

        manager = SGFEntry.Instance.GetManager<UIManager>();

        text = transform.FindText("Image/Text");
        
        /// 这里是关闭
        transform.RegisterButton("Image/Button",(() =>
        {
            manager.CloseCurrent();
        }));
    }

    public override void UnLoad()
    {
        
    }
}
```

然后在Test脚本中，默认打开Update1

```c#
public class test : MonoBehaviour
{
    private UIManager uiManager;
    void Start()
    {
        uiManager = SGFEntry.Instance.GetManager<UIManager>();
        uiManager.Open(UIs.Update1);
    }
}
```

运行一下，发现Button是有效的，现在可不用一个一个拖放点击事件了，方便了非常，非常，非常多！！！

### OnUpdate更新

写到这，其实还有一个生命周期我们是没有使用的，那就是OnUpdate！

现在我们给两个UI补上这个

```c#
public class Update1 : UIBase
{
    public override void OnUpdate(float deltaTime)
    {
        text.text = deltaTime.ToString();
    }
}

public class Update2 : UIBase
{
    public override void OnUpdate(float deltaTime)
    {
        text.text = deltaTime.ToString();
    }
}
```

还需要在UIManager中添加这个功能

```c#
public class UIManager : ManagerBase
{
    #region 管理器生命周期

    public override void Update(float time)
    {
        currentUI.OnUpdate(time);
    }
    
    #endregion
}

```

可以注意到，我们只会在每一帧更新当前UI，其他UI是不会被更新的

再次运行脚本，你会发现此时，只有Update2会更新

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/27.png)

### 更多功能

目前为止，大体上的UI管理算是完成了，但是仍然有很多地方需要扩展，比如典型的，你会发现现在打开UI的时候我们是没法传递参数的

比较简单的做法是在Show()方法里加一个Object类型的参数，这样就可以传递任意数据，本来是打算写这一部分逻辑的，但我想了想，其实没有必要，数据的传递和UI本身已经有点脱离了，我不想把数据发送也写在UI模块里，我们应该有通用的事件管理模块

另外，现在的UI管理器和UI界面本身，不带有任何消息，比如我们Open()一个UI，但实际上我们不知道具体什么时候在加载UI，什么时候在显示UI，因为现在没有任何事件或者说回调写在我们的管理器中，这一部分也是需要考虑的

好了！暂时放下UI这部分，让我们开启一个新的模块，事件管理器~





























