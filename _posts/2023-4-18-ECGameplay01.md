---
title: "ECGameplay01 组件式编程"
tags: 战斗
---

# 概览

EC，即Entity - Component，是一种代码组织方式

Unity也是基于组件式编程的，一个GameObject上可以挂载多个相同或者不同的组件

我们的Entity，可以类比于一个空的GameObject

**同样的地方在于**，Entity本身没有任何行为和数据，他的特性由他挂载的所有脚本决定

**不同的地方在于**，Entity只是一个基类，提供一些通用方法，而我们需要实现自己所需要的所有具体Entity，Entity是数据和行为的结合体

Component代表了某一种行为，而Entity则是多种行为的结合体

# Entity

## 基础架构

主要就是记录父子节点，以及维持自身的组件映射表

```c#
namespace ECGameplay
{
    public abstract partial class Entity
    {
        // ID
        public long Id { get; set; }

        // 名称
        private string name;

        public string Name
        {
            get => name;
            set
            {
                name = value;
#if UNITY_EDITOR
    			// 后文说明
                GetComponent<GameObjectComponent>()?.OnNameChanged(name);
#endif
            }
        }

        // 实例ID
        public long InstanceId { get; set; }

        // 是否释放
        public bool IsDispose => InstanceId == 0;

        // 父实体
        private Entity parent;
        public Entity Parent => parent;

        // 孩子实体
        public List<Entity> Children { get; private set; } = new List<Entity>();
        public Dictionary<long, Entity> Id2Children { get; private set; } = new Dictionary<long, Entity>();

        public Dictionary<Type, List<Entity>> Type2Children { get; private set; } =
            new Dictionary<Type, List<Entity>>();

        // 持有的组件
        public Dictionary<Type, Component> Components { get; private set; } = new Dictionary<Type, Component>();
    }
}
```

## 生命周期

和Mono几乎一致

```c#
namespace ECGameplay
{
    public abstract partial class Entity
    {
        public Entity()
        {
#if UNITY_EDITOR
    		// 后文说明
            if (this is MasterEntity)
                return;
            AddComponent<GameObjectComponent>();
#endif
        }

        public virtual void Awake()
        {
        }

        public virtual void Awake(object initData)
        {
        }

        public virtual void Update()
        {
        }

        public virtual void OnDestroy()
        {
        }

        private void Dispose()
        {
            if (Children.Count > 0)
            {
                for (int i = Children.Count - 1; i >= 0; i--)
                {
                    Destroy(Children[i]);
                }

                Children.Clear();
                Type2Children.Clear();
            }

            Parent?.RemoveChild(this);
            foreach (var component in Components.Values)
            {
                component.Enable = false;
                Component.Destroy(component);
            }

            Components.Clear();
            InstanceId = 0;
            if (Master.Entities.ContainsKey(GetType()))
            {
                Master.Entities[GetType()].Remove(this);
            }
        }
    }
}
```

## 组件方法

添加删除各种组件的方法，组件是不可以重复添加的

```c#
namespace ECGameplay
{
    public abstract partial class Entity
    {        
        public T AddComponent<T>() where T : Component
        {
            var component = Activator.CreateInstance<T>();
            component.Entity = this;
            component.IsDisposed = false;
            Components.Add(typeof(T), component);
            Master.AllComponents.Add(component);
            component.Awake();
            component.Enable = component.DefaultEnable;

#if UNITY_EDITOR
            GetComponent<GameObjectComponent>()?.OnAddComponent(component);
#endif
            return component;
        }

        public T AddComponent<T>(object initData) where T : Component
        {
            var component = Activator.CreateInstance<T>();
            component.Entity = this;
            component.IsDisposed = false;
            Components.Add(typeof(T), component);
            Master.AllComponents.Add(component);
            component.Awake(initData);
            component.Enable = component.DefaultEnable;
#if UNITY_EDITOR
            GetComponent<GameObjectComponent>()?.OnAddComponent(component);
#endif
            return component;
        }

        public void RemoveComponent<T>() where T : Component
        {
            var component = Components[typeof(T)];
            if (component.Enable) component.Enable = false;
            Component.Destroy(component);
            Components.Remove(typeof(T));
            
#if UNITY_EDITOR
            GetComponent<GameObjectComponent>()?.OnRemoveComponent(component);
#endif
        }

        public T GetComponent<T>() where T : Component
        {
            if (Components.TryGetValue(typeof(T), out var component))
            {
                return component as T;
            }

            return null;
        }

        public bool HasComponent<T>() where T : Component
        {
            return Components.TryGetValue(typeof(T), out var component);
        }

    }
}
```

## 子节点

```c#
namespace ECGameplay
{
    public abstract partial class Entity
    {
        public T GetParent<T>() where T : Entity
        {
            return parent as T;
        }

        public T As<T>() where T : class
        {
            return this as T;
        }


        public void RemoveChild(Entity child)
        {
            Children.Remove(child);
            Id2Children.Remove(child.Id);
            if (Type2Children.ContainsKey(child.GetType())) 
                Type2Children[child.GetType()].Remove(child);
        }
        

        public T AddChild<T>() where T : Entity
        {
            var entity = NewEntity(typeof(T));
            SetupEntity(entity, this);
            return entity as T;
        }
        
        public T AddChild<T>(object initData) where T : Entity
        {
            var entity = NewEntity(typeof(T));
            SetupEntity(entity, this, initData);
            return entity as T;
        }


        public T GetChild<T>(int index = 0) where T : Entity
        {
            if (Type2Children.ContainsKey(typeof(T)) == false)
            {
                return null;
            }

            if (Type2Children[typeof(T)].Count <= index)
            {
                return null;
            }

            return Type2Children[typeof(T)][index] as T;
        }

        public Entity[] GetChildren()
        {
            return Children.ToArray();
        }

        public T[] GetTypeChildren<T>() where T : Entity
        {
            return Type2Children[typeof(T)].ConvertAll(x => x.As<T>()).ToArray();
        }

        public Entity Find(string name)
        {
            foreach (var item in Children)
            {
                if (item.name == name) return item;
            }

            return null;
        }

        public T Find<T>(string name) where T : Entity
        {
            if (Type2Children.TryGetValue(typeof(T), out var chidren))
            {
                foreach (var item in chidren)
                {
                    if (item.name == name) return item as T;
                }
            }

            return null;
        }
    }
}
```



## Create

### MasterEntity

我们需要一个根节点管理我们所有的Entity

```c#
namespace ECGameplay
{
    public class MasterEntity : Entity
    {
        // 记录所有的实体，和所有的组件
        public Dictionary<Type, List<Entity>> Entities { get; private set; } = new Dictionary<Type, List<Entity>>();
        public List<Component> AllComponents { get; private set; } = new List<Component>();

        // 单例
        private static MasterEntity instance;

        public static MasterEntity Instance
        {
            get
            {
                if (instance==null)
                {
                    instance = new MasterEntity();
                    var go = instance.AddComponent<GameObjectComponent>().GameObject;
                    Object.DontDestroyOnLoad(go);
                }

                return instance;
            }
            private set => instance = value;
        }

        // 禁止实例化
        private MasterEntity()
        {
            
        }
        
        public static void Destroy()
        {
            Destroy(Instance);
            Instance = null;
        }

        public override void Update()
        {
            if (AllComponents.Count == 0)
            {
                return;
            }
            for (int i = AllComponents.Count - 1; i >= 0; i--)
            {
                var item = AllComponents[i];
                if (item.IsDisposed)
                {
                    AllComponents.RemoveAt(i);
                    continue;
                }
                if (item.Disable)
                {
                    continue;
                }
                item.Update();
            }
        }
    }
}
```

### IdFactory

用于创建唯一ID的一个简单工具类

```c#
namespace ECGameplay
{
    public static class IdFactory
    {
        public static long BaseRevertTicks { get; set; }

        public static long NewInstanceId()
        {
            if (BaseRevertTicks == 0)
            {
                var now = DateTime.UtcNow.Ticks;
                var str = now.ToString().Reverse();
                BaseRevertTicks = long.Parse(string.Concat(str));
            }
            BaseRevertTicks++;
            return BaseRevertTicks;
        }
    }
}
```

### Entity.Create

```c#
namespace ECGameplay
{
    public abstract partial class Entity
    {
        public static MasterEntity Master => MasterEntity.Instance;

        public static Entity NewEntity(Type entityType, long id = 0)
        {
            var entity = Activator.CreateInstance(entityType) as Entity;
            entity.InstanceId = IdFactory.NewInstanceId();
            if (id == 0) entity.Id = entity.InstanceId;
            else entity.Id = id;
            if (!Master.Entities.ContainsKey(entityType))
            {
                Master.Entities.Add(entityType, new List<Entity>());
            }
            Master.Entities[entityType].Add(entity);
            return entity;
        }

        public static T Create<T>() where T : Entity
        {
            var entity = NewEntity(typeof(T));
            SetupEntity(entity, Master);
            return entity as T;
        }

        public static T Create<T>(object initData) where T : Entity
        {
            var entity = NewEntity(typeof(T));
            SetupEntity(entity, Master, initData);
            return entity as T;
        }
        
        public static void Destroy(Entity entity)
        {
            entity.OnDestroy();
            entity.Dispose();
        }

        private static void SetupEntity(Entity entity, Entity parent)
        {
            parent.SetChild(entity);
            {
                entity.Awake();
            }
        }

        private static void SetupEntity(Entity entity, Entity parent, object initData)
        {
            parent.SetChild(entity);
            {
                entity.Awake(initData);
            }
        }
        
        private void SetChild(Entity child)
        {
            Children.Add(child);
            Id2Children.Add(child.Id, child);
            if (!Type2Children.ContainsKey(child.GetType())) 
                Type2Children.Add(child.GetType(), new List<Entity>());
            Type2Children[child.GetType()].Add(child);
            child.SetParent(this);
        }
        
        private void SetParent(Entity parent)
        {
            var preParent = Parent;
            preParent?.RemoveChild(this);
            this.parent = parent;
#if UNITY_EDITOR
            parent.GetComponent<GameObjectComponent>()?.OnAddChild(this);
#endif
        }
    }
}
```

# Component

相比Entity更加简单

```c#
namespace ECGameplay
{
    public class Component
    {
        // 持有该组件的实体
        public Entity Entity { get; set; }
        
        // 是否释放
        public bool IsDisposed { get; set; }
        
        // 默认是否启用
        public virtual bool DefaultEnable { get; set; } = true;
        
        // 是否启用
        private bool enable = false;
        public bool Enable
        {
            get => enable;
            set
            {
                if (enable == value) return;
                enable = value;
                if (enable) OnEnable();
                else OnDisable();
            }
        }
        public bool Disable => enable == false;
        
        public T GetEntity<T>() where T : Entity
        {
            return Entity as T;
        }

        public virtual void Awake()
        {

        }

        public virtual void Awake(object initData)
        {

        }

        public virtual void OnEnable()
        {

        }

        public virtual void OnDisable()
        {

        }

        public virtual void Update()
        {

        }

        public virtual void OnDestroy()
        {
            
        }

        public static void Destroy(Component component)
        {
            component.Enable = false;
            component.OnDestroy();
            component.IsDisposed = true;
        }
    }
}
```

# 一些基础组件

## ComponetView

这不是一个Component，这是一个Mono类，单纯用于编辑器可视化的

大部分都是可视化方法（单纯为了好看）

用到了Odin的几个特性哈，Odin不多介绍了

```c#
namespace ECGameplay
{
    public class ComponentView : MonoBehaviour
    {
        [HideInInspector] public List<string> componts = new List<string>();

        private Random random = new Random();
        private List<Color> colors = new List<Color>();

        [OnInspectorGUI]
        private void OnInspectGUI()
        {
            PrepareColor();
            GUI.enabled = false;
            for (int i = 0; i < componts.Count; i++)
            {
                GUI.color = colors[i];
                EditorGUILayout.TextField(componts[i]);
            }

            GUI.color = Color.white;
            GUI.enabled = true;
        }

        private void PrepareColor()
        {
            if (colors.Count < componts.Count)
            {
                for (int i = colors.Count; i < componts.Count; i++)
                {
                    colors.Add(GetRandomColor());
                }
            }
        }

        private Color GetRandomColor()
        {
            var r = (float)random.NextDouble();
            var g = (float)random.NextDouble();
            var b = (float)random.NextDouble();
            return new Color(r, g, b, 1);
        }
    }
}
```

## GameObjectComponent

一个标准的Componenet，不过也是用于编辑器可视化的

```c#
namespace ECGameplay.BasicComponent
{
    public class GameObjectComponent : Component
    {
        public GameObject GameObject { get;private set; }

        public override void Awake()
        {
            GameObject = new GameObject(Entity.GetType().Name);
        }
        
        public override void OnDestroy()
        {
            base.OnDestroy();
            UnityEngine.GameObject.Destroy(GameObject);
        }
        
        public void OnNameChanged(string name)
        {
            GameObject.name = $"{Entity.GetType().Name}: {name}";
        }

        public void OnAddComponent(Component component)
        {
            var view = GameObject.AddComponent<ComponentView>();
            view.Type = component.GetType().Name;
            view.Component = component;
        }

        public void OnRemoveComponent(Component component)
        {
            var comps = GameObject.GetComponents<ComponentView>();
            foreach (var item in comps)
            {
                if (item.Component == component)
                {
                    UnityEngine.GameObject.Destroy(item);
                }
            }
        }

        public void OnAddChild(Entity child)
        {
            if (child.GetComponent<GameObjectComponent>() != null)
            {
                child.GetComponent<GameObjectComponent>().GameObject.transform.SetParent(GameObject.transform);
            }
        }
    }
}
```

## UpdateComponent

我们所有Entity上的所有Component都被集中到MasterEntity里了，每一帧都会遍历更新

虽然我们的Entity也有Update方法， 但默认是没地方调用的，这个组件的作用就让Entity也可以Update

```c#
namespace ECGameplay
{
    public class UpdateComponent : Component
    {
        public override void Update()
        {
            Entity.Update();
        }
    }
}
```

# 测试

```c#
public class Test : MonoBehaviour
{
    private MasterEntity MasterEntity => MasterEntity.Instance;
    
    public class TestEntity : Entity
    {
        public override void Update()
        {
            Debug.Log(Time.time);
        }
    }
    
    private void Start()
    {
        var entity = MasterEntity.AddChild<TestEntity>();
        entity.AddComponent<UpdateComponent>();
    }

    private void Update()
    {
        MasterEntity.Update();
    }
}
```

![image-20230419224003357](C:\Users\Logarius\AppData\Roaming\Typora\typora-user-images\image-20230419224003357.png)

![image-20230419220447242](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304192204287.png)

