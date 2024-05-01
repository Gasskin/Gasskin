---
title: "GameFramework|07 引用池"
categories: GameFramework 
tags: 框架
---

引用池，和对象池的概念非常像，但是对象池服务于具体的GameObject对象，而引用池服务于普通的C#类。举个例子，如果打开UI的时候你想传递一些数据，最简单的方法就是

```c#
class UIData
{
    //...
}
        
main()
{
	UIData data = new UIData();
	OpenUI(data);
}
```

这个方法是我在上一节末尾提到的，即定义一个数据类型，然后打开UI的时候直接传进函数里，说实话我感觉这样做...其实也挺好的...很简单直观，感觉也没啥问题...

咳咳，但是！但是你想挑毛病还是有的...比如这个data，每次我们打开一个UI，都需要new一个，然后传参数进去。众所周知，频繁的new肯定是不得劲的！所以引用池避免的就是这个问题...

是不是很对象池很像？对象池避免的是频繁生成游戏对象，而引用池避免的是频繁生成普通对象

### 引用对象接口

```c#
/// 引用接口，实现这个接口的类会被引用池管理
public interface IReference
{
    /// 归还引用时的清理方法
    void Clear();
}
```

想要某些对象能够被统一的管理，那必然需要一个统一的接口，关于其中的Clear方法，则是当我们归还这个对象时调用的，类似于清空数据，恢复默认状态

### 引用集合

用于收集同一类的所有对象

```c#
/// 引用集合，收集同一类型的所有引用，会被引用池直接管理
public class ReferenceCollection   
{       
    #region Private      
    /// 这个队里里储存里我们目前所有可以使用的对象
    private Queue<IReference> m_References;       
    #endregion
       
    #region 构造方法        
    public ReferenceCollection()        
    {            
        m_References = new Queue<IReference>();      
    }        
    #endregion    
}
```

主要只有两个方法，获取引用和归还引用

```c#
public class ReferenceCollection   
{       
#region Public 接口方法
        /// 获取指定类型的引用，核心就是，如果引用队列内有空引用，那么直接获取，如果没有，那么new一个
        public T Acquire<T>() where T : class, IReference, new()
        {
            // =========== 这一部分是用于DEBUG的，会在Hierachy中显示出当前的引用状态 ==========
			#if UNITY_EDITOR
            var trans = SGFEntry.Instance.transform.Find("ReferenceManager");
            var collection = trans.Find(typeof(T).FullName);
			#endif
            // =========================================================================
            
            // 这部分是获取引用的算法
            lock (m_References)
            {
                if (m_References.Count > 0)
                {
                    // =========== 这一部分是用于DEBUG的，会在Hierachy中显示出当前的引用状态 ==========
					#if UNITY_EDITOR
                    if (collection != null)
                    {
                        // 找到一个active的孩子，设置为false
                        for (int i = 0; i < collection.childCount; i++)
                        {
                            if (collection.GetChild(i).gameObject.activeSelf)
                            {
                                collection.GetChild(i).gameObject.SetActive(false);
                                break;
                            }
                        }
                    }
					#endif
                    // =========================================================================
                    
                    return m_References.Dequeue() as T;
                }
            }
            
            // =========== 这一部分是用于DEBUG的，会在Hierachy中显示出当前的引用状态 ==========
			#if UNITY_EDITOR
            if (collection != null)
            {
                GameObject go = new GameObject();
                go.name = "reference";
                go.transform.SetParent(collection);
                go.gameObject.SetActive(false);
            }
			#endif
            // =========================================================================
            
            return new T();
        }
        
        /// 释放引用
        public void Release<T>(T reference) where T : class, IReference
        {
            // 会在这里调用引用对象的Clear方法，清空数据
            reference.Clear();
            lock (m_References)
            {
                // =========== 这一部分是用于DEBUG的，会在Hierachy中显示出当前的引用状态 ==========
				#if UNITY_EDITOR
                var trans = SGFEntry.Instance.transform.Find("ReferenceManager");
                var collection = trans.Find(typeof(T).FullName);
                if (collection != null)
                {
                    for (int i = 0; i < collection.childCount; i++)
                    {
                        if (!collection.GetChild(i).gameObject.activeSelf)
                        {
                            collection.GetChild(i).gameObject.SetActive(true);
                            break;
                        }
                    }
                }
				#endif
                // =========================================================================
                
                m_References.Enqueue(reference);
            }
        }

        /// 删除所有引用
        public void RemoveAll()
        {
            lock (m_References)
            {
               m_References.Clear();
            }
        }

#endregion
}
```

### 引用池管理器

注意，上面写的引用收集，只是收集了同一类的对象，或者说，是某一种引用。而我们的游戏里当然不可能只有这么一种对象，因此，我们还是需要用一个管理器来管理所有的引用收集器的。

```c
public class ReferenceManager : ManagerBase
{
#region Private
    
        /// 维护所有的引用集合，每一个集合内有多个同类型引用
        private Dictionary<string, ReferenceCollection> s_ReferenceCollections;

#endregion

#region 构造函数

        public ReferenceManager()
        {
            s_ReferenceCollections = new Dictionary<string, ReferenceCollection>();
        }

#endregion
}    
```

基本的获取与归还

```c#
public class ReferenceManager : ManagerBase
{
#region Public 接口方法

        /// 从引用集合获取引用
        public  T Acquire<T>() where T : class, IReference, new()
        {
            // =========== 这一部分是用于DEBUG的，会在Hierachy中显示出当前的引用状态 ==========
			#if UNITY_EDITOR
            var trans = SGFEntry.Instance.transform.Find("ReferenceManager");
            var collection = trans.Find(typeof(T).FullName);
            if (collection == null)
            {
                GameObject go = new GameObject();
                go.name = typeof(T).FullName;
                go.transform.SetParent(trans);
            }
			#endif
            // =========================================================================
            
            return GetReferenceCollection(typeof(T).FullName).Acquire<T>();
        }
        
        /// 将引用归还引用集合
        public  void Release<T>(T reference) where T : class, IReference
        {
            if (reference == null)
            {
                throw new Exception("要归还的引用为空...");
            }

            GetReferenceCollection(typeof(T).FullName).Release(reference);
        }

        /// 清除所有引用集合
        public  void ClearAll()
        {
            lock (s_ReferenceCollections)
            {
                foreach (KeyValuePair<string, ReferenceCollection> referenceCollection in s_ReferenceCollections)
                {
                    referenceCollection.Value.RemoveAll();
                }
 
                s_ReferenceCollections.Clear();
            }
        }
        
        /// 从引用集合中移除所有的引用
        public  void RemoveAll<T>() where T : class, IReference
        {
            GetReferenceCollection(typeof(T).FullName).RemoveAll();
        }

#endregion

#region Private 工具方法

        /// 获取引用集合，实际上获取引用的方法
        private  ReferenceCollection GetReferenceCollection(string fullName)
        {
            ReferenceCollection referenceCollection = null;
            lock (s_ReferenceCollections)
            {
                if (!s_ReferenceCollections.TryGetValue(fullName, out referenceCollection))
                {
                    referenceCollection = new ReferenceCollection();
                    s_ReferenceCollections.Add(fullName, referenceCollection);
                }
            }
 
            return referenceCollection;
        }
        
#endregion

} 
```

重写一些接口方法，还有Update方法

```c#
public class ReferenceManager : ManagerBase
{
#region Private

        /// 清理间隔，每过一段时间，就会清空队列里的引用（还在队列里，说明是空引用），m_temp用于实际计算，想要修改清理间隔可以修改clearInterval
        private float m_ClearInterval = ManagerConfig.ClearInterval;
        private float m_Temp;
        
#endregion
    
#region Override

        public override int Priority
        {
            get
            {
                return ManagerPriority.ReferenceManager.GetHashCode();
            }
        }

        public override void Init()
        {
            m_Temp = m_ClearInterval;
        }

    	// 每过一定时间，就会把引用池清空
        public override void Update(float time)
        {
            m_Temp -= time;
            if (m_Temp < 0f) 
            {
                // =========== 这一部分是用于DEBUG的，会在Hierachy中显示出当前的引用状态 ==========
				#if UNITY_EDITOR
                var trans = SGFEntry.Instance.transform.Find("ReferenceManager");
                for (int i = 0; i < trans.childCount; i++)
                {
                    GameObject.DestroyImmediate(trans.GetChild(i).gameObject);
                }
				#endif
                // =========================================================================
                
                foreach (var e in s_ReferenceCollections)
                {
                    e.Value.RemoveAll();
                }

                m_Temp = m_ClearInterval;
            }
        }
        
#endregion
}  
```

测试一下~

```c#
public class TestRef : IReference
{
    public string testName="引用池测试";
    public void Clear()
    {
        Debug.Log("TestRef被清空了");
    }
}

public class test : MonoBehaviour
{
    private ReferenceManager referenceManager;
    void Start()
    {
        referenceManager = SGFEntry.Instance.GetManager<ReferenceManager>();
        var tempRef = referenceManager.Acquire<TestRef>();
        Debug.Log(tempRef.testName);
        // 一定要在合适的事件归还引用，如果只获取，不归还，那和无限new没有任何区别
        referenceManager.Release(tempRef);
    }
}
```

![1](https://www.logarius996.icu/images/SimpleGameFramework/Reference/1.png)![1](https://www.logarius996.icu/images/SimpleGameFramework/Reference/2.png)











































