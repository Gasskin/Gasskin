---
title: "GameFramework|10 流程Procedure"
categories: GameFramework 
tags: 框架
---

所谓流程，本质是对于状态机的再封装。一个游戏在运行是，不可能全程处于一个状态，游戏运行的不同状态，我称其为流程，比如刚打开游戏的时候，可能处在登录页面，或者说黑屏动画的时候在加载资源，这就是游戏运行的两个不同状态，这就是流程。

![1](https://www.logarius996.icu/images/SimpleGameFramework/Procedure/1.png)

### ProcedureBase

流程基类，本质就是一个状态

```c#
/// <summary>
/// 流程基类
/// </summary>
public class ProcedureBase : FsmState<ProcedureManager>
{
 
    public override void OnEnter(Fsm<ProcedureManager> fsm)
    {
        base.OnEnter(fsm);
        Debug.Log("进入流程：" + GetType().FullName);
    }
 
    public override void OnLeave(Fsm<ProcedureManager> fsm, bool isShutdown)
    {
        base.OnLeave(fsm, isShutdown);
        Debug.Log("离开流程：" + GetType().FullName);
    }
 
}
```

### ProcedureManager

```c#
/// <summary>
/// 流程管理器
/// </summary>
public class ProcedureManager : ManagerBase
{
    /// <summary>
    /// 状态机管理器
    /// </summary>
    private FsmManager m_FsmManager;
 
    /// <summary>
    /// 流程的状态机
    /// </summary>
    private Fsm<ProcedureManager> m_ProcedureFsm;
 
    /// <summary>
    /// 所有流程的列表
    /// </summary>
    private List<ProcedureBase> m_procedures;
 
    /// <summary>
    /// 入口流程
    /// </summary>
    private ProcedureBase m_EntranceProcedure;
 
    /// <summary>
    /// 当前流程
    /// </summary>
    public ProcedureBase CurrentProcedure
    {
        get
        {
            if (m_ProcedureFsm == null)
            {
                Debug.LogError("流程状态机为空，无法获取当前流程");
            }
            return (ProcedureBase)m_ProcedureFsm.CurrentState;
        }
    }
 
    public override int Priority
    {
        get
        {
            return ManagerPriority.ProcedureManager.GetHashCode();
        }
    }
 
    public ProcedureManager()
    {
        m_FsmManager = FrameworkEntry.Instance.GetManager<FsmManager>();
        m_ProcedureFsm = null;
        m_procedures = new List<ProcedureBase>();
    }
 
    public override void Init()
    {
        
    }
 
    public override void Shutdown()
    {
        
    }
 
    public override void Update(float time)
    {
        
    }
}
```

流程相关

```c#
public class ProcedureManager : ManagerBase
{
   		/// <summary>
        /// 添加流程
        /// </summary>
        public void AddProcedure(ProcedureBase procedure)
        {
            if (procedure == null)
            {
                Debug.LogError("要添加的流程为空");
                return;
            }
            m_procedures.Add(procedure);
        }
 
        /// <summary>
        /// 设置入口流程
        /// </summary>
        /// <param name="procedure"></param>
        public void SetEntranceProcedure(ProcedureBase procedure)
        {
            m_EntranceProcedure = procedure;
        }
        
        /// <summary>
        /// 创建流程状态机
        /// </summary>
        public void CreateProceduresFsm()
        {
            m_ProcedureFsm = m_FsmManager.CreateFsm(this, "", m_procedures.ToArray());
 
            if (m_EntranceProcedure == null)
            {
                Debug.LogError("入口流程为空，无法开始流程");
                return;
            }
 
            //开始流程
            m_ProcedureFsm.Start(m_EntranceProcedure.GetType());
        }
}
```

到此为止流程管理器就算是完成了，让我们来测试一下吧~

```c#
// 用这个流程模拟资源加载，3秒后正式进入游戏流程
public class Procedure_Loading : ProcedureBase
{
    private float time = 0;
    
    public override void OnUpdate(Fsm<ProcedureManager> fsm, float time)
    {
        this.time += time;
        if (this.time > 3)
        {
            ChangeState<Procedure_Gaming>(fsm);
        }
    }
}

// 这个是游戏中，当点击鼠标左键时，游戏切换到暂停状态
public class Procedure_Gaming : ProcedureBase
{
    public override void OnUpdate(Fsm<ProcedureManager> fsm, float time)
    {
       	Debug.Log("游戏中...");
        if (Input.GetMouseButtonDown(0))
        {
            ChangeState<Procedure_Pause>(fsm);
        }
    }
}

// 这个是暂停中，当点击鼠标左键时，继续运行游戏
public class Procedure_Pause : ProcedureBase
{
    public override void OnUpdate(Fsm<ProcedureManager> fsm, float time)
    {
        Debug.Log("暂停中...");
        if (Input.GetMouseButtonDown(0)) 
        {
            ChangeState<Procedure_Gaming>(fsm);
        }
    }
}
```

```c#
public class test : MonoBehaviour
{
    void Start()
    {
        var procedureManager = SGFEntry.Instance.GetManager<ProcedureManager>();
        
        Procedure_Loading loading = new Procedure_Loading();
        procedureManager.AddProcedure(loading);
        procedureManager.SetEntranceProcedure(loading);

        procedureManager.AddProcedure(new Procedure_Gaming());
        procedureManager.AddProcedure(new Procedure_Pause());
        
        procedureManager.CreateProceduresFsm();
    }
}
```

![1](https://www.logarius996.icu/images/SimpleGameFramework/Procedure/2.png)