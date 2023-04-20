---
title: "ECGameplay03 行动机制"
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
        public Entity OwnerEntity { get; set; }
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
        /// 行动实体
        public Entity Creator { get; set; }
        /// 目标对象
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
        public Entity Action { get; set; }
        // 释放者
        public Entity Creator { get; set; }
        // 目标
        public Entity Target { get; set; }
        // 能力执行体
        public AttackAbilityExecution AttackAbilityExecution { get; set; }

        public void ApplyAttack()
        {
            var attackAbility = Creator.GetChild<AttackAbility>();
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
        public Entity OwnerEntity { get; set; }
        public Entity ParentEntity { get; }
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
    public class AttackAbility: Entity,IAbility
    {
        public Entity OwnerEntity
        {
            get=> GetParent<Entity>();
            set{}
        }

        public Entity ParentEntity => GetParent<Entity>();
        
        public bool Enable { get; set; }


        public override void Awake()
        {
            // 后文解释
            var effects = new List<Effect>();
            var damageEffect = new DamageEffect();
            damageEffect.Enabled = true;
            damageEffect.AddSkillEffectTargetType = AddSkillEffetTargetType.SkillTarget;
            damageEffect.EffectTriggerType = EffectTriggerType.Condition;
            damageEffect.CanCrit = true;
            damageEffect.DamageType = DamageType.Physic;
            damageEffect.DamageValueFormula = $"自身攻击力";
            effects.Add(damageEffect);
            AddComponent<AbilityEffectComponent>(effects);
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
            // 后文解释
            var execution = OwnerEntity.AddChild<AttackAbilityExecution>(this);
            execution.AbilityEntity = this;
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
        public Entity AbilityEntity { get; set; }
        public Entity OwnerEntity { get; set; }

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
public class AttackAbilityExecution : Entity, IAbilityExecution
{
    public Entity AbilityEntity { get; set; }
    public Entity OwnerEntity { get; set; }
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

        // 后文解释
        AttackAction.Creator.TriggerActionPoint(ActionPointType.PreGiveAttackEffect, AttackAction);
        AttackAction.Target.TriggerActionPoint(ActionPointType.PreReceiveAttackEffect, AttackAction);

        if (blocked)
        {
            Debug.LogError("被格挡了");
        }
        else
        {
            // 后文解释
            var effectAssigns = 
            	AbilityEntity.GetComponent<AbilityEffectComponent>().CreateAssignActions(AttackAction.Target);
            foreach (var item in effectAssigns)
            {
                item.AssignEffect();
            }
        	Debug.LogError("进行一次普工");
        }
    }
}
```

