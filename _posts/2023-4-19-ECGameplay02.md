---
title: "ECGameplay|02 数值系统"
categories: ECGameplay
tags: 战斗
---

# 概览

以攻击力为例，在大多数游戏中，无非是装备加成，以及BUFF（状态）加成两种

> 第一层攻击 = 初始攻击
>
> 第二层攻击 = （第一层攻击+装备固定加成）*（1+装备百分比加成）
>
> 第三层攻击 = （第二层攻击+BUFF固定加成）*（1+BUFF百分比加成）

# 数值抽象

不难抽象出一个数值类

## FloatNumeric

```c#
namespace ECGameplay
{
    public class FloatNumeric : Entity
    {
        // 最终数值
        public float Value { get; private set; }
        // 基础数值
        public float baseValue { get; private set; }
        // 装备固定加成
        public float add { get; private set; }
        // 装备百分比加成
        public float pctAdd { get; private set; }
        // BUFF固定加成
        public float finalAdd { get; private set; }
        // BUFF百分比加成
        public float finalPctAdd { get; private set; }
    }
}
```

我们可以直接增减add、pctAdd等，来间接控制最终数值

但是有一个问题，比如我们的装备A给我们增加了100固定攻击，装备B给我们增加了50固定攻击

那么我们的add会是150

但是，如果此时我希望暂时屏蔽装备2带来的加成呢？这时候，你会发现，我们无法区分add中的具体成分，即这部分攻击具体来自于哪儿

```c#
namespace ECGameplay
{
	public class FloatNumeric
    {
        public float Value { get; private set; }
        public float baseValue { get; private set; }

        private FloatModifierCollection AddCollection { get; } = new FloatModifierCollection();
        private FloatModifierCollection PctAddCollection { get; } = new FloatModifierCollection();
        private FloatModifierCollection FinalAddCollection { get; } = new FloatModifierCollection();
        private FloatModifierCollection FinalPctAddCollection { get; } = new FloatModifierCollection();

        public void SetBaseValue(float baseValue)
        {
            this.baseValue = baseValue;
            Refresh();
        }
        
        public void AddAddModifier(FloatModifier modifier)
        {
            AddCollection.AddModifier(modifier);
            Refresh();
        }

        public void AddPctAddModifier(FloatModifier modifier)
        {
            PctAddCollection.AddModifier(modifier);
            Refresh();
        }

        public void AddFinalAddModifier(FloatModifier modifier)
        {
            FinalAddCollection.AddModifier(modifier);
            Refresh();
        }

        public void AddFinalPctAddModifier(FloatModifier modifier)
        {
            FinalPctAddCollection.AddModifier(modifier);
            Refresh();
        }

        public void RemoveAddModifier(FloatModifier modifier)
        {
            AddCollection.RemoveModifier(modifier);
            Refresh();
        }

        public void RemovePctAddModifier(FloatModifier modifier)
        {
            PctAddCollection.RemoveModifier(modifier);
            Refresh();
        }

        public void RemoveFinalAddModifier(FloatModifier modifier)
        {
            FinalAddCollection.RemoveModifier(modifier);
            Refresh();
        }

        public void RemoveFinalPctAddModifier(FloatModifier modifier)
        {
            FinalPctAddCollection.RemoveModifier(modifier);
            Refresh();
        }

        private void Refresh()
        {
            var value1 = baseValue;
            var value2 = (value1 + AddCollection.TotalValue) * (100 + PctAddCollection.TotalValue) / 100f;
            var value3 = (value2 + FinalAddCollection.TotalValue) * (100 + FinalPctAddCollection.TotalValue) / 100f;
            Value = value3;
        }
    }
}
```



## FloatModifier

一个数值修饰器，自身没啥用

```c#
namespace ECGameplay
{
    public class FloatModifier
    {
        // Id，用于区分不同的修饰器
        public int Id;
        // 修饰值
        public float Value;
    }   
}
```

## FloatModifierCollection

修饰器收集器，整合所有的数值修饰器

```c#
namespace ECGameplay
{
    public class FloatModifierCollection
    {
        public float TotalValue { get; private set; }
        // 所有的修饰器	
        private List<FloatModifier> Modifiers { get; } = new List<FloatModifier>();

        public void AddModifier(FloatModifier modifier)
        {
            Modifiers.Add(modifier);
            Refresh();
        }

        public void RemoveModifier(FloatModifier modifier)
        {
            Modifiers.Remove(modifier);
            Refresh();
        }

        private void Refresh()
        {
            TotalValue = 0;
            foreach (var item in Modifiers)
            {
                TotalValue += item.Value;
            }
        }
    }
}
```

# AttributeComponent

```c#
namespace ECGameplay
{
    public class AttributeComponent : Component
    {
        // 移动速度
        public FloatNumeric MoveSpeed { get; set; } = new (); 
        // 当前生命值
        public FloatNumeric HealthPoint  { get; set; } = new (); 
        // 生命值上限
        public FloatNumeric HealthPointMax { get; set; } = new ();
        // 攻击力
        public FloatNumeric Attack { get; set; } = new (); 
        // 防御力（护甲）
        public FloatNumeric Defense { get; set; } = new (); 
        // 法术强度
        public FloatNumeric AbilityPower { get; set; } = new (); 
        // 魔法抗性
        public FloatNumeric SpellResistance { get; set; } = new (); 
        // 暴击概率
        public FloatNumeric CriticalProbability { get; set; } = new (); 
        // 暴击伤害
        public FloatNumeric CauseDamage { get; set; } = new ();


        public override void Awake()
        {
            MoveSpeed.SetBaseValue(1);
            HealthPoint.SetBaseValue(99_999);
            HealthPointMax.SetBaseValue(99_999);
            Attack.SetBaseValue(1000);
            Defense.SetBaseValue(300);
            AbilityPower.SetBaseValue(0);
            SpellResistance.SetBaseValue(300);
            CriticalProbability.SetBaseValue(0.5f);
            CauseDamage.SetBaseValue(1.5f);
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
            Debug.Log(GetComponent<AttributeComponent>()?.HealthPoint.Value);
        }
    }
    
    private void Start()
    {
        var entity = MasterEntity.AddChild<TestEntity>();
        entity.AddComponent<UpdateComponent>();
        entity.AddComponent<AttributeComponent>();
    }

    private void Update()
    {
        MasterEntity.Update();
    }
}
```

![image-20230420001132566](C:\Users\Logarius\AppData\Roaming\Typora\typora-user-images\image-20230420001132566.png)

![image-20230420001144814](C:\Users\Logarius\AppData\Roaming\Typora\typora-user-images\image-20230420001144814.png)
