---
title: "GameAbilitySystem|04 GameEffect 游戏效果"
categories: GAS
tags: 战斗
---

所谓游戏效果（GE），就是技能所附带的功能，技能本身不进行过多的逻辑判断，而是创建一个个的游戏效果，具体的逻辑应该交由游戏效果

# GameEffect

## 标签

### GameTagRequireIgnoreContainer

这就是一个容器，包含2个GameTag数组，自身不带逻辑，是交由后面的处理类进行使用的

```c#
namespace GameAbilitySystem
{
    [Serializable]
    public struct GameTagRequireIgnoreContainer
    {
        // 一般这里指需要包含的Tag
        public GameTag[] requireTags;
        // 一般这里指禁止包含的Tag
        public GameTag[] ignoreTags;
    }
}
```

然后我们回到GE中来，GE中包含了很多的Tag

```c#
namespace GameAbilitySystem
{
    public class GameEffect : ScriptableObject
    {
        // 这个GE会附带的标签，也就是说，如果你应用了这个游戏效果，那么这些标签也会附加到你人身上
        public GameTag[] grantedTags;

        // 移除任何带有这些标签的游戏效果，一个GE能够移除其他GE，当然很合理
        public GameTag[] removeGameEffectsWithTag;

        // 表示效果进行中的标签要求
        // 这个效果在满足标签要求时会被认为是进行中的，否则会被认为是暂停的，如果是暂停的，那么这个效果带来的任何修饰都会被移除，直到恢复，期间仍然会进行倒计时
        public GameTagRequireIgnoreContainer onGoingTagRequirements;

        // 能否应用效果的标签要求
        // 这个效果只会在满足标签要求时被应用，包含所有必须标签，且不包含任何禁止标签
        public GameTagRequireIgnoreContainer applicationTagRequirements;

        // 是否移除效果的标签条件
        // 这个效果会在不满足标签要求时被移除，没有包含所有必须标签，或者包含了任何禁止的标签
        public GameTagRequireIgnoreContainer removalTagRequirements;
    }
}
```

最后标签的面板如下

![image-20231114142706819](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311141427859.png)

## 生命周期

### EDurationPolicy

GE的生命周期，有三种，很好理解

```c#
public enum EDurationPolicy
{
    [LabelText("即时")] Instant,
    [LabelText("永久")] Infinite,
    [LabelText("持续时间")] HasDuration
}
```

```c#
namespace GameAbilitySystem
{
    public class GameEffect : ScriptableObject
    {
	    // 周期类型
        public EDurationPolicy durationPolicy;

        // 执行周期，每过多少秒执行一次
        [HideIf("@this.durationPolicy == EDurationPolicy.Instant")]
        public float period;

        // 是否立刻执行，第一次执行是立刻执行，还是等待一个周期再执行
        [HideIf("@this.durationPolicy == EDurationPolicy.Instant")]
        public bool executeImmediate;

        // 持续时间，是一个规格器，当然，只有这个GE是持续时间类型的时候，才需要选择
        [HideIf("@this.durationPolicy != EDurationPolicy.HasDuration")]
        public BaseMagnitude durationMagnitude;

        // 持续时间缩放，就是在规格器的基础上再进行一个缩放
        [HideIf("@this.durationPolicy != EDurationPolicy.HasDuration")]
        public float durationMultiplier;
    }       
}
```

编辑器面板如下

即时，只会生效一次，所以没什么配置

![image-20231114142827545](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311141428570.png)

永久，那需要知道多少秒生效一次，并且是否立刻生效

![image-20231114142856367](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311141428392.png)

周期，最常见的用法

![image-20231114142940683](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311141429710.png)

## 修饰器

对属性进行修改的东西

### EModifierOperator

操作类型

```c#
namespace GameAbilitySystem
{
    public enum EModifierOperator
    {
        [LabelText("加")]
        Add, 
        [LabelText("乘")]
        Multiply,
        [LabelText("覆写")]
        Override
    }
}
```

### GameEffectModifier

```c#
namespace GameAbilitySystem
{   
    [Serializable]
    public struct GameEffectModifier
    {
        // 目标属性
        public GameAttribute attribute;
        // 操作类型
        public EModifierOperator modifierOperator;
        // 数值规格器
        public BaseMagnitude modifierMagnitude;
        // 数值缩放
        public float multiplier;
    }
}
```

最后回到GE里就很简单了

```c#
namespace GameAbilitySystem
{
    public class GameEffect : ScriptableObject
    {
        // 修饰器数组
        public GameEffectModifier[] modifiers;
    }
}
```

很明显，一个GE可以对多个属性进行修改

最后举一个例子，比如一个常见的初始化GE，不附带任何标签，也没有任何使用条件，但是只应用一次，效果就是修改各种属性（初始化）

![image-20231114143437412](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311141434472.png)

# GameEffectSpec

GameEffect，简称GE

GameEffectSpec，简称GES

这两个的关系，不是很好解释，类似于Class和实例的区别，GE是一个配置项，而真正在运行时创建的GES，是基于某一个配置项的实例（后续中的GA与GAS也是一样的关系）

好处就是，让GE彻底成为一个可配置项，非常灵活，非常自由，完全不需要关心运行时会发生什么

先看一下GES有哪些字段

```c#
namespace GameAbilitySystem
{
    public class GameEffectSpec
    {
        // 这两个属性快照，是在规格器那一章提到过的，会在特定的时机对目标属性值进行快照
        public GameAttributeValue sourceCapturedAttributeValue;
        public GameAttributeValue targetCapturedAttributeValue;
        
        // AbilitySystemComponent，简称ACS，是技能核心组件，每个角色身上都会挂载，具体内容后面会说
        // source就是释放者，target就是目标，很好理解
        public AbilitySystemComponent source;
        public AbilitySystemComponent target;
        
        // 这个GES对应的GE
        public GameEffect gameEffect;

        // 这里就是一些基础属性了，等级，剩余时间，总时间
        public float level;
        public float durationRemaining;
        public float totalDuration;
        public float timeUntilPeriodTick;
    }
}
```

构造函数，很简单，就是对几个属性进行初始化

```c#
namespace GameAbilitySystem
{
    public class GameEffectSpec
    {
        private GameEffectSpec(GameEffect gameEffect, AbilitySystemComponent source, float level)
        {
            // 保存3个基础属性
            this.gameEffect = gameEffect;
            this.source = source;
            this.level = level;

            // 初始化所有的GE修饰器
            foreach (var modifier in this.gameEffect.modifiers)
            {
                modifier.modifierMagnitude.Initialise(this);
            }

            // 持续时间，也是一个规格器，当然这里是可能没有的（即时，或者永久类型）
            var durationMagnitude = this.gameEffect.durationMagnitude;
            if (durationMagnitude != null)
            {
                // 总时间和剩余时间一致
                durationRemaining = durationMagnitude.CalculateMagnitude(this) * this.gameEffect.durationMultiplier;
                totalDuration = durationRemaining;
            }

            // 这里是执行周期
            timeUntilPeriodTick = this.gameEffect.period;
            // 这里就是前面说的，是否立刻执行GE，还是等待一个周期
            if (this.gameEffect.executeImmediate)
                timeUntilPeriodTick = 0;
        }
    }
}
```

然后是两个生命周期函数，用于Update的

```c#
namespace GameAbilitySystem
{
    public class GameEffectSpec
    {
        // 更新剩余时间，如果是永久GE，那么剩余时间一致会是1
        public void UpdateRemainingDuration(float deltaTime)
        {
            if (gameEffect.durationPolicy == EDurationPolicy.Infinite)
                durationRemaining = 1;
            else
                durationRemaining -= deltaTime;
        }

        // 更新周期，这里没有具体的业务逻辑，只是返回一个bool告诉你，是否要执行GE
        public void TickPeriod(float deltaTime, out bool executePeriodicTick)
        {
            executePeriodicTick = false;
            timeUntilPeriodTick -= deltaTime;
            if (timeUntilPeriodTick <= 0)
            {
                timeUntilPeriodTick = gameEffect.period;
                if (gameEffect.period > 0)
                    executePeriodicTick = true;
            }
        }
    }
}
```

最后是一个创建GES的工具方法，没什么特殊用法，就是一个纯粹的工具方法

PS：上面的构造方法是private的哦，所以只能通过这个工具方法创建GES

```c#
namespace GameAbilitySystem
{
    public class GameEffectSpec
    {
        public static GameEffectSpec CreateNew(GameEffect gameEffect, AbilitySystemComponent source, float Level = 1)
        {
            return new GameEffectSpec(gameEffect, source, Level);
        }
    }
}
```



















