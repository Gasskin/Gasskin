---
title: "GameAbilitySystem|06 AbilitySystemComponent"
tags: 战斗
---

GAS的核心系统，负责所有GAS、GES的推动

# 两个容器

简单来说就是针对每一个GE，都有一个修饰器容器

```c#
namespace GameAbilitySystem
{
    public class ModifierContainer
    {
        public GameAttribute attribute;
        public GameAttributeModifier modifier;
    }

    public class GameEffectContainer
    {
        public GameEffectSpec spec;
        public List<ModifierContainer> modifiers;
    }
}
```

# 基本属性

比较简单

```c#
namespace GameAbilitySystem
{
    public class AbilitySystemComponent : MonoBehaviour
    {
        // 对属性系统的引用
        public AttributeSystemComponent attributeSystemComponent;
        // 所有的GES容器
        public List<GameEffectContainer> appliedGameEffects = new();
        // 所有的能力，能力需要Add到这里，Add的同时会创建对应的GAS
        public Dictionary<GameAbility, GameAbilitySpec> grantedAbilities = new();
        // 当前应用中的GAS
        public List<GameAbilitySpec> currentAbilitySpecs = new();
        // 玩家等级
        public float level;
        // 这是用于蓝图系统的，和ASC本身没有太多关联（后文细说）
        private Transform abilityBlueprintRoot;
    }
}
```

# 生命周期

主要就两个

```c#
namespace GameAbilitySystem
{
    public class AbilitySystemComponent : MonoBehaviour
    {
        private void Awake()
        {
            abilityBlueprintRoot = new GameObject("_ability_blueprint_root").transform;
            abilityBlueprintRoot.SetParent(transform, false);
        }

        private void Update()
        {
            attributeSystemComponent.ResetAttributeModifiers();
            UpdateAttributeSystem();
            TickGameEffects();
            CleanGameEffects();
        }
    }
}
```

## Awake

只是创建了一个蓝图节点，放到自己下面

```c#
namespace GameAbilitySystem
{
    public class AbilitySystemComponent : MonoBehaviour
    {
        private void Awake()
        {
            abilityBlueprintRoot = new GameObject("_ability_blueprint_root").transform;
            abilityBlueprintRoot.SetParent(transform, false);
        }
    }
}
```

## Update

```c#
namespace GameAbilitySystem
{
    public class AbilitySystemComponent : MonoBehaviour
    {
        private void Update()
        {
            // 重置所有属性修饰器
            attributeSystemComponent.ResetAttributeModifiers();
            UpdateAttributeSystem();
            TickGameEffects();
        }
    }
}
```

### UpdateAttributeSystem

更新所有属性

```c#
private void UpdateAttributeSystem()
{
    foreach (var appliedGameEffect in appliedGameEffects)
    {
        foreach (var modifier in appliedGameEffect.modifiers)
        {
            attributeSystemComponent.UpdateAttributeModifier(modifier.attribute, modifier.modifier);
        }
    }
}
```

### TickGameEffects

更新GE

```c#
private void TickGameEffects()
{
    foreach (var appliedGameEffect in appliedGameEffects)
    {
        // 如果是即时的GE，忽略，因为已经应用过了
        if (appliedGameEffect.spec.gameEffect.durationPolicy == EDurationPolicy.Instant)
            continue;
        // 更新生命周期
        appliedGameEffect.spec.UpdateRemainingDuration(Time.deltaTime);
        // 更新应用周期
        appliedGameEffect.spec.TickPeriod(Time.deltaTime, out var execute);
        // 这里其实就是判断，这个GE是否需要应用
        var onGoingTags = appliedGameEffect.spec.gameEffect.onGoingTagRequirements;
        if (execute && HasAllTags(onGoingTags.requireTags) && HasNoTags(onGoingTags.ignoreTags))
        {
            // 如果需要应用，那遍历GE所有的修饰器
            foreach (var modifier in appliedGameEffect.spec.gameEffect.modifiers)
            {
                // 添加规格器到这个GE的修饰器中，如果是周期性GE，那每次应用都会添加一个修饰器
                var magnitude = modifier.modifierMagnitude.CalculateMagnitude(appliedGameEffect.spec) * modifier.multiplier;
                var attributeModifier = new GameAttributeModifier();
                switch (modifier.modifierOperator)
                {
                    case EModifierOperator.Add:
                        attributeModifier.add = magnitude;
                        break;
                    case EModifierOperator.Multiply:
                        attributeModifier.multiply = magnitude;
                        break;
                    case EModifierOperator.Override:
                        attributeModifier.overwrite = magnitude;
                        break;
                }

                appliedGameEffect.modifiers.Add(new ModifierContainer()
                {
                    attribute = modifier.attribute,
                    modifier = attributeModifier
                });
            }

            // 按要求移除其他GE
            RemoveEffectsWithTags(appliedGameEffect.spec.gameEffect.removeGameEffectsWithTag);
        }
    }

    // 清除失效的GE
    CleanGameEffects();
}
```

# 接口方法

## 创建GE/应用GE

```c#
// 创建，单纯只是调用了GES的工具接口
public GameEffectSpec MakeGameEffectSpec(GameEffect gameEffect, float level = 1f)
{
    return GameEffectSpec.CreateNew(gameEffect, this, level);
}

// 应用
public void ApplyGameEffectSpecToSelf(GameEffectSpec spec)
{
    if (spec == null)
        return;
    // 先判断标签条件
    if (!HasAllTags(spec.gameEffect.applicationTagRequirements.requireTags) ||
        !HasNoTags(spec.gameEffect.applicationTagRequirements.ignoreTags))
        return;
    // 然后按照配置应用
    switch (spec.gameEffect.durationPolicy)
    {
        case EDurationPolicy.Infinite:
        case EDurationPolicy.HasDuration:
            ApplyDurationGameEffect(spec);
            break;
        case EDurationPolicy.Instant:
            ApplyInstantGameEffect(spec);
            break;
    }
}
```

### ApplyDurationGameEffect

持续的GE，直接添加到容器里就行了（包括永久持续的）

```c#
private void ApplyDurationGameEffect(GameEffectSpec spec)
{
    appliedGameEffects.Add(new GameEffectContainer
    {
        spec = spec,
        modifiers = new List<ModifierContainer>()
    });
}
```

### ApplyInstantGameEffect

即时的GE，直接应用，即时的GE不会保存到容器里，而是直接修改目标属性的基础值

```c#
private void ApplyInstantGameEffect(GameEffectSpec spec)
{
    foreach (var modifier in spec.gameEffect.modifiers)
    {
        if (attributeSystemComponent.TryGetAttributeValue(modifier.attribute, out var attributeValue))
        {
            var value = modifier.modifierMagnitude.CalculateMagnitude(spec) * modifier.multiplier;
            switch (modifier.modifierOperator)
            {
                case EModifierOperator.Add:
                    attributeValue.baseValue += value;
                    break;
                case EModifierOperator.Multiply:
                    attributeValue.baseValue *= value;
                    break;
                case EModifierOperator.Override:
                    attributeValue.baseValue = value;
                    break;
            }

            attributeSystemComponent.SetAttributeBaseValue(modifier.attribute, attributeValue.baseValue);
        }
    }

    RemoveEffectsWithTags(spec.gameEffect.removeGameEffectsWithTag);
    CleanGameEffects();
}
```

## HasAllTag/HasNoTag

就是简单判断一下，是否具有所有的Tag，或者不包含任何Tag

```c#
public bool HasAllTags(GameTag[] tags)
{
    if (tags == null)
        return true;

    foreach (var tag in tags)
    {
        var flag = false;
        foreach (var appliedGameEffect in appliedGameEffects)
        {
            foreach (var grantedTag in appliedGameEffect.spec.gameEffect.grantedTags)
            {
                if (grantedTag.IsDescendantOf(tag))
                    flag = true;
            }
        }

        if (!flag)
            return false;
    }

    return true;
}

public bool HasNoTags(GameTag[] tags)
{
    if (tags == null || tags.Length == 0)
        return true;

    foreach (var tag in tags)
    {
        var flag = true;
        foreach (var appliedGameEffect in appliedGameEffects)
        {
            foreach (var grantedTag in appliedGameEffect.spec.gameEffect.grantedTags)
            {
                if (grantedTag.IsDescendantOf(tag))
                    flag = false;
            }
        }

        if (!flag)
            return false;
    }

    return true;
}
```

# 添加能力

会涉及到一些蓝图的代码

```c#
public void AddAbility(GameAbility gameAbility)
{
    if (!grantedAbilities.ContainsKey(gameAbility))
    {
        var blueprint = new GameObject($"{gameAbility}");
        var controller = blueprint.AddComponent<FlowScriptController>();
        controller.graph = gameAbility.blueprint;
        blueprint.transform.SetParent(abilityBlueprintRoot, false);
        controller.StartBehaviour();
        
        grantedAbilities.Add(gameAbility, gameAbility.CreateSpec(this, controller));
    }
}
```

# 激活能力

调用GAS里的激活接口

```c#
public void ActiveAbility(GameAbility gameAbility)
{
    if (grantedAbilities.TryGetValue(gameAbility, out var spec))
    {
        spec.TryActivateAbility();
    }
}
```

# 取消能力

```c#
public void CancelAbilityWithTags(GameTag[] gameTags)
{
    if (gameTags == null || gameTags.Length <= 0)
        return;

    foreach (var abilitySpec in currentAbilitySpecs)
    {
        foreach (var gameTag in gameTags)
        {
            if (abilitySpec.ability.assetTag && abilitySpec.ability.assetTag.IsDescendantOf(gameTag))
            {
                abilitySpec.EndAbility();
                break;
            }
        }
    }

    currentAbilitySpecs.RemoveAll(x => !x.isActive);
}
```





