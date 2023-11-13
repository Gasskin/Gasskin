---
title: "GameAbilitySystem|01 GameAttribute 属性"
tags: 战斗
---

# GameAttribute

![image-20231113164250605](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131642631.png)

属性标签，仅仅是一个标签，没有具体的数值

```c#
namespace GameAbilitySystem
{
    /// <summary>
    /// 不包含任何数据，仅仅是一个标记
    /// </summary>
    [CreateAssetMenu(menuName = "GameAbilitySystem/Attribute")]
    public class GameAttribute : ScriptableObject
    {
        public string name;

        /// 用于计算当前这个一级属性的具体数值， 这个类本身只是一种标签，具体的数值，肯定是要结合具体情况计算的，而具体的情况，则是放入GameAttributeValue这个类中
        public virtual GameAttributeValue CalculateCurrentAttributeValue(GameAttributeValue gameAttributeValue, List<GameAttributeValue> allAttributeValues)
        {
            gameAttributeValue.currentValue = (gameAttributeValue.baseValue + gameAttributeValue.modifier.add) * (gameAttributeValue.modifier.multiply + 1);

            if (gameAttributeValue.modifier.overwrite != 0)
            {
                gameAttributeValue.currentValue = gameAttributeValue.modifier.overwrite;
            }

            return gameAttributeValue;
        }
    }
}
```

具体就可以有这样一些属性

![image-20231113164313190](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131643213.png)

你可以看到，他确实只是一个标签，代表玩家有这样一个属性

![image-20231113164338447](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131643476.png)

# GameAttributeValue

属性值，真正存储属性数值的地方，当然，你看到了，这仅仅是一个数值容器，他本身不会去计算数值，而是反过来，我们会有一个类去计算数值，然后把数值存到这儿

```c#
namespace GameAbilitySystem
{
    /// <summary>
    /// 属性的数值，有属性标记+数值组成+修改器
    /// </summary>
    [Serializable]
    public struct GameAttributeValue
    {
        [LabelText("目标属性")] public GameAttribute attribute;
        [LabelText("基础值")] public float baseValue;
        [LabelText("当前值")] public float currentValue;

        [LabelText("修饰器")]
        public GameAttributeModifier modifier;
    }
}
```

## GameAttributeModifier

属性修饰器，当我们的属性增减时（因为有其他BUFF啊或者技能啥的），我们不会直接去修改属性，那可扩展性就太差了，我们会去修改属性的修饰器，最终数值=基础数值+修饰器修改

```c#
namespace GameAbilitySystem
{
    /// <summary>
    /// 属性的修改器，有叠加，乘加，覆写
    /// </summary>
    [Serializable]
    public struct GameAttributeModifier
    {
        [LabelText("叠加")]
        public float add;

        [LabelText("乘加")]
        public float multiply;

        [LabelText("覆写")]
        public float overwrite;
		
        /// 这里是用于多个属性修饰器叠加
        public GameAttributeModifier Combine(GameAttributeModifier other)
        {
            other.add += add;
            other.multiply += multiply;
            other.overwrite = overwrite;
            return other;
        }
    }
}
```

# EventHandler

当属性发生改变前，会进行事件派发

```c#
namespace GameAbilitySystem
{
    /// <summary>
    /// 属性事件处理器，会在属性改变前被调用，可以继承这个类实现自己的属性事件处理器
    /// </summary>
    public abstract class BaseAttributeEventHandler : ScriptableObject
    {
        public abstract void PreAttributeChange(AttributeSystemComponent attributeSystem, List<GameAttributeValue> prevAttributeValues, ref List<GameAttributeValue> currentAttributeValues);
    }
}
```

这是一个抽象类，所以具体的逻辑实现，需要你去继承实现

举个例子，但属性值发生改变时，我们需要Log一下，那就可以实现这样一个属性改变事件器

```c#
namespace GameAbilitySystem
{
    [CreateAssetMenu(menuName = "GameAbilitySystem/Attribute EventHandler/Log Attribute Change")]
    public class LogAttributeChangeEventHandler : BaseAttributeEventHandler
    {
        [SerializeField]
        [LabelText("目标属性")]
        private GameAttribute primaryAttribute;
        
        public override void PreAttributeChange(AttributeSystemComponent attributeSystem, List<GameAttributeValue> prevAttributeValues, ref List<GameAttributeValue> currentAttributeValues)
        {
            // 属性系统，下文细说，总之所有属性都会存放到属性系统中
            var attributeCacheDict = attributeSystem.attributeCache;
            // 这里就是找到了我们的目标属性
            if (attributeCacheDict.TryGetValue(primaryAttribute, out var primaryAttributeIndex))
            {
                // 拿到上一帧的值与当前帧的值，对比，如果有差异，就Log出来
                var prevValue = prevAttributeValues[primaryAttributeIndex].currentValue;
                var currentValue = currentAttributeValues[primaryAttributeIndex].currentValue;

                if (Math.Abs(prevValue - currentValue) > 0.0001f) 
                {
                    Debug.Log($"【{attributeSystem.gameObject.name}】 【{currentAttributeValues[primaryAttributeIndex].attribute.name}】 {prevValue} >>> {currentValue}");
                }
            }
        }
    }
}
```

你可以发现，这个事件触发器，并不是针对某一个属性的，而是针对某一类行为的，比如上面这个，我们可以Log血量改变，也可以Log蓝量改变，这取决于我们的目标属性是什么

![image-20231113165304453](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131653477.png)

![image-20231113165314791](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131653818.png)

这个事件触发器针对的就是生命值改变

