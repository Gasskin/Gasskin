---
title: "GameFramework|09 Fsm有限状态机"
categories: GameFramework 
tags: 框架
---

状态机很好理解，它包含了很多的状态，最简单的例子就是Unity自带的Animator Controller，他本身就是一个状态机，里面有很多的动画状态，不同的状态代表不同的动画，然后我们给不同状态之间加上条件，描述状态的转移。

这个模块本身是很好理解的，但目前我也不是很清楚具体会应用在什么地方，参照UGF的描述，一些用于状态切换的游戏逻辑，是可以写在这个模块的。当然，我们后面还有一个流程管理模块Procedure，这个模块就是基于状态机的再封装，也算是一个应用。

咱们新建如下脚本

![1](https://www.logarius996.icu/images/SimpleGameFramework/Fsm/1.png)

其中

IFsm是状态机接口

Fsm是状态机

FsmState是状态，状态机管理很多状态

FsmManager是状态机管理类，我们可以有很多状态机

### Fsm

定义状态机接口

```c#
/// <summary>
/// 状态机接口
/// </summary>
public interface IFsm
{
    /// <summary>
    /// 状态机名字
    /// </summary>
    string Name { get; }
 
    /// <summary>
    /// 状态机持有者类型
    /// </summary>
    Type OwnerType { get; }
 
    /// <summary>
    /// 状态机是否被销毁
    /// </summary>
    bool IsDestroyed { get; }
 
    /// <summary>
    /// 当前状态运行时间
    /// </summary>
    float CurrentStateTime { get; }
 
    /// <summary>
    /// 状态机轮询
    /// </summary>
    void Update(float time);
 
    /// <summary>
    /// 关闭并清理状态机。
    /// </summary>
    void Shutdown();
 
}
```

定义状态机

```c#
/// <summary>
/// 状态机
/// </summary>
/// <typeparam name="T">状态机持有者类型</typeparam>
public class Fsm<T> : IFsm where T : class
{   
        /// <summary>
    /// 状态机名字
    /// </summary>
    public string Name { get; private set; }
 
    /// <summary>
    /// 获取状态机持有者类型
    /// </summary>
    public Type OwnerType
    {
        get
        {
            return typeof(T);
        }
    }
 
    /// <summary>
    /// 状态机是否被销毁
    /// </summary>
    public bool IsDestroyed { get; private set; }
 
    /// <summary>
    /// 当前状态运行时间
    /// </summary>
    public float CurrentStateTime { get; private set; }
 
    /// <summary>
    /// 状态机里所有状态的字典
    /// </summary>
    private Dictionary<string, FsmState<T>> m_States;
 
    /// <summary>
    /// 状态机里所有数据的字典
    /// </summary>
    private Dictionary<string, object> m_Datas;
 
    /// <summary>
    /// 当前状态
    /// </summary>
    public FsmState<T> CurrentState { get; private set; }
 
    /// <summary>
    /// 状态机持有者
    /// </summary>
    public T Owner { get; private set; }
    
    public Fsm(string name, T owner, params FsmState<T>[] states)
    {
        if (owner == null)
        {
            Debug.LogError("状态机持有者为空");
        }
 
        if (states == null || states.Length < 1)
        {
            Debug.LogError("没有要添加进状态机的状态");
        }
 
        Name = name;
        Owner = owner;
        m_States = new Dictionary<string, FsmState<T>>();
        m_Datas = new Dictionary<string, object>();
 
        foreach (FsmState<T> state in states)
        {
            if (state == null)
            {
                Debug.LogError("要添加进状态机的状态为空");
            }
 
            string stateName = state.GetType().FullName;
            if (m_States.ContainsKey(stateName))
            {
                Debug.LogError("要添加进状态机的状态已存在：" + stateName);
            }
 
            m_States.Add(stateName, state);
            state.OnInit(this);
        }
 
        CurrentStateTime = 0f;
        CurrentState = null;
        IsDestroyed = false;
 
    }
 
    /// <summary>
    /// 关闭并清理状态机。
    /// </summary>
    public void Shutdown()
    {
        if (CurrentState != null)
        {
            CurrentState.OnLeave(this, true);
            CurrentState = null;
            CurrentStateTime = 0f;
        }
 
        foreach (KeyValuePair<string, FsmState<T>> state in m_States)
        {
            state.Value.OnDestroy(this);
        }
 
        m_States.Clear();
        m_Datas.Clear();
 
        IsDestroyed = true;
    }
 
    /// <summary>
    /// 状态机轮询
    /// </summary>
    public void Update(float elapseSeconds, float realElapseSeconds)
    {
        if (CurrentState == null)
        {
            return;
        }
 
        CurrentStateTime += elapseSeconds;
        CurrentState.OnUpdate(this, elapseSeconds, realElapseSeconds);
    }
 
}
```

获取状态

```c#
public class Fsm<T> : IFsm where T : class
{   
	/// <summary>
    /// 获取状态
    /// </summary>
    public TState GetState<TState>() where TState : FsmState<T>
    {
        return GetState(typeof(TState)) as TState;
    }
    
    /// <summary>
    /// 获取状态
    /// </summary>
    public FsmState<T> GetState(Type stateType)
    {
        if (stateType == null)
        {
            Debug.LogError("要获取的状态为空");
        }
 
        if (!typeof(FsmState<T>).IsAssignableFrom(stateType))
        {
            Debug.LogError("要获取的状态" + stateType.FullName + "没有直接或间接的实现" + typeof(FsmState<T>).FullName);
        }
 
        FsmState<T> state = null;
        if (m_States.TryGetValue(stateType.FullName, out state))
        {
            return state;
        }
 
        return null;
    }
}
```

切换状态

```c#
public class Fsm<T> : IFsm where T : class
{   
	/// <summary>
    /// 切换状态
    /// </summary>
    public void ChangeState<TState>() where TState : FsmState<T>
    {
        ChangeState(typeof(TState));
    }
 
    /// <summary>
    /// 切换状态
    /// </summary>
    public void ChangeState(Type type)
    {
        if (CurrentState == null)
        {
            Debug.LogError("当前状态机状态为空，无法切换状态");
        }
 
        FsmState<T> state = GetState(type);
        if (state == null)
        {
            Debug.Log("获取到的状态为空，无法切换：" + type.FullName);
        }
        CurrentState.OnLeave(this, false);
        CurrentStateTime = 0f;
        CurrentState = state;
        CurrentState.OnEnter(this);
    }
}
```

开始状态机

```c#
public class Fsm<T> : IFsm where T : class
{   
	/// <summary>
    /// 开始状态机
    /// </summary>
    /// <typeparam name="TState">开始的状态类型</typeparam>
    public void Start<TState>() where TState : FsmState<T>
    {
        Start(typeof(TState));
    }
    /// <summary>
    /// 开始状态机
    /// </summary>
    /// <param name="stateType">要开始的状态类型。</param>
    public void Start(Type stateType)
    {
        if (CurrentState != null)
        {
            Debug.LogError("当前状态机已开始，无法再次开始");
        }
 
        if (stateType == null)
        {
            Debug.LogError("要开始的状态为空，无法开始");
        }
 
        FsmState<T> state = GetState(stateType);
        if (state == null)
        {
            Debug.Log("获取到的状态为空，无法开始");
        }
 
        CurrentStateTime = 0f;
        CurrentState = state;
        CurrentState.OnEnter(this);
    }
}
```

抛出事件

```c#
public class Fsm<T> : IFsm where T : class
{   
	/// <summary>
    /// 抛出状态机事件
    /// </summary>
    /// <param name="sender">事件源</param>
    /// <param name="eventId">事件编号</param>
    public void FireEvent(object sender, int eventId)
    {
        if (CurrentState == null)
        {
            Debug.Log("当前状态为空，无法抛出事件");
        }
        CurrentState.OnEvent(this, sender, eventId, null);
    }
}
```

以及状态机的全局数据（其实这部分可以额外封装成数据类

```c#
public class Fsm<T> : IFsm where T : class
{   
	/// <summary>
    /// 是否存在状态机数据
    /// </summary>
    public bool HasData(string name)
    {
        if (string.IsNullOrEmpty(name))
        {
            Debug.Log("要查询的状态机数据名字为空");
        }
 
        return m_Datas.ContainsKey(name);
    }
 
    /// <summary>
    /// 获取状态机数据
    /// </summary>
    public TDate GetData<TDate>(string name)
    {
        return (TDate)GetData(name);
    }
 
    /// <summary>
    /// 获取状态机数据
    /// </summary>
    public object GetData(string name)
    {
        if (string.IsNullOrEmpty(name))
        {
            Debug.Log("要获取的状态机数据名字为空");
        }
 
        object data = null;
        m_Datas.TryGetValue(name, out data);
        return data;
    }
 
    /// <summary>
    /// 设置状态机数据
    /// </summary>
    public void SetData(string name, object data)
    {
        if (string.IsNullOrEmpty(name))
        {
            Debug.Log("要设置的状态机数据名字为空");
        }
 
        m_Datas[name] = data;
    }
 
    /// <summary>
    /// 移除状态机数据
    /// </summary>
    public bool RemoveData(string name)
    {
        if (string.IsNullOrEmpty(name))
        {
            Debug.Log("要移除的状态机数据名字为空");
        }
 
        return m_Datas.Remove(name);
    }
}
```

### FsmState

```c#
/// <summary>
/// 状态基类
/// </summary>
/// <typeparam name="T">状态持有者类型</typeparam>
public class FsmState<T> where T : class
{
 
}
```

参考UGF，状态机的事件系统，不经过事件管理器，而是自己维护一套

```c#
public class FsmState<T> where T : class
{
 	/// <summary>
	/// 状态机事件的响应方法模板
	/// </summary>
	public delegate void FsmEventHandler<T>(Fsm<T> fsm, object sender, object userData) where T : class;
    
    /// <summary>
    /// 状态订阅的事件字典
    /// </summary>
    private Dictionary<int, FsmEventHandler<T>> m_EventHandlers;
 
    public FsmState()
    {
        m_EventHandlers = new Dictionary<int, FsmEventHandler<T>>();
    }
}
```

订阅事件相关的方法

```c#
public class FsmState<T> where T : class
{
    /// <summary>
    /// 订阅状态机事件。
    /// </summary>
    protected void SubscribeEvent(int eventId, FsmEventHandler<T> eventHandler)
    {
        if (eventHandler == null)
        {
            Debug.LogError("状态机事件响应方法为空，无法订阅状态机事件");
        }
 
        if (!m_EventHandlers.ContainsKey(eventId))
        {
            m_EventHandlers[eventId] = eventHandler;
        }
        else
        {
            m_EventHandlers[eventId] += eventHandler;
        }
    }
 
    /// <summary>
    /// 取消订阅状态机事件。
    /// </summary>
    protected void UnsubscribeEvent(int eventId, FsmEventHandler<T> eventHandler)
    {
        if (eventHandler == null)
        {
            Debug.LogError("状态机事件响应方法为空，无法取消订阅状态机事件");
        }
 
        if (m_EventHandlers.ContainsKey(eventId))
        {
            m_EventHandlers[eventId] -= eventHandler;
        }
    }
 
    /// <summary>
    /// 响应状态机事件。
    /// </summary>
    public void OnEvent(Fsm<T> fsm, object sender, int eventId, object userData)
    {
        FsmEventHandler<T> eventHandlers = null;
        if (m_EventHandlers.TryGetValue(eventId, out eventHandlers))
        {
            if (eventHandlers != null)
            {
                eventHandlers(fsm, sender, userData);
            }
        }
    }
}
```

生命周期

```c#
public class FsmState<T> where T : class
{
 	/// <summary>
    /// 状态机状态初始化时调用
    /// </summary>
    /// <param name="fsm">状态机引用</param>
    public virtual void OnInit(Fsm<T> fsm)
    {
 
    }
 
    /// <summary>
    /// 状态机状态进入时调用
    /// </summary>
    /// <param name="fsm">状态机引用</param>
    public virtual void OnEnter(Fsm<T> fsm)
    {
 
    }
 
    /// <summary>
    /// 状态机状态轮询时调用
    /// </summary>
    /// <param name="fsm">状态机引用</param>
    public virtual void OnUpdate(Fsm<T> fsm, float elapseSeconds, float realElapseSeconds)
    {
 
    }
 
    /// <summary>
    /// 状态机状态离开时调用。
    /// </summary>
    /// <param name="fsm">状态机引用。</param>
    /// <param name="isShutdown">是关闭状态机时触发</param>
    public virtual void OnLeave(Fsm<T> fsm, bool isShutdown)
    {
 
    }
 
    /// <summary>
    /// 状态机状态销毁时调用
    /// </summary>
    /// <param name="fsm">状态机引用。</param>
    public virtual void OnDestroy(Fsm<T> fsm)
    {
        m_EventHandlers.Clear();
    }
}
```

切换状态

```c#
public class FsmState<T> where T : class
{
 	/// <summary>
    /// 切换状态
    /// </summary>
    protected void ChangeState<TState>(Fsm<T> fsm) where TState : FsmState<T>
    {
        ChangeState(fsm, typeof(TState));
    }
 
    /// <summary>
    /// 切换状态
    /// </summary>
    protected void ChangeState(Fsm<T> fsm, Type type)
    {
        if (fsm == null)
        {
            Debug.Log("需要切换状态的状态机为空，无法切换");
        }
 
        if (type == null)
        {
            Debug.Log("需要切换到的状态为空，无法切换");
        }
 
        if (!typeof(FsmState<T>).IsAssignableFrom(type))
        {
            Debug.Log("要切换的状态没有直接或间接实现FsmState<T>，无法切换");
        }
 
        fsm.ChangeState(type);
    }
}
```

### FsmManager

```c#
public class FsmManager : ManagerBase
{
    /// <summary>
    /// 所有状态机的字典（在这里，状态机接口的作用就显示出来了）
    /// </summary>
    private Dictionary<string, IFsm> m_Fsms;
    private List<IFsm> m_TempFsms;
 
 
    public override int Priority
    {
        get
        {
            return 60;
        }
    }
 
 
    public override void Init()
    {
        m_Fsms = new Dictionary<string, IFsm>();
        m_TempFsms = new List<IFsm>();
    }
 
 
    /// <summary>
    /// 关闭并清理状态机管理器
    /// </summary>
    public override void Shutdown()
    {
        foreach (KeyValuePair<string, IFsm> fsm in m_Fsms)
        {
            fsm.Value.Shutdown();
        }
 
 
        m_Fsms.Clear();
        m_TempFsms.Clear();
    }
 
 
    /// <summary>
    /// 轮询状态机管理器
    /// </summary>
    public override void Update(float elapseSeconds, float realElapseSeconds)
    {
        m_TempFsms.Clear();
        if (m_Fsms.Count <= 0)
        {
            return;
        }
 
        // 这里用一个额外链表的作用是，字典类型的数据结构，foreach时无法增删
        foreach (KeyValuePair<string, IFsm> fsm in m_Fsms)
        {
            m_TempFsms.Add(fsm.Value);
        }
 
        foreach (IFsm fsm in m_TempFsms)
        {
            if (fsm.IsDestroyed)
            {
                continue;
            }
            //轮询状态机
            fsm.Update(elapseSeconds, realElapseSeconds);
        }
    }
}
```

添加和销毁状态机

```c#
public class FsmManager : ManagerBase
{
    /// <summary>
    /// 是否存在状态机
    /// </summary>
    private bool HasFsm(string fullName)
    {
        return m_Fsms.ContainsKey(fullName);
    }
 
    /// <summary>
    /// 是否存在状态机
    /// </summary>
    public bool HasFsm<T>()
    {
        return HasFsm(typeof(T));
    }
 
    /// <summary>
    /// 是否存在状态机
    /// </summary>
    public bool HasFsm(Type type)
    {
        return HasFsm(type.FullName);
    }
    
    /// <summary>
    /// 创建状态机。
    /// </summary>
    /// <typeparam name="T">状态机持有者类型</typeparam>
    /// <param name="name">状态机名称</param>
    /// <param name="owner">状态机持有者</param>
    /// <param name="states">状态机状态集合</param>
    /// <returns>要创建的状态机</returns>
    public Fsm<T> CreateFsm<T>(T owner, string name = "", params FsmState<T>[] states) where T : class
    {
        if (HasFsm<T>())
        {
            Debug.LogError("要创建的状态机已存在");
        }
        if (name == "")
        {
            name = typeof(T).FullName;
        }
        Fsm<T> fsm = new Fsm<T>(name, owner, states);
        m_Fsms.Add(name, fsm);
        return fsm;
    }
    
    /// <summary>
    /// 销毁状态机
    /// </summary>
    public bool DestroyFsm(string name)
    {
        IFsm fsm = null;
        if (m_Fsms.TryGetValue(name, out fsm))
        {
            fsm.Shutdown();
            return m_Fsms.Remove(name);
        }
 
        return false;
    }
    public bool DestroyFsm<T>() where T : class
    {
        return DestroyFsm(typeof(T).FullName);
    }
    public bool DestroyFsm(IFsm fsm)
    {
        return DestroyFsm(fsm.Name);
    }
}
```

状态机模块算是完成了，其作用会在下一节Procedure中体现























