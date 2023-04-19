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
        public Dictionary<long,Entity> Id2Children { get; private set; } = new Dictionary<long, Entity>();
        public Dictionary<Type,List<Entity>> Type2Children { get; private set; } = new Dictionary<Type, List<Entity>>();
        
        // 持有的组件
        public Dictionary<Type,Component> Components { get; private set; } = new Dictionary<Type, Component>();
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
        }

        public virtual void Awake()
        {
        }

        public virtual void Awake(object initData)
        {
        }

        public virtual void Start()
        {
        }

        public virtual void Start(object initData)
        {
        }

        public virtual void OnSetParent(Entity pre, Entity now)
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

添加删除各种组件的方法

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
            component.Setup();
            component.Enable = component.DefaultEnable;
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
            component.Setup(initData);
            component.Enable = component.DefaultEnable;
            return component;
        }

        public void RemoveComponent<T>() where T : Component
        {
            var component = Components[typeof(T)];
            if (component.Enable) component.Enable = false;
            Component.Destroy(component);
            Components.Remove(typeof(T));
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

        public Component GetComponent(Type componentType)
        {
            if (this.Components.TryGetValue(componentType, out var component))
            {
                return component;
            }

            return null;
        }

        public T Get<T>() where T : Component
        {
            if (Components.TryGetValue(typeof(T), out var component))
            {
                return component as T;
            }

            return null;
        }

        public bool TryGet<T>(out T component) where T : Component
        {
            if (Components.TryGetValue(typeof(T), out var c))
            {
                component = c as T;
                return true;
            }

            component = null;
            return false;
        }

        public bool TryGet<T, T1>(out T component, out T1 component1) where T : Component where T1 : Component
        {
            component = null;
            component1 = null;
            if (Components.TryGetValue(typeof(T), out var c)) component = c as T;
            if (Components.TryGetValue(typeof(T1), out var c1)) component1 = c1 as T1;
            if (component != null && component1 != null) return true;
            return false;
        }

        public bool TryGet<T, T1, T2>(out T component, out T1 component1, out T2 component2)
            where T : Component where T1 : Component where T2 : Component
        {
            component = null;
            component1 = null;
            component2 = null;
            if (Components.TryGetValue(typeof(T), out var c)) component = c as T;
            if (Components.TryGetValue(typeof(T1), out var c1)) component1 = c1 as T1;
            if (Components.TryGetValue(typeof(T2), out var c2)) component2 = c2 as T2;
            if (component != null && component1 != null && component2 != null) return true;
            return false;
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

        public bool As<T>(out T entity) where T : Entity
        {
            entity = this as T;
            return entity != null;
        }
        
        private void SetParent(Entity parent)
        {
            var preParent = Parent;
            preParent?.RemoveChild(this);
            this.parent = parent;
            OnSetParent(preParent, parent);
        }

        public void SetChild(Entity child)
        {
            Children.Add(child);
            Id2Children.Add(child.Id, child);
            if (!Type2Children.ContainsKey(child.GetType())) Type2Children.Add(child.GetType(), new List<Entity>());
            Type2Children[child.GetType()].Add(child);
            child.SetParent(this);
        }

        public void RemoveChild(Entity child)
        {
            Children.Remove(child);
            if (Type2Children.ContainsKey(child.GetType())) Type2Children[child.GetType()].Remove(child);
        }

        public Entity AddChild(Type entityType)
        {
            var entity = NewEntity(entityType);
            SetupEntity(entity, this);
            return entity;
        }

        public Entity AddChild(Type entityType, object initData)
        {
            var entity = NewEntity(entityType);
            SetupEntity(entity, this, initData);
            return entity;
        }

        public T AddChild<T>() where T : Entity
        {
            return AddChild(typeof(T)) as T;
        }

        public T AddIdChild<T>(long id) where T : Entity
        {
            var entityType = typeof(T);
            var entity = NewEntity(entityType, id);
            SetupEntity(entity, this);
            return entity as T;
        }

        public T AddChild<T>(object initData) where T : Entity
        {
            return AddChild(typeof(T), initData) as T;
        }

        public Entity GetIdChild(long id)
        {
            Id2Children.TryGetValue(id, out var entity);
            return entity;
        }

        public T GetIdChild<T>(long id) where T : Entity
        {
            Id2Children.TryGetValue(id, out var entity);
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
            get => instance ??= new MasterEntity();
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
            return Create(typeof(T)) as T;
        }

        public static T Create<T>(object initData) where T : Entity
        {
            return Create(typeof(T), initData) as T;
        }

        private static void SetupEntity(Entity entity, Entity parent)
        {
            parent.SetChild(entity);
            {
                entity.Awake();
            }
            entity.Start();
        }

        private static void SetupEntity(Entity entity, Entity parent, object initData)
        {
            parent.SetChild(entity);
            {
                entity.Awake(initData);
            }
            entity.Start(initData);
        }

        public static Entity Create(Type entityType)
        {
            var entity = NewEntity(entityType);
            SetupEntity(entity, Master);
            return entity;
        }

        public static Entity Create(Type entityType, object initData)
        {
            var entity = NewEntity(entityType);
            SetupEntity(entity, Master, initData);
            return entity;
        }

        public static void Destroy(Entity entity)
        {
            entity.OnDestroy();
            entity.Dispose();
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

        public virtual void Setup()
        {

        }

        public virtual void Setup(object initData)
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

        private void Dispose()
        {
            Enable = false;
            IsDisposed = true;
        }

        public static void Destroy(Component entity)
        {
            entity.OnDestroy();
            entity.Dispose();
        }
    }
}
```

