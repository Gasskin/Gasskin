---
title: "GameFramework|08 事件池"
categories: GameFramework 
tags: 框架
---

上一节我们写了一个简单的引用池，而单一的引用池是没有啥作用的，事件池就是引用池的作用之一

我们当然可以把事件定义在模块的内部，比如把一个打开UI的事件定义在UI模块内，但是随着我们模块的不断增多，如果所有事件都定义在自己的模块内，在后期是很难维护的，我们都不知道一共定义了哪些事件...因此，一个全局的事件管理中心，是很有必要的

### 事件池

事件池是实际上保存事件的地方，可以添加或者取消订阅事件。

在此之前，我希望我们的委托遵循.NET规范，即应该是如下格式

```c#
 public delegate void TestEventHandler(Object sender, EventArgs e);
```

其中sender是事件的发送者，而e则是发送的数据

发送者自然是之后会写到的事件管理器，而这里的数据类型我们需要自定义，提供一个统一的基类

```c#
/// 所有事件的基类，会被引用池管理
public abstract class GlobalEventArgs:IReference
{
        public abstract int Id { get; set; }
        public abstract void Clear();
}
```

其中ID代表事件类型，不同类型的事件需要有不同的ID。Clear()方法属于IReference，我们会发现这个数据类型继承自引用接口，这说明什么？说明事件池传递的参数，是被引用池管理的！这就是引用池的作用之一，如果没有引用池，那每一次事件的调用我们都需要主动的new一个类型。

接着写好基本的事件池框架

```c#
public class EventPool<T> where T : GlobalEventArgs
{
#region Private
        /// 事件编码与对应的处理方法，这里的key，就是我们注册在事件数据里的ID，EventHandler<T>是一个(Object,T)类型的委托
        private Dictionary<int, EventHandler<T>> m_EventHandlers;
#endregion

#region 构造方法
        public EventPool()
        {
            m_EventHandlers = new Dictionary<int, EventHandler<T>>();
        }
#endregion
}
```

接口方法，添加订阅与取消订阅

```c#
public class EventPool<T> where T : GlobalEventArgs
{
#region Public 接口方法
        /// 订阅事件
        public void Subscribe(int id, EventHandler<T> handler)
        {
            if (handler == null)
            {
                throw new Exception("事件处理方法为空，无法订阅...");
            }
 
            EventHandler<T> eventHandler = null;
            
            // 检查是否获取处理方法失败或获取到的为空
            if (!m_EventHandlers.TryGetValue(id, out eventHandler) || eventHandler == null)
            {
                m_EventHandlers[id] = handler;
            }
            // 不为空，就检查处理方法是否重复了
            else if (Check(id, handler))
            {
                Debug.LogError("ID为:" + (SGFEvents)id + "的事件下已存在这个处理方法:" + nameof(handler) + "...");
            }
            else
            {
                eventHandler += handler;
                m_EventHandlers[id] = eventHandler;
            }
        }
 
        /// 取消订阅事件
        public void Unsubscribe(int id, EventHandler<T> handler)
        {
            if (handler == null)
            {
                throw new Exception("事件处理方法为空，无法取消订阅...");
            }
 
            if (m_EventHandlers.ContainsKey(id))
            {
                m_EventHandlers[id] -= handler;
            }
        }

        
        /// 抛出事件（线程不安全），抛出之后会立刻执行
        public void FireNow(object sender, T e)
        {
            HandleEvent(sender, e);
        }
#endregion
    
#region Private 工具方法 
        /// 检查某个编码的事件是否存在它对应的处理方法
        private bool Check(int id, EventHandler<T> handler)
        {
            if (handler == null)
            {
                throw new Exception("事件处理方法为空...");
            }
 
            EventHandler<T> handlers = null;
            if (!m_EventHandlers.TryGetValue(id, out handlers))
            {
                return false;
            }
 
            if (handlers == null)
            {
                return false;
            }
 
            // 遍历委托里的所有方法
            foreach (EventHandler<T> i in handlers.GetInvocationList())
            {
                if (i == handler)
                {
                    return true;
                }
            }
 
            return false;
        }
        
        /// 事件处理
        private void HandleEvent(object sender, T e)
        {
            // 尝试获取事件的处理方法
            int eventId = e.Id;
            EventHandler<T> handlers = null;
            if (m_EventHandlers.TryGetValue(eventId, out handlers))
            {
                if (handlers != null)
                {
                    handlers(sender, e);
                }
                else
                {
                    throw new Exception("事件没有对应处理方法：" + eventId);
                }
            }
        }
#endregion
}
```

到此为止大体上的内容就已经完成了，我们来写我们的事件管理器

### 事件管理器

其实事件管理器就是对事件池的一个代理罢了

```c#
public class EventManager : ManagerBase
{
#region Private
        /// 事件池，维护一个事件池，其实事件管理器就是对事件池的代理
        private EventPool<GlobalEventArgs> m_EventPool;
#endregion

#region 构造方法
        public EventManager()
        {
            m_EventPool = new EventPool<GlobalEventArgs>();
        }
#endregion
        
#region Override
        public override int Priority
        {
            get { return ManagerPriority.EventManager.GetHashCode(); }
        }
#endregion

#region Public 接口方法
        /// 订阅事件
        public void Subscribe(SGFEvents id, EventHandler<GlobalEventArgs> handler)
        {
            // =========== 这一部分是用于DEBUG的，会在Hierachy中显示出当前的引用状态 ========== 
            #if UNITY_EDITOR
            var trans = SGFEntry.Instance.transform.Find("EventManager");
            var name = id.ToString();
            var temp = trans.Find(name);
            if (temp == null)   
            {
                GameObject go = new GameObject();
                go.name = name;
                go.transform.SetParent(trans);
                
                GameObject go2 = new GameObject();
                go2.name = $"{handler.Target}：{handler.Method.Name}";
                go2.transform.SetParent(go.transform);
            }
            else
            {
                GameObject go2 = new GameObject();
                go2.name = $"{handler.Target}：{handler.Method.Name}";
                go2.transform.SetParent(temp.transform);
            }
            #endif
            // =========================================================================
            
            m_EventPool.Subscribe(id.GetHashCode(), handler);
        }
 
        /// 取消订阅事件
        public void Unsubscribe(SGFEvents id, EventHandler<GlobalEventArgs> handler)
        {
            m_EventPool.Unsubscribe(id.GetHashCode(), handler);
            
            // =========== 这一部分是用于DEBUG的，会在Hierachy中显示出当前的引用状态 ==========
			#if UNITY_EDITOR
            var trans = SGFEntry.Instance.transform.Find("EventManager");
            var group = trans.Find(id.ToString());
            var child = group.Find($"{handler.Target}：{handler.Method.Name}");
            if (child != null) 
            {
                GameObject.DestroyImmediate(child.gameObject);
            }
            if (group.childCount <= 0) 
            {
                GameObject.DestroyImmediate(group.gameObject);
            }
			#endif
            // =========================================================================
        }
        
        /// 抛出事件（线程不安全）
        public void FireNow(object sender, GlobalEventArgs e)
        {
            m_EventPool.FireNow(sender, e);
        }
#endregion
}
```

完全都是在调用事件池中的方法，单纯只是一个代理类而已

好了让我们先来测试一下，以打开UI为例，首先我们需要有一个打开UI的数据类型，且必须继承自GolbalEventArgs

![1](https://www.logarius996.icu/images/SimpleGameFramework/Event/1.png)

```c#
public class UIOpenEventArgs : GlobalEventArgs
{
    /// UI的名字，所有的UI都会注册这个事件，则需要靠名称来确定当前到底打开了哪一个UI
    public string uiName;
    
    public string data;

    /// 这个ID是事件的ID，会根据事件ID找到相应的处理方法 
    public override int Id
    {
        get
        {
            return SGFEvents.OpenUI.GetHashCode();
        }
        set { }
    }

    /// 归还引用后清空数据
    public override void Clear()
    {
        Id = UIRegister.None.GetHashCode();
        uiName = string.Empty;
        data = string.Empty;
    }
}
```

然后让我们在UI中注册这个事件，打开UI这个事件应该是所有UI都需要的，所以我们直接在UIBase中注册

在此之前，为了事件方便管理，我们把所有事件的ID注册到一个类中

```c#
/// 注册所有事件的事件编码
public enum SGFEvents
{
        /*================== UI相关 =================*/
        OpenUI,
}
```

更新UIBase

```c#
public abstract class UIBase : MonoBehaviour
{
#region Field
        /// 事件管理器
        private EventManager eventManager;
        /// UI的名字，不可重复
        public string uiName;
#endregion
    
#region MonoBehaviour
        private void Awake()
        {
            eventManager = SGFEntry.Instance.GetManager<EventManager>();
            eventManager.Subscribe(SGFEvents.OpenUI,AfterShow);
            Load();
        }

        private void OnDestroy()
        {
            eventManager.Unsubscribe(SGFEvents.OpenUI,AfterShow);
            UnLoad();
        }
#endregion


#region 工具方法
        private void AfterShow(object o, GlobalEventArgs e)
        {
            var temp = e as UIOpenEventArgs;
            if (!temp.uiName.Equals(uiName))
            {
                return;
            }
            DoAfterShow(o,temp);
        }
#endregion
    
#region 事件
        /// <summary>
        /// 会在这个UI显示后调用 
        /// </summary>
        public virtual void DoAfterShow(object o, UIOpenEventArgs e)
        {
        
        }
#endregion
}
```

更新UIManager

```c#
public class UIManager : ManagerBase
{
#region Field
        /// 事件管理器
        private EventManager eventManager;
        /// 引用池
        private ReferenceManager referenceManager;
#endregion

#region 管理器生命周期
        public override void Init()
        {
            //...
        
            // 初始化管理器
            eventManager = SGFEntry.Instance.GetManager<EventManager>();
            referenceManager = SGFEntry.Instance.GetManager<ReferenceManager>();
        }
#endregion

 #region 接口方法
        /// <summary>
        /// 打开一个UI
        /// </summary>
        public void Open(UIStruct data, UIOpenEventArgs uiOpenEventArgs)
        {
            // 如果这个UI还没被加载，那需要先加载
            if (!uiLoaded.TryGetValue(data.name,out var ui)) 
            {
                LoadUI(data,out ui);
            }
            // 显示UI
            ShowUI(ui);
            // 发送打开UI事件
            uiOpenEventArgs.uiName = ui.uiName;
            eventManager.FireNow(this,uiOpenEventArgs);
        }
#endregion
}
```

以Fixed1这个UI为例

```c#
public class Fixed1 : UIBase
{
    public override void DoAfterShow(object o, UIOpenEventArgs e)
    {
        Debug.Log(e.data);
    }
}

```

```c#
public class test : MonoBehaviour
{
    private ReferenceManager referenceManager;
    void Start()
    {
        referenceManager = SGFEntry.Instance.GetManager<ReferenceManager>();
        var tempRef = referenceManager.Acquire<UIOpenEventArgs>();
        tempRef.data = "测试：打开UI的事件！";
        SGFEntry.Instance.GetManager<UIManager>().Open(UIs.Fixed1, tempRef);
    }
}
```

![1](https://www.logarius996.icu/images/SimpleGameFramework/Event/2.png)

大体上算是完成了，但会有一些细节可能会影响大家的需求，在我这里，我把所有的打开UI归类为一个委托，不关我打开A还是打开B还是打开C，所有注册了这个事件的页面，都会被执行，所以如果大家只想当前最新打开的页面执行委托，那么在重写DoAfterShow方法时，需要进行额外的判断，根据传入的数据来判断是不是当前页面。







































