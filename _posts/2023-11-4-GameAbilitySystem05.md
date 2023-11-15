---
title: "GameAbilitySystem|05 GameAbility 技能/能力"
tags: 战斗
---

# GameAbility

简称GA，GA其实应该叫做 **能力** 而不是 **技能**，在GAS中，挨打也是一种GA，所以应该是：“角色具有挨打的能力”，任何行为都可以是一个GA，但还是不建议把基础的移动控制作为一种GA...

能力其实比技能更加抽象，比如我拥有释放火焰的能力，只约定了你在什么时候可以释放火焰，具体怎么释放，动作是什么，效果是什么，这些都不由GA决定，GA只是纯粹的约定，你可以有这个能力，而已

抽象类，是能力的基类，只带有属性，不带有逻辑，我们需要继承这个类，去实现自己的能力

当然，其实这个基类的中的属性，和具体的实现类中的逻辑，是强相关的，也就是说，某个字段对应了什么功能，是约定俗成的（这一点设计上不太好，当然你也可以搞自己的抽象用法。。）

```c#
namespace GameAbilitySystem
{
    public abstract class GameAbility : ScriptableObject
    {
        // 这个技能自身的标签
        public GameTag assetTag;

        // 取消带有这些标签的能力
        // 举例，玩家A有两个技能，一个火魔法，一个冰魔法，当我们释放火魔法时，想要打断自身的所有冰魔法，那么这里就需要加上冰魔法的标签
        public GameTag[] cancelAbilityWithTags;

        // 约定的基本用法：持有这些标签时该能力无法激活
        public GameTag[] blockAbilityWithTags;

        // 约定的基本用法：激活能力时的附带标签
        public GameTag[] activationOwnedTags;

        // 约定的基本用法：当技能持有者不满足标签条件时，无法释放技能
        public GameTagRequireIgnoreContainer ownerTags;

        // 约定的基本用法：当技能释放者不满足标签条件时，无法释放技能，只有通过事件触发技能时（后文说明），会设置这个标签
        public GameTagRequireIgnoreContainer sourceTags;

        // 约定的基本用法：当技能目标不满足标签条件时，无法释放技能，只有通过事件触发技能时（后文说明），会设置这个标签
        public GameTagRequireIgnoreContainer targetTags;

	    // 冷却时间，其实GE不放在这里也行，但是一般来说技能都是有冷却的，所以直接在这里加了一个GE
        public GameEffect coolDown;

        // 技能消耗，其实GE不放在这里也行，但是一般来说技能都是有消耗的，所以直接在这里加了一个GE
        public GameEffect cost;

        // 技能激活时能否再次激活
        public bool canActiveWhenActive = false;
        // 激活时再次激活，是否需要取消自身
        public bool endSelfWhenActiveInActive = false;

        // 技能蓝图，后文说明
        public FlowScript blueprint;

        // 创建一个GAS
        public abstract GameAbilitySpec CreateSpec(AbilitySystemComponent owner,FlowScriptController blueprintController);
    }
}
```

# GameAbilitySpec

可以发现，上面的GA就是一个纯数据类，基本不带有任何逻辑，与GE一样，GA是一个纯配置类，在运行时，我们会主动创建GA对应的GAS

但不一样的地方在于，GAS，同样是一个抽象类

也就是说，我们不但要继承GA实现自己的能力，同时，每一类GA对对应了一个GAS，这个也需要我们继承然后实现，可以说GAS才是真正对能力进行逻辑处理的地方

## AbilityCooldownTime

一个辅助类，用来存储冷却时间的

```c#
namespace GameAbilitySystem
{
	public struct AbilityCooldownTime
    {
        public float timeRemaining;
        public float totalDuration;
    }
}
```

显卡一下GAS的基础属性，和构造方法

```c#
namespace GameAbilitySystem
{
    public abstract class GameAbilitySpec
    {
        // 该GAS对应的GA
        public GameAbility ability;
	    // ACS，技能处理中心，后文细说
        public AbilitySystemComponent owner;
		// 等级
        public float level;
		// 是否激活
        public bool isActive;

        public GameAbilitySpec(GameAbility ability, AbilitySystemComponent owner)
        {
            this.ability = ability;
            this.owner = owner;
        }
    }
}
```

然后我们详细看一下GAS有哪些方法，只有一个入口，那就是尝试激活能力

```c#
namespace GameAbilitySystem
{
    public abstract class GameAbilitySpec
    {
        public void TryActivateAbility()
        {
            if (!CanActivateAbility())
                return;
            ActivateAbility();
        }
    }
}
```

首先是判断能否激活能力

```c#
namespace GameAbilitySystem
{
    public abstract class GameAbilitySpec
    {
        // 有多个条件
        // 1.没有激活，或者激活中但是这个能力可以重复激活
        // 2.符合标签要求
        // 3.符合消耗要求
        // 4.符合冷却要求
        private bool CanActivateAbility()
        {
            return (!isActive || (isActive && ability.canActiveWhenActive))
                   && CheckGameTags()
                   && CheckCost()
                   && CheckCooldown().timeRemaining <= 0;
        }
        
        // 符合标签要求，这是一个抽象方法，需要我们在我们自己的能力里判断
        protected abstract bool CheckGameTags();
        
        // 检查技能消耗
        private bool CheckCost()
        {
            // 这里的cost是一个GE，上面说过了
            if (ability.cost == null)
                return true;
            // 创建一个GES，这里的GES，他的时间类型，只能是即时的（用完就删除），否则直接返回true
            // 注意，这里只是创建，没有应用，具体应用消耗的逻辑，在后文里
            var spec = owner.MakeGameEffectSpec(ability.cost, level);
            if (spec.gameEffect.durationPolicy != EDurationPolicy.Instant)
                return true;
		   // 这里就是遍历这个GES的所有修饰器
            foreach (var modifier in spec.gameEffect.modifiers)
            {
                if (modifier.modifierOperator != EModifierOperator.Add)
                    continue;
                // 然后挨个计算消耗，对比数值，这里也说明，如果你的消耗是法力-40，那么你只能有一个修饰器是法力-40，而不能有两个法力-20
                var cost = modifier.modifierMagnitude.CalculateMagnitude(spec) * modifier.multiplier;
                if (owner.attributeSystemComponent.TryGetAttributeValue(modifier.attribute, out var attributeValue))
                {
                    if (attributeValue.currentValue + cost < 0)
                        return false;
                }
                else
                {
                    return false;
                }
            }

            return true;
        }

        // 检查冷却时间
        public AbilityCooldownTime CheckCooldown()
        {
            if (ability.coolDown == null)
                return new AbilityCooldownTime();
            var maxDuration = 0f;
            var longestCooldown = 0f;
            var coolDownTags = ability.coolDown.grantedTags;

            // 遍历ASC的所有GE，找到所有的CD GE，找到最长的CD时间
            foreach (var appliedGameEffect in owner.appliedGameEffects)
            {
                foreach (var grantedTag in appliedGameEffect.spec.gameEffect.grantedTags)
                {
                    foreach (var coolDownTag in coolDownTags)
                    {
                        if (grantedTag.IsDescendantOf(coolDownTag))
                        {
                            if (appliedGameEffect.spec.gameEffect.durationPolicy == EDurationPolicy.Infinite)
                            {
                                return new AbilityCooldownTime
                                {
                                    timeRemaining = float.MaxValue,
                                    totalDuration = 0
                                };
                            }

                            var durationRemaining = appliedGameEffect.spec.durationRemaining;
                            if (durationRemaining > longestCooldown)
                            {
                                longestCooldown = durationRemaining;
                                maxDuration = appliedGameEffect.spec.totalDuration;
                            }
                        }
                    }
                }
            }

            return new AbilityCooldownTime
            {
                timeRemaining = longestCooldown,
                totalDuration = maxDuration
            };
        }
    }
}
```

我们看一下具体的配置，先看消耗，可以发现，这个GE，基本没有任何要求，也不带什么标签，单纯只是附加了一个修饰器

![image-20231115141544446](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311151415501.png)

在看冷却时间的GE，这个稍微不太一样，这个GE会附加一个标签，这个标签，就是冷却标签

没有其他任何效果，也没有消耗，但他有一个持续时间，当持续时间结束后，这个GE就会被删除

所以很明显，第一次释放技能时，我们身上没有这个冷却标签，自然可以释放，第二次释放时，因为我们身上已经有这个标签了，自然不符合冷却时间要求，就不能释放了

而当时序时间结束后，这个GE就会被删除，标签自然也就没了（标签是附加在GE身上的）

![image-20231115141700285](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311151417333.png)

其次是激活/取消激活的两个方法

```c#
namespace GameAbilitySystem
{
    public abstract class GameAbilitySpec
    {   
        public virtual void ActivateAbility()
        {
            isActive = true;
			// 如果当前技能正在释放中，判断需不需要取消
            var index = owner.currentAbilitySpecs.IndexOf(this);
            if (index > 0)
            {
                var current = owner.currentAbilitySpecs[index];
                if (current.ability.endSelfWhenActiveInActive)
                    current.EndAbility();
                owner.currentAbilitySpecs.Remove(this);
            }
            owner.currentAbilitySpecs.Add(this);
            
            owner.CancelAbilityWithTags(ability.cancelAbilityWithTags);
        }

        public virtual void EndAbility()
        {
            isActive = false;
        }
    }
}
```

很明显，好像整个GAS里也没什么关于技能的逻辑，都是一些最基础的判断

因为真正的技能逻辑需要我们继承GAS去实现

# 实践

千万要记住，GA/GAS，代表的是能力，能力意味着某一类的技能，千万不要每一个技能都写一个GA，大多数技能是可以通过某一个GA配置实现的

以普通攻击为例，我们实现一个最简单的能力

## SimpleAbility

GA其实是没什么东西的，就是返回一个对应的GAS就行

```c#
namespace GASExample
{
    [CreateAssetMenu(menuName = "GASExample/Ability/Simple Ability")]
    public class SimpleAbility : GameAbility
    {
        public override GameAbilitySpec CreateSpec(AbilitySystemComponent owner,FlowScriptController blueprintController)
        {
            var spec = new SimpleAbilitySpec(this, owner, blueprintController);
            return spec;
        }
    }
}
```

## SimpleAbilitySpec

主要来看一下这个GAS，这是个内部类

先看一下属性和构造方法

```c#
namespace GASExample
{
    [CreateAssetMenu(menuName = "GASExample/Ability/Simple Ability")]
    public class SimpleAbility : GameAbility
    {
        public class SimpleAbilitySpec : GameAbilitySpec
        {
            // 蓝图控制器（后文细说）
            private readonly FlowScriptController blueprintController;
            // 玩家控制器（后文细说）
            private readonly PlayerController controller;
            // 对应的GA
            private readonly SimpleAbility simpleAbility;

            // 构造方法就是属性初始化，没什么特殊逻辑
            public SimpleAbilitySpec(GameAbility ability, AbilitySystemComponent owner, FlowScriptController blueprintController) : base(ability, owner)
            {
                simpleAbility = ability as SimpleAbility;
                this.blueprintController = blueprintController;
                controller = owner.GetComponent<PlayerController>();
                level = owner.level;
            }
        }
    }
}
```

然后再看他的一些接口方法

```c#
namespace GASExample
{
    [CreateAssetMenu(menuName = "GASExample/Ability/Simple Ability")]
    public class SimpleAbility : GameAbility
    {
        public class SimpleAbilitySpec : GameAbilitySpec
        {
            // 重写GAS的抽象方法
            // HasAllTags和HasNoTags都是ACS的接口方法，用法和名字一样，就是判断玩家身上有某些标签，或者没有某些标签
            // 这里就是最基础的标签检测，如果玩家要能够释放这个技能，那么必须满足的标签要求
            protected override bool CheckGameTags()
            {
                return owner.HasAllTags(ability.ownerTags.requireTags)
                       && owner.HasNoTags(ability.ownerTags.ignoreTags)
                       && owner.HasAllTags(ability.sourceTags.requireTags)
                       && owner.HasNoTags(ability.sourceTags.ignoreTags)
                       && owner.HasAllTags(ability.targetTags.requireTags)
                       && owner.HasNoTags(ability.targetTags.ignoreTags);
            }

            // 重写激活能力的接口
            public override void ActivateAbility()
            {
                // 必须要先走base
                base.ActivateAbility();
                // 这里才是真正应用消耗和冷却GE的地方
                if (simpleAbility.coolDown)
                {
                    var cdSpec = owner.MakeGameEffectSpec(simpleAbility.coolDown, level);
                    owner.ApplyGameEffectSpecToSelf(cdSpec);
                }

                if (simpleAbility.cost)
                {
                    var costSpec = owner.MakeGameEffectSpec(simpleAbility.cost, level);
                    owner.ApplyGameEffectSpecToSelf(costSpec);
                }
			   // 蓝图控制器派发事件（后文细说）
                blueprintController.SendEvent("OnAbilityStart", (GameAbilitySpec)this, owner.GameObject());
            }

            public override void EndAbility()
            {
                if (!isActive)
                    return;
                base.EndAbility();
            }
        }
    }
}
```

这就是最基础的一个GAS了，其实也不复杂， 就多了两步：

- 检查释放标签要求
- 应用消耗GES，冷却GES
- 派发蓝图事件开始技能流程

最后，提前放一个完整版的蓝图编辑流程在这里：

![蓝图案例](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311151544743.png)









