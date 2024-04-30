---
title: "ECGameplay|03 行动机制"
tags: 战斗
---

# 概览

对一次普工行为进行拆分，可以有如下阶段

> **普工行动** 行为入口，可以有各种判断，比如是否可以进行普工等
>
> **普工行动执行** 整个普工行为的数据中心，进行任何数据中转，或者逻辑中转
>
> **普工能力** 普工行为所需要依赖的普工能力，每一个英雄的能力可能是不一样的，所以会有不同的能力
>
> **普工能力执行** 最终对普工能力进行逻辑处理的地方

# IAction

行动接口

```c#
namespace ECGameplay
{
    /// <summary>
    /// 行动
    /// </summary>
    public interface IAction
    {
        // 后改为 CombatEntity
        public Entity OwnerEntity { get; set; }
        public bool Enable { get; set; }
    }
}
```

以普工行动为例，很简单，就是一个入口类，我们会创造一个AttackActionExecution（行动执行体），当然也可以在这个入口进行各种判断

同时继承了Entity，也因此我们把基类抽象成接口了（不能多集成

```c#
namespace ECGameplay
{
    public class AttackAction : Entity,IAction
    {
        // 后改为 CombatEntity
        public Entity OwnerEntity {get => GetParent<Entity>();set{}}
        public bool Enable { get; set; }
        
        public bool TryMakeAction(out AttackActionExecution action)
        {
            if (Enable == false)
            {
                action = null;
            }
            else
            {
                // 后文解释
                action = OwnerEntity.AddChild<AttackActionExecution>();
                action.ActionAbility = this;
                action.Creator = OwnerEntity;
            }
            return Enable;
        }
    }
}
```

# IActionExecution

行动执行体

```c#
namespace ECGameplay
{
    public interface IActionExecution
    {
        /// 行动能力
        public Entity ActionAbility { get; set; }
        /// 后文解释，效果赋给行动，不是每种行为都会附带效果赋给行为的
        public EffectAssignActionExecution EffectActionExecution { get; set; }
        /// 行动实体 后改为 CombatEntity
        public Entity Creator { get; set; }
        /// 目标对象 后改为 CombatEntity
        public Entity Target { get; set; }
        /// 行动结束
        public void FinishAction();
    }
}
```

以AttackActionExecution为例

行为执行体，是整个行动过程中的中间类，本身不会有复杂的逻辑，但是他需要负责整个行为逻辑/数据的中转

比如普工行为执行体，就会提供一个入口去执行能力，并且为能力执行体记录下所有必须的数据

```c#
namespace ECGameplay
{
    public class AttackActionExecution : Entity, IActionExecution
    {
        // 行为
        public IAction Action { get; set; }
		// 效果应用行为，不过攻击行为执行体是没有的
        public EffectAssignActionExecution EffectActionExecution { get; set; }
        // 释放者
        public CombatEntity Creator { get; set; }
        // 目标
        public CombatEntity Target { get; set; }
        // 能力执行体
        public AttackAbilityExecution AttackAbilityExecution { get; set; }

        public void ApplyAttack()
        {
            // 获取Creator的攻击能力
            var attackAbility = Creator.GetChild<AttackAbility>();
            // 创建一个能力执行体，并开始执行
            AttackAbilityExecution = attackAbility.CreateExecution().As<AttackAbilityExecution>();
            AttackAbilityExecution.AttackActionExecution = this;
            AttackAbilityExecution.BeginExecute();
        }      
        public void FinishAction()
        {
            Destroy(this);
        }
    }
}
```

# IAbility

真正的能力的数据类，负责记录当前能力的各种数值，以及创建能力执行体

```c#
namespace ECGameplay
{
    /// <summary>
    /// 能力
    /// </summary>
    public interface IAbility
    {
        // 后改为 CombatEntity
        public Entity OwnerEntity { get; set; }
        public bool Enable { get; set; }

        /// 尝试激活能力
        public void TryActivateAbility();

        /// 激活能力
        public void ActivateAbility();

        /// 禁用能力
        public void DeactivateAbility();

        /// 结束能力
        public void EndAbility();

        /// 创建能力执行体
        public Entity CreateExecution();
    }
}
```

以普工为例，主要有两个地方

1. 在Awake时候初始化能力数据
2. 提供一个方法创建能力执行体

```c#
namespace ECGameplay
{
    public class AttackAbility : Entity, IAbility
    {
        public CombatEntity OwnerEntity
        {
            get => GetParent<CombatEntity>();
            set { }
        }

        public bool Enable { get; set; }
        
        private SkillConfig SkillConfig { get; set; }

        public override void Awake(object initObject)
        {
            // 需要传入具体的能力配表
            if (initObject is SkillConfig skillConfig)
            {
                // 后文解释，能力效果组件
                AddComponent<AbilityEffectComponent>(skillConfig);
            }
        }

        public void TryActivateAbility()
        {
        }

        public void ActivateAbility()
        {
            Enable = true;
        }

        public void DeactivateAbility()
        {
        }

        public void EndAbility()
        {
        }

        public Entity CreateExecution()
        {
            // 添加一个能力执行体
            var execution = OwnerEntity.AddChild<AttackAbilityExecution>(this);
            execution.Ability = this;
            return execution;
        }
    }
}
```

# IAbilityExection

能力执行体，最终对能力进行逻辑处理的地方，会持有行动执行体（为了获取行动期间的所有必须数据

```c#
namespace ECGameplay
{
    /// <summary>
    /// 能力执行体
    /// </summary>
    public interface IAbilityExecution
    {
        public IAbility Ability { get; set; }
        public CombatEntity OwnerEntity { get; set; }

        /// 开始执行
        public void BeginExecute();

        /// 结束执行
        public void EndExecute();
    }
}
```

以普工为例，普工能力执行体，是真正的对普工进行逻辑处理的地方

当开始普工时，我们会给自身Entity添加一个UpdateComponent

当检测到攻击时，再进行逻辑处理

整个能力结束后，再释放行为执行体

```c#
namespace ECGameplay
{
    public class AttackAbilityExecution : Entity, IAbilityExecution
    {
        // 持有的能力
        public IAbility Ability { get; set; }
        // 后改为CombatEntity
        public Entity OwnerEntity { get; set; }
        // 对应的行为执行体
        public AttackActionExecution AttackActionExecution { get; set; }

        // 被格挡
        private bool blocked;

        // 已经触发伤害
        private bool damaged;

        public void BeginExecute()
        {
            AddComponent<UpdateComponent>();
        }

        public void EndExecute()
        {
            AttackActionExecution.FinishAction();
            Destroy(this);
        }
        
        public void SetBlocked()
        {
            blocked = true;
        }

        public override void Update()
        {
            if (!IsDispose)
            {
                if (!damaged)
                {
                    TryTriggerAttackEffect();
                }
                else
                {
                    EndExecute();
                }
            }
        }
        
        private void TryTriggerAttackEffect()
        {
            damaged = true;

            // 行动点
            AttackActionExecution.Creator?.TriggerActionPoint(ActionPointType.BeforeGiveAttackEffect, AttackActionExecution);
            AttackActionExecution.Target?.TriggerActionPoint(ActionPointType.BeforeReceiveAttackEffect, AttackActionExecution);

            if (blocked)
            {
                Debug.LogError("被格挡了");
                return;
            }
            
            // 创建所有的效果应用行为，并执行，后文解释
            var actionExecutions = (Ability as AttackAbility)?.GetComponent<AbilityEffectComponent>()
                .CreateAssignActions(AttackActionExecution.Target);
            if (actionExecutions == null) return;
            foreach (var actionExecution in actionExecutions)
                actionExecution.AssignEffect();
            
            Debug.LogError("进行一次普工");
            
            // 行动点
            AttackActionExecution.Creator?.TriggerActionPoint(ActionPointType.AfterGiveAttack, AttackActionExecution);
            AttackActionExecution.Target?.TriggerActionPoint(ActionPointType.AfterReceiveAttack, AttackActionExecution);
        }
    }
}
```

# ActionPoint

行动点，其实就是一个行为各个阶段，抛个事件出来而已

```c#
namespace ECGameplay
{
    public sealed class ActionPoint
    {
        public List<Action<Entity>> Listeners { get; set; } = new List<Action<Entity>>();


        public void AddListener(Action<Entity> action)
        {
            Listeners.Add(action);
        }

        public void RemoveListener(Action<Entity> action)
        {
            Listeners.Remove(action);
        }

        public void TriggerAllActions(Entity actionExecution)
        {
            if (Listeners.Count == 0)
            {
                return;
            }
            for (int i = Listeners.Count - 1; i >= 0; i--)
            {
                var item = Listeners[i];
                item.Invoke(actionExecution);
            }
        }
    }
}
```

## ActionPointComponent

也很简单，就是基于ActionPoint的一个字典形消息管理器

```c#
namespace ECGameplay
{
    [Flags]
    public enum ActionPointType
    {
        [LabelText("（空）")]
        None = 0,

        [LabelText("造成伤害前")]
        BeforeCauseDamage = 1 << 1,
        [LabelText("承受伤害前")]
        BeforeReceiveDamage = 1 << 2,

        [LabelText("造成伤害后")]
        AfterCauseDamage = 1 << 3,
        [LabelText("承受伤害后")]
        AfterReceiveDamage = 1 << 4,

        [LabelText("给予治疗后")]
        AfterGiveCure = 1 << 5,
        [LabelText("接受治疗后")]
        AfterReceiveCure = 1 << 6,

        [LabelText("赋给技能效果")]
        AssignEffect = 1 << 7,
        [LabelText("接受技能效果")]
        ReceiveEffect = 1 << 8,

        [LabelText("赋加状态后")]
        AfterGiveStatus = 1 << 9,
        [LabelText("承受状态后")]
        AfterReceiveStatus = 1 << 10,

        [LabelText("给予普攻前")]
        BeforeGiveAttack = 1 << 11,
        [LabelText("给予普攻后")]
        AfterGiveAttack = 1 << 12,

        [LabelText("遭受普攻前")]
        BeforeReceiveAttack = 1 << 13,
        [LabelText("遭受普攻后")]
        AfterReceiveAttack = 1 << 14,

        [LabelText("起跳前")]
        BeforeJumpTo= 1 << 15,
        [LabelText("起跳后")]
        AfterJumpTo = 1 << 16,

        [LabelText("施法前")]
        BeforeSpell = 1 << 17,
        [LabelText("施法后")]
        AfterSpell = 1 << 18,

        [LabelText("赋给普攻效果前")]
        BeforeGiveAttackEffect = 1 << 19,
        [LabelText("赋给普攻效果后")]
        AfterGiveAttackEffect = 1 << 20,
        [LabelText("承受普攻效果前")]
        BeforeReceiveAttackEffect = 1 << 21,
        [LabelText("承受普攻效果后")]
        AfterReceiveAttackEffect = 1 << 22,

        Max,
    }
    
    public sealed class ActionPointComponent : Component
    {
        private Dictionary<ActionPointType, ActionPoint> ActionPoints { get; set; } 
        	= new Dictionary<ActionPointType, ActionPoint>();


        public void AddListener(ActionPointType actionPointType, Action<Entity> action)
        {
            if (!ActionPoints.ContainsKey(actionPointType))
            {
                ActionPoints.Add(actionPointType, new ActionPoint());
            }
            ActionPoints[actionPointType].AddListener(action);
        }

        public void RemoveListener(ActionPointType actionPointType, Action<Entity> action)
        {
            if (ActionPoints.ContainsKey(actionPointType))
            {
                ActionPoints[actionPointType].RemoveListener(action);
            }
        }

        public ActionPoint GetActionPoint(ActionPointType actionPointType)
        {
            if (ActionPoints.TryGetValue(actionPointType, out var actionPoint)) ;
            return actionPoint;
        }

        public void TriggerActionPoint(ActionPointType actionPointType, Entity actionExecution)
        {
            if (ActionPoints.TryGetValue(actionPointType, out var actionPoint))
            {
                actionPoint.TriggerAllActions(actionExecution);
            }
        }
    }
}
```

# CombatEntity

我们当然可以基于Entity，就像上面的例子那样，但是每次都这样创建实在是太麻烦了，大多数战斗单位的很多行为其实是通用的，简单封装一下

```c#
namespace ECGameplay
{
    public class CombatEntity : Entity
    {
        public GameObject target;
        public AttackAction AttackAction { get; private set; }
        public AttackAbility AttackAbility { get; private set; }

        public override void Awake()
        {
            AddComponent<AttributeComponent>();
            AddComponent<ActionPointComponent>();

            AttackAction = AttachAction<AttackAction>();

            AttackAbility = AttachAbility<AttackAbility>();
        }

        public void ListenActionPoint(ActionPointType actionPointType, Action<Entity> action)
        {
            GetComponent<ActionPointComponent>().AddListener(actionPointType, action);
        }

        public void UnListenActionPoint(ActionPointType actionPointType, Action<Entity> action)
        {
            GetComponent<ActionPointComponent>().RemoveListener(actionPointType, action);
        }

        public void TriggerActionPoint(ActionPointType actionPointType, Entity action)
        {
            GetComponent<ActionPointComponent>().TriggerActionPoint(actionPointType, action);
        }

        private T AttachAbility<T>(int id) where T : Entity, IAbility
        {
            // 配表读取，后文解释
            if (TableUtil.Tables.SkillTable.DataMap.TryGetValue(id, out var config))
            {
                return AddChild<T>(config);
            }

            Debug.LogError("AttachAbility Error : " + id);
            return null;
        }

        private T AttachAction<T>(object config = null) where T : Entity, IAction
        {
            var action = config == null ? AddChild<T>() : AddChild<T>(config);
            action.Enable = true;
            return action;
        }
    }
}
```

本身没啥逻辑，其实就是把上述的组件整合到一个通用类里了

后续这个类会不断扩展，不再说明

## MathUtil

工具类

```c#
namespace ECGameplay
{
    public static class MathUtil
    {
        private static Random random = new Random();

        /// <summary>
        /// 抽奖
        /// </summary>
        /// <param name="probability">概率（0-1）</param>
        /// <returns>是否抽中</returns>
        public static bool PrizeDraw(float probability)
        {
            probability = Math.Clamp(probability, 0, 1);
            if (probability == 0)
                return false;
            if (Math.Abs(probability - 1) < 0.00001f)
                return true;
            return (float)random.NextDouble() <= probability;
        }
    }
}
```

## BlockAction

所有逻辑一次性写完了，确实也不会有问题，因为这是很简单的一个行为

但是复杂行为还是建议拆分

```c#
namespace ECGameplay
{
    /// <summary>
    /// 格挡行为
    /// </summary>
    public class BlockAction : Entity,IAction
    {
        public CombatEntity OwnerEntity
        {
            get=> GetParent<CombatEntity>();
            set{}
        }
        
        public bool Enable { get; set; }

        public override void Awake()
        {
            OwnerEntity?.ListenActionPoint(ActionPointType.PreReceiveAttackEffect, TryBlock);
        }

        
        private void TryBlock(Entity actionExecution)
        {
            // 模拟概率30%
            var flag = MathUtil.PrizeDraw(0.3f);
            if (flag)
            {
                var attackExecution = actionExecution.As<AttackActionExecution>();
                attackExecution.AttackAbilityExecution.SetBlocked();
            }
        }
    }
}
```











































