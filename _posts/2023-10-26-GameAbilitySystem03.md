---
title: "GameAbilitySystem|03 Magnitude 规格"
categories: GAS
tags: 战斗
---

规格，这个翻译很怪，但我也想不到什么更好的词语

> 生命周期=100*力量，这里的100，就是规格

规格代表一种尺度，是对数值的一种缩放，或者说数值公式

# BaseMagnitude

规格的抽象基类，只有两个接口（GameEffectSpec，简称GES，游戏效果执行器，后文细说）

- 初始化
- 计算数值

```c#
namespace GameAbilitySystem
{
    /// <summary>
    /// 规格，根据需求返回一个特定的数值
    /// </summary>
    public abstract class BaseMagnitude : ScriptableObject
    {
        public abstract void Initialise(GameEffectSpec spec);
        public abstract float CalculateMagnitude(GameEffectSpec spec);
    }
}
```

# SimpleFloatMagnitude

最简单的一种数值规格，根据GES中的等级，返回一个曲线中的数值

```c#
namespace GameAbilitySystem
{
    /// <summary>
    /// 简单的数值规格，可以通过曲线编辑
    /// </summary>
    [CreateAssetMenu(menuName = "GameAbilitySystem/Magnitude/Simple Float")]
    public class SimpleFloatMagnitude : BaseMagnitude
    {
        [Title("说明")]
        [InfoBox("最基础的规格器\nCurve的采样目标是所属GES的Level")]
        [SerializeField]
        [LabelText("采样曲线"), LabelWidth(50)]
        private AnimationCurve animationCurve;
        
        public override void Initialise(GameEffectSpec spec)
        {
        }

        public override float CalculateMagnitude(GameEffectSpec spec)
        {
            return animationCurve.Evaluate(spec.level);
        }
    }
}
```

比如这样一个Float1的规格器，不管你的等级是多少，他都返回1，也就是说，不进行数值计算

![image-20231113174315251](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131743283.png)

![image-20231113174324846](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131743905.png)

# AttributeBackedMagnitude

当然规格器同样能够做的很复杂，比如依据属性进行数值计算

```c#
namespace GameAbilitySystem
{
    // 从谁身上进行属性捕获？自身，还是目标？
    public enum ECaptureAttributeFrom
    {
        Source,
        Target
    }

    // 什么时候进行属性捕获？创建时，还是应用时？
    public enum ECaptureAttributeWhen
    {
        OnCreate,
        OnApplication
    }

    /// <summary>
    /// 基于属性的数值规格，可以捕获目标属性
    /// </summary>
    [CreateAssetMenu(menuName = "GameAbilitySystem/Magnitude/Attribute Backed")]
    public class AttributeBackedMagnitude : BaseMagnitude
    {
        // 目标属性
        private GameAttribute capturedAttribute;

        // 捕获来源
        private ECaptureAttributeFrom captureAttributeFrom;

        // 捕获时机
        private ECaptureAttributeWhen captureAttributeWhen;

        // 采样曲线
        private AnimationCurve animationCurve;

        // 这里的逻辑涉及到后文的GES，不需要多考虑，只需要知道，我们会根据捕获的来源，以及目标属性，获得一个属性值，可能是来自于自身的，也可能来自于目标对象
        public override void Initialise(GameEffectSpec spec)
        {
            switch (captureAttributeFrom)
            {
                case ECaptureAttributeFrom.Source:
                    spec.source.attributeSystemComponent.TryGetAttributeValue(capturedAttribute,
                        out var capturedAttributeValue);
                    spec.sourceCapturedAttributeValue = capturedAttributeValue;
                    break;
                case ECaptureAttributeFrom.Target:
                    spec.target.attributeSystemComponent.TryGetAttributeValue(capturedAttribute,
                        out capturedAttributeValue);
                    spec.targetCapturedAttributeValue = capturedAttributeValue;
                    break;
            }
        }

        // 这里本质上还是通过曲线采样一个数值，具体采样那个值呢？这需要根据我们的配置计算
        public override float CalculateMagnitude(GameEffectSpec spec)
        {
            return animationCurve.Evaluate(GetCapturedAttribute(spec).currentValue);
        }

        private GameAttributeValue GetCapturedAttribute(GameEffectSpec spec)
        {
            // 如果是在创建时捕获属性，那么直接根据捕获目标，返回捕获属性就行，这些值在创建时已经赋值过了
            if (captureAttributeWhen == ECaptureAttributeWhen.OnCreate)
            {
                switch (captureAttributeFrom)
                {
                    case ECaptureAttributeFrom.Source:
                        return spec.sourceCapturedAttributeValue;
                    case ECaptureAttributeFrom.Target:
                        return spec.targetCapturedAttributeValue;
                }
            }

            // 否则的话就是应用时捕获，那么其实就是在走一遍Initialise的流程，返回目标对象当前的目标属性值
            switch (captureAttributeFrom)
            {
                case ECaptureAttributeFrom.Source:
                    spec.source.attributeSystemComponent.TryGetAttributeValue(capturedAttribute,
                        out var sourceAttributeValue);
                    return sourceAttributeValue;
                case ECaptureAttributeFrom.Target:
                    spec.target.attributeSystemComponent.TryGetAttributeValue(capturedAttribute,
                        out var targetAttributeValue);
                    return targetAttributeValue;
            }

            return default;
        }
    }
}
```

然后就可以得到这样一个规格器，会在应用时，去捕获自身的生命恢复速度

![image-20231113175758852](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131757886.png)