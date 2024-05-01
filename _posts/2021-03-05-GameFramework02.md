---
title: "GameFramework|02 一切从管理器出发"
categories: GameFramework 
tags: 框架
---
### 万事开头难！

想了半天，应该从哪儿开始呢，算了还是别开始了...

哈哈开玩笑的~让我们先新建一个Unity项目，并给咱们这个框架取一个响当当的名字：SimpleGameFramework，简称SGF（好吧我承认名字很随便，你随意~）

> 项目结构：
> 
>  SimpleGameFramework

好吧这只是一个毫无看点的空项目...那么我们该从哪儿开始学习呢？

在上一篇的结尾我曾经提出过一个想法，即：把系统分为很多不同的模块，每个模块只关心自己的事情。那么很自然的会有一个问题，如果我们的项目有很多模块，那我们要怎么统一管理这些模块呢？当然我们也可以不统一管理...随便管理就好，但后果就是当模块数量越来越多的时候，我们就开始难以控制他们，不知道什么模块在什么时候被什么调用...

可能大家还不太能体会到统一管理所有模块的好处，但没关系，就假设这是我们的一个需求好了！即，我们需要有一个地方能够管理我们所有的模块，那就叫这个类为SGFEntry好了，即框架的入口，所有模块都从这里开始产生效果。

那我再问个问题，咱们这个SGFEntry类，可以是普普通通的随便一个类吗？答案当然是否定的...试想一下，如果这个类很普通，你可以随便的new，那我们项目里就可以存在很多的入口，哪怕不考虑代码细节，光从逻辑上想这都是很搞笑的，入口当然有且只有一个！那该怎么实现呢？

### 从单例模式开始

单例模式是很好用的一个模式，他可以保证我们的类在整个项目的运行期间都是唯一的，因此不管我们从什么地方调用，进入的都是唯一的那个类，也就是唯一的入口！Perfect！完美符合我们的想法！

我们可以直接把SGFEntry这个类写成单例类，当然这就可以满足我们的要求，但在这里我提供另一个思路

我们先写一个单例模板类，继承这个模板的类会是一个单例类

```c#
namespace SimpleGameFramework.Core
{
    /// <summary>
    /// 脚本的单例模板基类
    /// </summary>
    public abstract class Singleton<T> : MonoBehaviour where T : Singleton<T>
    {
        /// 保存实际的对象
        protected static T _instance;
        /// 对外提供访问接口
        /// 即我们可以使用Singleton<T>.Instance来访问_instance，而当_instance不存在时，我们会对其初始化
        /// 注意，这不会在程序运行的时候初始化这个类，只有我们访问时，才可能进行初始化
        public static T Instance
        {
            get
            {
                // 如果不存在实例，尝试进行初始化
                if (_instance == null)
                {
                    //从场景中找T脚本的对象，这会找到所有T对象，并返回最后一个加入场景中的那个
                    _instance = FindObjectOfType<T>();

                    // 如果T对象的数量比1大，那很明显场景里出现了多个单例类，必然是有错误的
                    if (FindObjectsOfType<T>().Length > 1)
                    {
                        Debug.LogError("场景中的单例脚本数量 > 1:" + _instance.GetType().ToString());
                        return _instance;
                    }

                    // 场景中找不到的情况，那就要进行初始化了
                    if (_instance == null)
                    {	
                        
                        string instanceName = typeof(T).Name;
                        GameObject instanceGO = GameObject.Find(instanceName);
						// 我们会生成一个空的gameobject并挂在对应脚本，如果场景里有同名的物体，自然就会有问题 
                        if (instanceGO == null)
                        {
                            instanceGO = new GameObject(instanceName);
                            DontDestroyOnLoad(instanceGO);
                            _instance = instanceGO.AddComponent<T>();
                            // 删除这一行代码并运行游戏可以很明显的发现变化，他的作用是切换场景的时候不销毁指定对象
                            DontDestroyOnLoad(_instance);
                        }
                        else
                        {
                            //场景中已存在同名游戏物体时就打印提示
                            Debug.LogError("场景中已存在单例脚本所挂载的游戏物体:" + instanceGO.name);
                        }
                    }
                }

                return _instance;
            }
        }

        void OnDestroy()
        {
            _instance = null;
        }
    }
}
```

并新建一个SimpleGameFramework，再新建一个Core文件夹，把单例模板放入其中，在这个文件夹中，我们会存放我们的核心类

> SimpleGameFramework
> 
> - Core
> 
>   Singleton.cs

### 管理器

继续我们的思维探索，我们现在有很多模块，每一个模块都需要有自己的管理器，并且所有的模块会被SGFEntry统一管理，即总管理器，这种设计模式我愿称其为Manager Of Managers

那很明显，虽然管理器是不同的，但他们都是管理器，都需要被统一管理，那自然他们都需要继承自同一个基类，这应该是很好理解的吧，所以我们来写一个管理器的基类，定义管理器的一些核心属性和方法~

```c#
namespace SimpleGameFramework.Core
{
    /// <summary>
    /// 模块管理基类
    /// </summary>
    public abstract class ManagerBase
    {
        /// 模块优先级，优先级高的模块会被先SGFEntry处理
        public abstract int Priority { get; }

        /// 初始化模块
        public abstract void Init();

        /// 模块更新
        public abstract void Update(float time);

        /// 关闭模块
        public abstract void ShutDown();
    }
}
```

我先不多解是其中的具体含义，但提醒同学们一点，大家发现这个类竟然只是一个普通的C#类而没有继承Monobehaviour！这说明什么？这说明这个类是没法被Unity所Update的！那么我们要怎么更新我们的模块呢？嘿嘿，卖个关子，看完下面的内容大家肯定都能理解了~

好了我们现在写我们的框架总入口：SGFEntry！

首先，我们的框架入口需要继承我们的单例模板，表示我们的的入口是全局唯一的！

```c#
namespace SimpleGameFramework.Core
{
    /// 框架入口，管理所有的模块
    public class SGFEntry : Singleton<SGFEntry>
    {
        
    }
}
```

定义我们的管理器链表，这里维护了我们所有的管理器，并按照优先级排序

```c#
namespace SimpleGameFramework.Core
{
    public class SGFEntry : Singleton<SGFEntry>
    {
        /// 维护了所有的管理器，并按照管理器的优先级由大到小排序
		private LinkedList<ManagerBase> m_Managers = new LinkedList<ManagerBase>();
    }
}
```

定义获取和新建管理器的方法

```c#
using System;
using System.Collections.Generic;
using UnityEngine;

namespace SimpleGameFramework.Core
{
    public class SGFEntry : Singleton<SGFEntry>
    {
        /// 从管理器链表中获取指定的管理器，如果没有，那会创建一个对应的管理器，并加入管理器链表 
        public TManager GetManager<TManager>() where TManager : ManagerBase
        {
            Type managerType = typeof(TManager);
            // 检查是否存在对应管理器
            foreach (var manager in m_Managers)
            {
                if (manager.GetType() == managerType)
                {
                    return manager as TManager;
                }
            }
            // 不存在就创建
            return CreateManager(managerType) as TManager;
        }
        
        /// 创建一个管理器 
        private ManagerBase CreateManager(Type managerType)
        {
            ManagerBase manager = Activator.CreateInstance(managerType) as ManagerBase;

            if (manager == null)
            {
                throw new Exception("创建管理器失败...");
            }
            
            // 根据模块优先级决定它在链表里的位置
            LinkedListNode<ManagerBase> current = m_Managers.First;
            while (current != null)
            {
 
                if (manager.Priority < current.Value.Priority)
                {
                    break;
                }
 
                current = current.Next;
            }
            // 如果存在current，那么会在他前面插入manager
            if (current != null)
            {
 
                m_Managers.AddBefore(current, manager);
            }
            // 如果不存在current，说明所有节点的优先级都比当前manager高，那就插入到末尾
            else
            {
 
                m_Managers.AddLast(manager);
            }
 
            // 初始化管理器
            manager.Init();
            return manager;
        }
    }
}
```

SGFEntry是继承自单例脚本的，而单例模板是继承了Monobehaviour的，所以咱们的入口是存在Update等Mono方法的~

添加更新管理器的方法

```c#
using System;
using System.Collections.Generic;
using UnityEngine;


namespace SimpleGameFramework.Core
{
    public class SGFEntry : Singleton<SGFEntry>
    {
        /// 依次更新所有的管理器
        private void Update()
        {
            foreach (var manager in m_Managers)
            {
                manager.Update(Time.deltaTime);
            }
        }

        /// 倒序销毁所有的管理器
        private void OnDestroy()
        {
            for (var manager = m_Managers.Last; manager != null; manager = manager.Previous)
            {
                manager.Value.ShutDown();
            }
            m_Managers.Clear();
        }
    }
}
```

好啦，核心类暂时就是这些，但就凭这些我们还没办法测试...因为我们压根还没有任何一个管理器...

举个例子，假如我们写了一个UI管理器UIManager，那我们就可以这么调用

```c#
public class UIManager : ManagerBase
{
    public void OpenUI(UI ui);
}

public class TestMain : MonoBehaviour
{
    public void Start()
    {
    	SGFEntry.Instance.GetManager<UIManager>.OpenUI(ui);    
    }
}
```

目前咱们的结构是这样的：

> SimpleGameFramework
> 
> - Core
> 
>   SIngleton.cs
>   
>   ManagerBase.cs
>   
>   SGFEntry.cs
