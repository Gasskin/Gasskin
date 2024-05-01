---
title: "GameFramework|12 对象池"
categories: GameFramework 
tags: 框架
---

首先新建如下脚本

![1](https://www.logarius996.icu/images/SimpleGameFramework/ObjectPool/1.png)

ObjectBase是对象基类，如果某个对象想要被对象池管理，那就必须继承自这个类，实际上真正的对象被封装在这个类内部

IObjectPool是对象池接口，OBjectPool是对象池，和引用池的结构很像

ObjectPoolManager就是核心管理器了

### ObjectBase

```c#
namespace SimpleGameFramework.ObjectPool
{
    /// <summary>
    /// 池对象基类
    /// </summary>
    public abstract class ObjectBase 
    {
        #region Property

            /// <summary>
            /// 对象名称
            /// </summary>
            public string Name { get; set; }
     
            /// <summary>
            /// 对象（实际使用到的对象放到这里）
            /// </summary>
            public object Target { get; set; }
     
            /// <summary>
            /// 对象上次使用时间
            /// </summary>
            public DateTime LastUseTime { get; private set; }
     
            /// <summary>
            /// 对象的获取计数
            /// </summary>
            public int SpawnCount { get; set; }
     
            /// <summary>
            /// 对象是否正在使用
            /// </summary>
            public bool IsInUse
            {
                get
                {
                    return SpawnCount > 0;
                }
            }
     
            public ObjectBase(object target, string name = "")
            {
                Name = name;
                Target = target;
            }

        #endregion

        #region 生命周期

            /// <summary>
            /// 获取对象时
            /// </summary>
            protected virtual void OnSpawn()
            {
     
            }
     
            /// <summary>
            /// 回收对象时
            /// </summary>
            protected virtual void OnUnspawn()
            {
     
            }
     
            /// <summary>
            /// 释放对象时
            /// </summary>
            public abstract void Release();

        #endregion

        #region 接口

            /// <summary>
            /// 获取对象
            /// </summary>
            public ObjectBase Spawn()
            {
                SpawnCount++;
                LastUseTime = DateTime.Now;
                OnSpawn();
                return this;
            }
     
            /// <summary>
            /// 回收对象
            /// </summary>
            public void Unspawn()
            {
                OnUnspawn();
                LastUseTime = DateTime.Now;
                SpawnCount--;
            }

        #endregion
    }
}
```

这里的关键是接口里的两个方法，我们注意到，不管是生成对象还是归还对象，我们都只是对引用计数进行了操作，哪怕归还后引用计数为0，我们也没有释放他，注意，我们的ObjectBase，也就是被管理的对象，它自身是不会自动释放的， 单纯只是维护了一个引用计数，和生成销毁时调用的方法！而是否释放，应该由我们的对象池来进行控制。

### IObjectPool

```c#
namespace SimpleGameFramework.ObjectPool
{
    public interface IObjectPool
    {
        #region Property

            /// <summary>
            /// 对象池名称
            /// </summary>
            string Name { get; }
     
            /// <summary>
            /// 对象池对象类型
            /// </summary>
            Type ObjectType { get; }
     
     
            /// <summary>
            /// 对象池中对象的数量。
            /// </summary>
            int Count { get; }
     
     
            /// <summary>
            /// 对象池中能被释放的对象的数量。
            /// </summary>
            int CanReleaseCount { get; }
     
            /// <summary>
            /// 对象池自动释放可释放对象的间隔秒数（隔几秒进行一次自动释放）
            /// </summary>
            float AutoReleaseInterval { get; set; }
     
            /// <summary>
            /// 对象池的容量。
            /// </summary>
            int Capacity { get; set; }
     
     
            /// <summary>
            /// 对象池对象过期秒数（被回收几秒钟视为过期，需要被释放）
            /// </summary>
            float ExpireTime { get; set; }

        #endregion

        #region 释放

            /// <summary>
            /// 释放超出对象池容量的可释放对象
            /// </summary>
            void Release();
     
            /// <summary>
            /// 释放指定数量的可释放对象
            /// </summary>
            /// <param name="toReleaseCount">尝试释放对象数量。</param>
            void Release(int toReleaseCount);
     
            /// <summary>
            /// 释放对象池中的所有未使用对象
            /// </summary>
            void ReleaseAllUnused();

        #endregion

        #region Update与ShutDown

            /// <summary>
            /// 轮询对象池
            /// </summary>
            void Update(float time);
     
            /// <summary>
            /// 清理并关闭对象池
            /// </summary>
            void Shutdown();

        #endregion
    }
}
```

对象池接口，定义了我们的对象池必须实现的方法和属性

### ObjectPool

首先咱们给对象池加上一个泛型，代表它管理的对象的类型，注意一个对象池只管理一种类型的对象，然后由管理器管理很多不同类型的对象池，和引用池的设计思路是一样的

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPool<T> : IObjectPool where T : ObjectBase
    {
        
    }
}
```

#### 实现接口并添加一些字段

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPool<T> : IObjectPool where T : ObjectBase
    {
        #region Implement

            public string Name { get; private set; }
            
            public Type ObjectType
            {
                get { return typeof(T); }
            }
            
            public int Count
            {
                get { return m_Objects.Count; }
            }
            
            public int CanReleaseCount {
                get
                {
                    // 这个方法定义在了后面
                    return GetCanReleaseObjects().Count;
                }
            }
            
            public float AutoReleaseInterval { get; set; }

            public int Capacity
            {
                get { return m_Capacity; }
                set
                {
                    if (value < 0)
                        Debug.LogError("对象池容量必须大于0");

                    if (m_Capacity == value) 
                        return;

                    m_Capacity = value;
                }
            }

            public float ExpireTime
            {
                get { return m_ExpireTime; }

                set
                {
                    if (value < 0)
                        Debug.LogError("对象过期秒数必须大于0");

                    if (m_ExpireTime == value)
                        return;

                    m_ExpireTime = value;
                }
            }

            public void Release()
            {
            }
 
            public void Release(int toReleaseCount)
            {
            }
 
            public void ReleaseAllUnused()
            {
            }

            public void Update(float time)
            {
            }
 
            public void Shutdown()
            {
            }

        #endregion

        #region Field

            /// <summary>
            /// 对象池容量
            /// </summary>
            private int m_Capacity;
         
            /// <summary>
            /// 对象池对象过期秒数
            /// </summary>
            private float m_ExpireTime;
             
            /// <summary>
            /// 对象链表
            /// </summary>
            private LinkedList<ObjectBase> m_Objects;

        #endregion
        
        #region Property
         
            /// <summary>
            /// 池对象是否可被多次获取
            /// </summary>
            public bool AllowMultiSpawn { get; private set; }
         
            /// <summary>
            /// 对象池自动释放可释放对象计时
            /// </summary>
            public float AutoReleaseTime { get; private set; }

        #endregion
    }
}
```

这里比较关键的属性是对象池的对象过期时间，这是对象池自动管理的关键，之后会解释。

#### 构造方法

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPool<T> : IObjectPool where T : ObjectBase
    {
       #region 生命周期

            public ObjectPool(string name, int capacity, float expireTime, bool allowMultiSpawn)
            {
                Name = name;
                m_Objects = new LinkedList<ObjectBase>();
     
                Capacity = capacity;
                AutoReleaseInterval = expireTime; 
                ExpireTime = expireTime;
                AutoReleaseTime = 0f;
                AllowMultiSpawn = allowMultiSpawn;
            }

        #endregion
    }
}
```

可以看到，我们需要初始化的一共有4个值，依次是，对象池的名字、对象池的容量、对象池内对象的过期时间、单一对象是否允许多重引用

#### 对象的注册，获取，归还

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPool<T> : IObjectPool where T : ObjectBase
    {
        #region 对象的注册、获取、归还
            /// <summary>
            /// 注册对象
            /// </summary>
            /// <param name="obj">对象</param>
            /// <param name="spawned">对象是否已被获取</param>
            public void Register(T obj, bool spawned = false)
            {
                if (obj == null)
                {
                    Debug.LogError("要放入对象池的对象为空:" + typeof(T).FullName);
                    return;
                }
                // 已被获取就让计数+1
                if (spawned)
                {
                    obj.SpawnCount++;
                }
                m_Objects.AddLast(obj);
            }
 
            /// <summary>
            /// 获取对象
            /// </summary>
            /// <param name="name">对象名称</param>
            /// <returns>要获取的对象</returns>
            public T Spawn(string name = "")
            {
                foreach (ObjectBase obj in m_Objects)
                {
                    if (obj.Name != name)
                    {
                        continue;
                    }
 
                    if (AllowMultiSpawn || !obj.IsInUse)
                    {
                        Debug.Log("获取了对象：" + typeof(T).FullName + "/" + obj.Name);
                        return obj.Spawn() as T;
                    }
                }
                return null;
            }
 
            /// <summary>
            /// 回收对象
            /// </summary>
            public void Unspawn(ObjectBase obj)
            {
                Unspawn(obj.Target);
            }
 
            /// <summary>
            /// 回收对象
            /// </summary>
            public void Unspawn(object target)
            {
                if (target == null)
                    Debug.LogError("要回收的对象为空：" + typeof(object).FullName);
 
                foreach (ObjectBase obj in m_Objects)
                {
                    if (obj.Target == target)
                    {
                        obj.Unspawn();
                        Debug.Log("对象被回收了：" + typeof(T).FullName + "/" + obj.Name);
                        return;
                    }
                }
 
                Debug.LogError("找不到要回收的对象：" + typeof(object).FullName);
            }
          
        #endregion
    }
}
```

通过代码可以知道，所谓注册，就是把某一个对象放到对象池的对象链表中，只有这样，这个对象才能够被对象池所管理。获取与归还就很好理解了，需要注意的是，这里就能够体现变量AllowMultiSpawn的作用了，在获取的时候我们发现，如果一个对象允许多重引用，那么不管它是否正在使用中，我们都可以获取到它。当然，一般来说是不应该支持这种方式的...

#### 对象的释放

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPool<T> : IObjectPool where T : ObjectBase
    {
       #region 对象的释放
        	/// <summary>
            /// 获取所有可以释放的对象
            /// </summary>
            private LinkedList<T> GetCanReleaseObjects()
            {
                LinkedList<T> canReleaseObjects = new LinkedList<T>();
 
                foreach (ObjectBase obj in m_Objects)
                {
                    if (obj.IsInUse)
                        continue;
 
                    canReleaseObjects.AddLast(obj as T);
                }
 
                return canReleaseObjects;
            }
        #endregion
    }
}
```

对象的释放分为两部分，首先咱们获取到所有可以释放的对象，所谓可以释放，就是指对象链表里那些没有被使用的对象，但是，并不是说我们会释放所有没有被使用的对象，这个释放需要满足一定的规则，那么这个规则如何制定呢？额...是自定义的，大家可以根据自己的需求制定自己的规则...那么如何能够让我们的释放可以满足不同的自定义规则呢？那可以利用委托机制...本质就函数指针...

定义一个委托，并写好默认的匹配规则

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPool<T> : IObjectPool where T : ObjectBase
    {
       #region Delegate

        /// <summary>
        /// 释放对象筛选方法
        /// </summary>
        /// <typeparam name="T">对象类型</typeparam>
        /// <param name="candidateObjects">要筛选的对象集合</param>
        /// <param name="toReleaseCount">需要释放的对象数量</param>
        /// <param name="expireTime">对象过期参考时间</param>
        /// <returns>经筛选需要释放的对象集合</returns>
        public delegate LinkedList<T> ReleaseObjectFilterCallback<T>(LinkedList<T> candidateObjects, int toReleaseCount, DateTime expireTime) where T : ObjectBase;

        /// <summary>
        /// 默认的释放对象筛选方法（未被使用且过期的对象）
        /// </summary>
        private LinkedList<T> DefaultReleaseObjectFilterCallBack(LinkedList<T> candidateObjects, int toReleaseCount, DateTime expireTime)
        {
            LinkedList<T> toReleaseObjects = new LinkedList<T>();
 
           	// 必须保证预计过期时间是有效的
            if (expireTime > DateTime.MinValue)
            {
                // 从第一个可释放对象开始
                LinkedListNode<T> current = candidateObjects.First;
                while (current != null)
                {
                    // 对象最后使用时间 <= 过期参考时间，就需要释放
                    if (current.Value.LastUseTime <= expireTime)
                    {
                        toReleaseObjects.AddLast(current.Value);
                        LinkedListNode<T> next = current.Next;
                        candidateObjects.Remove(current);
                        // 这里限制了我们能够释放的对象数量
                        toReleaseCount--;
                        if (toReleaseCount <= 0)
                        {
                            return toReleaseObjects;
                        }
                        current = next;
                        continue;
                    }
 
                    current = current.Next;
                }
 
            }
 
            return toReleaseObjects;
        }
        
        #endregion
    }
}
```

首先定义一个委托，用来筛选对象，所谓筛选对象，就是从可以释放的对象中，找到哪些我们想要释放掉的对象。

第一个参数就是是可以释放的对象链表，通过上一步定义的函数可以得到。第二个参数是我们想要释放的数量。第三个参数是对象的过期时间，这个参数在上面我强调过，所谓过期时间，举个例子来说，如果我们对象的过期时间是10:30:40（十点三十零四十秒），而我们调用这个筛选方法的时间是10:30:20（十点三四零二十秒），那么肯定对象还没有过期啊！我们自然就不能释放！否则就发生了错误！

另外需要强调的是，首先我们定义了一个筛选对象的委托，这个委托是作为一个模板的，也就是筛选规则必须满足这个委托指定的参数和返回值，但是其具体内容是自定义的，比如说我们给出了我们默认的筛选规则，但只要符合这个委托的参数和返回值，其具体内容完全是可以由大家自己定义的！

最后是真正的释放对象

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPool<T> : IObjectPool where T : ObjectBase
    {
       #region 对象的释放          
            /// <summary>
            /// 释放对象池中的可释放对象
            /// </summary>
            /// <param name="toReleaseCount">尝试释放对象数量</param>
            /// <param name="releaseObjectFilterCallback">释放对象筛选方法</param>
            public void Release(int toReleaseCount, ReleaseObjectFilterCallback<T> releaseObjectFilterCallback)
            {
                // 重置计时
                AutoReleaseTime = 0;
 
                if (toReleaseCount <= 0)
                    return;
 
                // 计算对象过期参考时间
                DateTime expireTime = DateTime.MinValue;
                if (m_ExpireTime < float.MaxValue)
                    // 当前时间 - 过期秒数 = 过期参考时间
                    expireTime = DateTime.Now.AddSeconds(-m_ExpireTime);
 
                // 获取可释放的对象和实际要释放的对象
                LinkedList<T> canReleaseObjects = GetCanReleaseObjects();
                LinkedList<T> toReleaseObjects = releaseObjectFilterCallback(canReleaseObjects, toReleaseCount, expireTime);
                if (toReleaseObjects == null || toReleaseObjects.Count <= 0)
                    return;
 
                // 遍历实际要释放的对象
                foreach (ObjectBase toReleaseObject in toReleaseObjects)
                {
                    if (toReleaseObject == null)
                        Debug.LogError("无法释放空对象");
 
                    foreach (ObjectBase obj in m_Objects)
                    {
                        if (obj != toReleaseObject)
                            continue;
 
                        // 释放对象
                        m_Objects.Remove(obj);
                        obj.Release();
                        Debug.Log("对象被释放了：" + obj.Name);
                        break;
                    }
                }
 
            }

        #endregion
    }
}
```

这里的关键在于，我们如何定义我们对象的过期时间，想理解这个问题，那我们必须知道Release这个方法会在什么时候被调用，接着往下看

#### 接口方法实现

Update和ShutDown

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPool<T> : IObjectPool where T : ObjectBase
    {
       #region Implement
        	/// <summary>
            /// 对象池的定时释放
            /// </summary>
            public void Update(float time)
            {
                AutoReleaseTime += time;
                if (AutoReleaseTime < AutoReleaseInterval)
                    return;
 
                Release();
            }
 
            /// <summary>
            /// 清理对象池
            /// </summary>
            public void Shutdown()
            {
                LinkedListNode<ObjectBase> current = m_Objects.First;
                while (current != null)
                {
 
                    LinkedListNode<ObjectBase> next = current.Next;
                    m_Objects.Remove(current);
                    current.Value.Release();
                    Debug.Log("对象被释放了：" + current.Value.Name);
                    current = next;
                }
            }
        #endregion
    }
}
```

咱们看Update，首先，Update是一个固定时间的检查，假设AutoReleaseInterval是10，那就是每10秒会执行一次Release()，当然，只是尝试释放，并不是说每10秒就会释放一次对象。

而Release()，最终还是调用的我们上面所写的那个方法。那我们假设我们会在15秒的时候归还一个对象，并且对象的过期时间是3秒。

好，时间从0开始，0 1 2 ...10，第一次尝试释放，但是这时候我们还没有归还对象，所以啥也释放不掉

然后继续 11 12 13 14 15 UnSpawn() 16 17 18 19 20 ，UnSpawn()会标记对象的最后使用时间为15

（注意，UnSpawn()并不是直接归还对象，它只是把对象的引用计数-1，如果引用计数=0了，那么这个对象就会被标记为未使用，即可以释放）

好了，20秒的时候我们第二次尝试释放对象，因为对象的过期时间为3秒，那么20秒的时候，我们只会释放最后使用时间在17秒之前的对象（着重理解这句话）

而我们释放的对象，被标记为15秒，那么他自然就会被释放了！

最后，实现我们的另外一些接口

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPool<T> : IObjectPool where T : ObjectBase
    {
       #region Implement
        	/// <summary>
            /// 释放超出对象池容量的可释放对象
            /// </summary>
            public void Release()
            {
                Release(m_Objects.Count - m_Capacity, DefaultReleaseObjectFilterCallBack);
            }
 
            /// <summary>
            /// 释放指定数量的可释放对象
            /// </summary>
            /// <param name="toReleaseCount"></param>
            public void Release(int toReleaseCount)
            {
                Release(toReleaseCount, DefaultReleaseObjectFilterCallBack);
            }
 
            /// <summary>
            /// 释放对象池中所有未使用对象
            /// </summary>
            public void ReleaseAllUnused()
            {
                LinkedListNode<ObjectBase> current = m_Objects.First;
                while (current != null)
                {
                    if (current.Value.IsInUse)
                    {
                        current = current.Next;
                        continue;
                    }
 
                    LinkedListNode<ObjectBase> next = current.Next;
                    m_Objects.Remove(current);
                    current.Value.Release();
                    Debug.Log("对象被释放了：" + current.Value.Name);
                    current = next;
                }
            }
        #endregion
    }
}
```

对象池到这里就基本结束了，然后是管理器

### ObjectPoolManager

继承ManagerBase，实现一些基本内容

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPoolManager : ManagerBase
    {
        #region Implement
            public override int Priority 
            {
                get
                {
                    return ManagerPriority.ObjectPoolManager.GetHashCode();
                } 
            }
            
            public override void Init()
            {
                m_ObjectPools = new Dictionary<string, IObjectPool>();
            }

            public override void Update(float time)
            {
                foreach (IObjectPool objectPool in m_ObjectPools.Values)
                {
                    objectPool.Update(time);
                }

            }

            public override void ShutDown()
            {
                foreach (IObjectPool objectPool in m_ObjectPools.Values)
                {
                    objectPool.Shutdown();
                }
                m_ObjectPools.Clear();
            }

        #endregion
    }
}
```

添加一些属性

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPoolManager : ManagerBase
    {
        #region Field

            /// <summary>
            /// 默认对象池容量
            /// </summary>
            private const int DefaultCapacity = int.MaxValue;
     
            /// <summary>
            /// 默认对象过期秒数
            /// </summary>
            private const float DefaultExpireTime = float.MaxValue;
            
            /// <summary>
            /// 对象池字典
            /// </summary>
            private Dictionary<string, IObjectPool> m_ObjectPools;

        #endregion

        #region Property

        /// <summary>
        /// 对象池数量
        /// </summary>
        public int Count
        {
            get { return m_ObjectPools.Count; }
        }

        #endregion
    }
}
```

最后是对于对象池的管理

```c#
namespace SimpleGameFramework.ObjectPool
{
    public class ObjectPoolManager : ManagerBase
    {
        #region 对象池管理

            /// <summary>
            /// 检查对象池
            /// </summary>
            public bool HasObjectPool<T>() where T : ObjectBase
            {
                return m_ObjectPools.ContainsKey(typeof(T).FullName);
            }
            
            /// <summary>
            /// 创建对象池
            /// </summary>
            public ObjectPool<T> CreateObjectPool<T>(int capacity = DefaultCapacity, float exprireTime = DefaultExpireTime, bool allowMultiSpawn = false) where T : ObjectBase
            {
                string name = typeof(T).FullName;
                if (HasObjectPool<T>())
                {
                    Debug.LogError("要创建的对象池已存在");
                    return null;
                }
                ObjectPool<T> objectPool = new ObjectPool<T>(name, 0, 2, allowMultiSpawn);
                m_ObjectPools.Add(name, objectPool);
                return objectPool;
            }
 
            /// <summary>
            /// 获取对象池
            /// </summary>
            public ObjectPool<T> GetObjectPool<T>() where T : ObjectBase
            {
                IObjectPool objectPool = null;
                m_ObjectPools.TryGetValue(typeof(T).FullName, out objectPool);
                return objectPool as ObjectPool<T>;
            }
 
            /// <summary>
            /// 销毁对象池
            /// </summary>
            public bool DestroyObjectPool<T>()
            {
                IObjectPool objectPool = null;
                if (m_ObjectPools.TryGetValue(typeof(T).FullName, out objectPool))
                {
                    objectPool.Shutdown();
                    return m_ObjectPools.Remove(typeof(T).FullName);
                }
 
                return false;
            }
            
            /// <summary>
            /// 释放所有对象池中的可释放对象。
            /// </summary>
            public void Release()
            {
                foreach (IObjectPool objectPool in m_ObjectPools.Values)
                {
                    objectPool.Release();
                }
            }
 
            /// <summary>
            /// 释放所有对象池中的未使用对象。
            /// </summary>
            public void ReleaseAllUnused()
            {
                foreach (IObjectPool objectPool in m_ObjectPools.Values)
                {
                    objectPool.ReleaseAllUnused();
                }
            }

        #endregion
    }
}
```

最后测试一下

```c#
public class TestObject : ObjectBase
{
    public TestObject(object target, string name = "") : base(target, name)
    {
    }

    public override void Release()
    {
        
    }
}

public class test : MonoBehaviour
{
    private ObjectPoolManager objectPoolManager;
    private ObjectPool<TestObject> testPool;
    void Start()
    {
        objectPoolManager = SGFEntry.Instance.GetManager<ObjectPoolManager>();
        testPool = objectPoolManager.CreateObjectPool<TestObject>();

        TestObject temp = new TestObject("HelloWorld","test1");
        testPool.Register(temp);
    }

    private void Update()
    {
        if (Input.GetMouseButtonDown(0))
        {
            TestObject testObject = testPool.Spawn("test1");
            Debug.Log(testObject.Target);
            testPool.Unspawn(testObject.Target);
        }
    }
}
```

![1](https://www.logarius996.icu/images/SimpleGameFramework/ObjectPool/2.png)





























