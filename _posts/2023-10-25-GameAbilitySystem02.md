---
title: "GameAbilitySystem|02 GameAttribute 属性"
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

# AttributeSystemComponent

上面说了很多东西，但始终没有一个”触发中心“，而这个组件，就是整个属性系统的数据中心，负责更新属性，派发事件

先来看看它的基础字段有哪些，其实很简单，只有几部分

- 有哪些属性事件触发器？

- 有哪些属性？这些属性的值是什么？
- 属性的缓存，上一帧的属性缓存

```c#
namespace GameAbilitySystem
{
    /// <summary>
    /// 属性系统
    /// </summary>
    public class AttributeSystemComponent : MonoBehaviour
    {
        [SerializeField] [LabelText("属性事件")] 
        private List<BaseAttributeEventHandler> attributeSystemEvents;

        [SerializeField] [LabelText("属性")] 
        private List<GameAttribute> attributes;

        [SerializeField] [LabelText("属性值")]
        private List<GameAttributeValue> attributeValues;

        /// 标记属性为脏时，会重置属性缓存
        private bool isAttributeDirty = false;
        public readonly Dictionary<GameAttribute, int> attributeCache = new();

        private readonly List<GameAttributeValue> preAttributeValues = new();
    }
}
```

再看生命周期

```c#
namespace GameAbilitySystem
{
    /// <summary>
    /// 属性系统
    /// </summary>
    public class AttributeSystemComponent : MonoBehaviour
    {
        private void Awake()
        {
            InitialiseAttributeValues();
            MarkAttributeDirty();
            GetAttributeCache();
        }
        
        // 初始化属性值，就是给每一个属性都生成了一个对应的属性值，并且把修饰器置为空
        private void InitialiseAttributeValues()
        {
            attributeValues = new List<GameAttributeValue>();
            for (var i = 0; i < attributes.Count; i++)
            {
                attributeValues.Add(new GameAttributeValue()
                    {
                        attribute = attributes[i],
                        modifier = new GameAttributeModifier()
                        {
                            add = 0f,
                            multiply = 0f,
                            overwrite = 0f
                        }
                    }
                );
            }
        }
        
        // 标记属性为脏
        public void MarkAttributeDirty()
        {
            isAttributeDirty = true;
        }
        
        // 缓存属性值，只有当属性标记为脏时，才会重置更新
        private Dictionary<GameAttribute, int> GetAttributeCache()
        {
            if (isAttributeDirty)
            {
                attributeCache.Clear();
                for (var i = 0; i < attributeValues.Count; i++)
                {
                    attributeCache.Add(attributeValues[i].attribute, i);
                }

                isAttributeDirty = false;
            }

            return attributeCache;
        }        

        // 在LateUpdate中更新当前属性值
        private void LateUpdate()
        {
            UpdateAttributeCurrentValues();
        }

        private void UpdateAttributeCurrentValues()
        {
            preAttributeValues.Clear();
            // 遍历所有属性值，先把当前帧的值缓存下来，然后调用虚方法计算当前属性值
            for (var i = 0; i < attributeValues.Count; i++)
            {
                var attr = attributeValues[i];
                preAttributeValues.Add(attr);
                attributeValues[i] = attr.attribute.CalculateCurrentAttributeValue(attr, attributeValues);
            }

            // 派发属性事件
            for (var i = 0; i < attributeSystemEvents.Count; i++)
            {
                attributeSystemEvents[i].PreAttributeChange(this, preAttributeValues, ref attributeValues);
            }
        }
    }
}
```

大致效果是这样的

![image-20231113171846075](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131718118.png)

一些重要的接口方法

```c#
namespace GameAbilitySystem
{
    public class AttributeSystemComponent : MonoBehaviour
    {
        // 更新某个属性的修饰器
        public void UpdateAttributeModifier(GameAttribute attr, GameAttributeModifier modifier)
        {
            var cache = GetAttributeCache();
            if (cache.TryGetValue(attr, out var index))
            {
                var attrValue = attributeValues[index];
                attrValue.modifier = attrValue.modifier.Combine(modifier);
                attributeValues[index] = attrValue;
            }
        }
        
        // 重置所有属性的修饰器
        public void ResetAttributeModifiers()
        {
            for (var i = 0; i < attributeValues.Count; i++)
            {
                var attributeValue = attributeValues[i];
                attributeValue.modifier = default;
                attributeValues[i] = attributeValue;
            }
        }
    }
}
```



# 更复杂的属性

回到最初的GameAttribute，你会发现，**CalculateCurrentAttributeValue**是一个虚方法，所以你是可以重写计算数值的逻辑的，这里可以高度自定义

比如，我们可以简单实现一个二级属性的逻辑

> 二级属性？比如：生命值=100*力量，这里的力量就是一级属性，生命值就是二级属性，100则代表一个规格（后文详说）

```c#
namespace GameAbilitySystem
{
    /// <summary>
    /// 二级属性，比如角色的生命值是基于他的力量的，那么生命值就是一个二级属性二级属性，需要有他的一级属性，和计算规则
    /// </summary>
    [CreateAssetMenu(menuName = "GameAbilitySystem/Linear Derived Attribute")]
    public class LinearDerivedGameAttribute : GameAttribute
    {
        [LabelText("目标属性")]
        public GameAttribute Attribute;
        [LabelText("倍率")]
        [SerializeField] private float gradient;
        [LabelText("偏移")]
        [SerializeField] private float offset;

        public override GameAttributeValue CalculateCurrentAttributeValue(GameAttributeValue gameAttributeValue, List<GameAttributeValue> allAttributeValues)
        {
            // 找到目标一级属性
            var baseAttributeValue = allAttributeValues.Find(x => x.attribute == Attribute);

            // 基础值=一级属性*倍率+偏移
            gameAttributeValue.baseValue = (baseAttributeValue.currentValue * gradient) + offset;

            // 当前值
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

那可以有这些二级属性

![image-20231113172817783](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131728819.png)

![image-20231113172825144](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131728176.png)











