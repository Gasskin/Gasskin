---
title: "ECGameplay04 效果和状态"
tags: 战斗
---

# 配置文件

这一章往后很多效果依赖于数据，所以需要配表

直接使用Luban，不多讲述了（略）

## 效果类型

后续还会添加，不再补充说明

![image-20230422003654759](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304220036825.png)

## 技能表

后续字段还会添加，不再补充说明

![image-20230422003742126](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304220037152.png)

## 效果表

后续字段还会添加，不再补充说明

![image-20230422003814767](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304220038795.png)

# 效果

效果依附于能力，比如普攻，最基础的普攻自带是带有伤害效果，但普攻还可以附带比如减速效果

所以说，效果是基于能力的再一次抽象

## AbilityEffectComponent

能力效果组件，在Ability被创建的时候会添加

同时会传入技能配表，一个技能会附有多个效果，每个效果都会创建一个AbilityEffect

只是作为一个中转组件，本身没啥逻辑，即使没有这个类，我们也可以把所有信息存储在Ability里

```c#
namespace ECGameplay
{
    public class AbilityEffectComponent : Component
    {
        /// 默认不激活
        public override bool DefaultEnable { get; set; } = false;

        /// 效果列表，一个能力可以有多个效果
        public List<AbilityEffect> AbilityEffects { get; set; } = new List<AbilityEffect>();

        public override void Awake(object initData)
        {
            // 技能配表
            if (initData is SkillConfig skillConfig)
            {
                // 创建该技能的所有效果
                foreach (var skillEffect in skillConfig.AttachEffect_Ref)
                {
                    var abilityEffect = Entity.AddChild<AbilityEffect>(skillEffect);
                    AbilityEffects.Add(abilityEffect);
                }
            }
        }

        public override void OnEnable()
        {
            foreach (var abilityEffect in AbilityEffects)
            {
                abilityEffect.EnableEffect();
            }
        }

        public override void OnDisable()
        {
            foreach (var abilityEffect in AbilityEffects)
            {
                abilityEffect.DisableEffect();
            }
        }

        // 应用某一个具体效果
        public void AssignAbilityEffect(int index, IAbilityExecution execution)
        {
            if (AbilityEffects.Count <= index) 
                return;
            AbilityEffects[index].AssignEffect(execution);
        }

        // 应用这个能力上的所有效果
        public void AssignAllAbilityEffect(IAbilityExecution execution)
        {
            foreach (var abilityEffect in AbilityEffects)
            {
                abilityEffect.AssignEffect(execution);
            }
        }
    }
}
```

## AbililtyEffect

能力效果，诸如造成伤害啊，减速啊，等等

其中IEffectComponent是需要Component实现的，用来标注这个Component是一个能力应用组件

当然把所有逻辑都写在AbilityEffect里也是可以的，但是会让这个类变成一个超级类（非常复杂

```C#
namespace ECGameplay
{
    public interface IEffectComponent
    {
        void OnApplyEffect(IAbilityExecution execution, AbilityEffect effect);
    }

   
    [DrawProperty]
    public class AbilityEffect : Entity
    {
        public bool Enable { get; set; }

        /// 持有能力效果的Entity，自然是能力本身
        public IAbility OwnerAbility => Parent as IAbility;

        /// 效果配表
        public SkillEffectConfig SkillEffectConfig { get; set; }


        public override void Awake(object initData)
        {
            SkillEffectConfig = initData as SkillEffectConfig;
            if (SkillEffectConfig == null) 
                return;
            // 根据配表的不同，添加不同的组件
            switch (SkillEffectConfig.EffectType)
            {
                case EffectType.Damage:
                    AddComponent<DamageEffectComponent>();
                    break;
                case EffectType.Cure:
                    break;
            }
        }

        public override void OnDestroy()
        {
            DisableEffect();
        }

        public void EnableEffect()
        {
            Enable = true;
            foreach (var comp in Components.Values)
            {
                comp.Enable = true;
            }
        }

        public void DisableEffect()
        {
            Enable = false;
            foreach (var comp in Components.Values)
            {
                comp.Enable = false;
            }
        }

        // 应用效果的地方
        public void AssignEffect(IAbilityExecution execution)
        {
            foreach (var comp in Components.Values)
            {
                if (comp is IEffectAssignComponent effectAssignComponent)
                {
                    effectAssignComponent.OnApplyEffect(execution, this);
                }
            }
        }
    }
}
```

### EffectDamageComponent

一个伤害效果组件的，需要实现IEffectTrigger接口

仍然是一个逻辑中转，从CombatEntity的DamageAction里创建一个伤害行为

```c#

namespace ECGameplay
{
	public class DamageEffectComponent : Component, IEffectComponent
    {
        public SkillEffectConfig SkillEffectConfig { get; set; }

        public override void Awake()
        {
            SkillEffectConfig = GetEntity<AbilityEffect>().SkillEffectConfig;
        }

        public void OnApplyEffect(IAbilityExecution execution, AbilityEffect effect)
        {
            var abilityExecution = (AttackAbilityExecution)execution;
            if (GetEntity<AbilityEffect>().OwnerAbility.OwnerEntity.DamageAction.TryMakeAction(out var actionExecution))
            {
                actionExecution.Target = abilityExecution.AttackActionExecution.Target;
                actionExecution.AbilityEffect = effect;
                actionExecution.ApplyDamage();
            }
        }
    }
}
```

## 公式解析

用于把一串string解析成一个数学公式，支持括号嵌套

```C#
namespace ECGamePlay
{
    public class ExpressionUtil
    {
        private static ExpressionParser ExpressionParser { get; set; } = new ExpressionParser();


        public static double TryEvaluate(string expressionStr, AttributeComponent attr)
        {
            Expression expression = null;
            try
            {
                expression = ExpressionParser.EvaluateExpression(expressionStr);
                if (expression.Parameters.ContainsKey("攻击力"))
                {
                    expression.Parameters["攻击力"].Value = attr.Attack.Value;
                }
            }
            catch (System.Exception e)
            {
                Debug.LogError(expressionStr);
                Debug.LogError(e);
            }
            if (expression == null)
                return 0;
            return expression.Value;
        }
    }
}
```

其中`ExpressionParser`是引入的一个库，源码就不放了，作用如下

```c#
var test = ExpressionUtil.TryEvaluate("(攻击力1+100)*2+攻击力1+攻击力2");
```

![image-20230422001436088](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304220014117.png)

然后可以

```c#
test.Parameters["攻击力1"].Value = 100;
test.Parameters["攻击力2"].Value = 200;
Debug.Log(test.Value);
```

![image-20230422001615501](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304220016521.png)

当然，这里面的字符串，如何和我们的属性对应，则需要代码写死了，没有太好的办法

# DamageAction

伤害行为，真正进行伤害计算的地方

```c#
namespace ECGameplay
{
    public class DamageAction: Entity,IAction
    {
        public CombatEntity OwnerEntity
        {
            get=> GetParent<CombatEntity>();
            set{}
        }
        
        public bool Enable { get; set; }

        public bool TryMakeAction(out DamageActionExecution actionExecution)
        {
            if (!Enable)
            {
                actionExecution = null;
            }
            else
            {
                actionExecution = AddChild<DamageActionExecution>();
                actionExecution.Creator = OwnerEntity;
                actionExecution.Action = this;
            }
            return Enable;
        }
    }

    public class DamageActionExecution : Entity, IActionExecution
    {
        public IAction Action { get; set; }
        public AbilityEffect AbilityEffect { get; set; }
        public CombatEntity Creator { get; set; }
        public CombatEntity Target { get; set; }

        public float Damage { get; set; }

        public void ApplyDamage()
        {
            Creator?.TriggerActionPoint(ActionPointType.BeforeCauseDamage, this);
            Target?.TriggerActionPoint(ActionPointType.BeforeReceiveDamage, this);

            var skillEffectConfig = AbilityEffect.SkillEffectConfig;
            var attr = Creator?.GetComponent<AttributeComponent>();
            if (attr == null || skillEffectConfig == null)
            {
                FinishAction();
                return;
            }

            // 没触发
            if (!MathUtil.PrizeDraw(skillEffectConfig.Probability))
            {
                FinishAction();
                return;
            }
            
            Damage = (float)ExpressionUtil.TryEvaluate(skillEffectConfig.ValueFormula, attr);
            var isCritical = MathUtil.PrizeDraw(attr.CriticalProbability.Value);
            if (isCritical && skillEffectConfig.CanCritical)
            {
                Damage *= attr.CriticalDamage.Value;
            }
            
            Target?.ReceiveDamage(this);
            
            Creator?.TriggerActionPoint(ActionPointType.AfterCauseDamage, this);
            Target?.TriggerActionPoint(ActionPointType.AfterReceiveDamage, this);
            
            FinishAction();
        }
        
        public void FinishAction()
        {
            Destroy(this);
        }
    }
}
```

其中ReceiveDamage如下

```c#
namespace ECGameplay
{
    public class CombatEntity: Entity
    {
		public void ReceiveDamage(IActionExecution actionExecution)
        {
            var damageAction = actionExecution as DamageActionExecution;
            if (damageAction == null)
                return;
            Debug.LogError("ReceiveDamage : " + damageAction.Damage);
        }
    }
}
```

# 测试

需要给CombatEntity添加以上新增的能力和行为

![image-20230423002231195](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304230022229.png)

为什么会有两次伤害呢？

因为配表里给这个技能添加了两个效果

编辑器结构如下

![image-20230423002326125](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304230023157.png)

![image-20230423002336682](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304230023713.png)

# 流程解析

以普攻为例，解析整个框架的运行流程

## 各组件的关系

### CombatEntity

仅仅是对于战斗Entity的总结，自身没有什么特殊逻辑

![image-20230423003016412](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304230030445.png)

### Action

极其简单的Action，比如格挡，所有逻辑都会直接写在Action里

但是大多数Action会有对应的执行体

执行体是进行逻辑运算的地方，并不一定会调用Ability

![image-20230423004547452](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304230045484.png)

### Ablity

一定会有对应的能力执行体

并且每一个能力都会附有一个或者多个AbilityEffect

AbilityEffect由AbilityEffectComponent添加

![image-20230423005848299](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304230058326.png)

### AbilityEffect

能力效果，本身没有啥逻辑，会根据不同的效果，去调用CombatEntity里的不同行为

![image-20230423010008990](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304230100013.png)

## 生命周期

![image-20230423221806771](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304232218865.png)

# 简单Demo

截止到当前，编写的代码基本都是纯逻辑的，所以简单写一个小demo，提供一些可视化

## Hero

```c#
public class Hero : MonoBehaviour
{
    public CombatEntity combatEntity;

    public Monster monsterMono;
    public GameObject lineEffectPrefab;
    public GameObject hitEffectPrefab;
    public Image healthBar;
    public Text valueText;

    void Start()
    {
        combatEntity = MasterEntity.Instance.AddChild<CombatEntity>();
        combatEntity.ListenActionPoint(ActionPointType.BeforeGiveAttackEffect, OnBeforeGiveAttackEffect);
    }

    public void Attack()
    {
        if (combatEntity.AttackAction.TryMakeAction(out var actionExecution))
        {
            actionExecution.Target = monsterMono.combatEntity;
            actionExecution.ApplyAttack();
        }
    }

    private void OnBeforeGiveAttackEffect(Entity actionExecution)
    {
        SpawnLineEffect(transform.position, monsterMono.transform.position);
        SpawnHitEffect(transform.position, monsterMono.transform.position);
    }


    private void SpawnLineEffect(Vector3 p1, Vector3 p2)
    {
        var attackEffect = Instantiate(lineEffectPrefab);
        attackEffect.transform.position = Vector3.zero;
        attackEffect.GetComponent<LineRenderer>().SetPosition(0, p1);
        attackEffect.GetComponent<LineRenderer>().SetPosition(1, p2);
        Destroy(attackEffect, 0.05f);
    }
    
    private void SpawnHitEffect(Vector3 p1, Vector3 p2)
    {
        var vec = p1 - p2;
        var hitPoint = p2 + vec.normalized * .6f;
        hitPoint += Vector3.up;
        var hitEffect = Instantiate(hitEffectPrefab);
        hitEffect.transform.position = hitPoint;
        GameObject.Destroy(hitEffect, 0.2f);
    }
}
```

## Monster

```c#
public class Monster : MonoBehaviour
{
    public CombatEntity combatEntity;

    public Hero heroMono;
    public Image healthBar;
    public Text valueText;

    private void Start()
    {
        combatEntity = MasterEntity.Instance.AddChild<CombatEntity>();
        combatEntity.ListenActionPoint(ActionPointType.AfterReceiveDamage, OnReceiveDamage);
    }

    private void OnReceiveDamage(Entity actionExecution)
    {
        var damageActionExecution = actionExecution.As<DamageActionExecution>();
        if (damageActionExecution == null) 
            return;
        var attr = combatEntity.GetComponent<AttributeComponent>();
        var hpPct = attr.HealthPoint.Value / attr.HealthPointMax.Value;
        healthBar.fillAmount = hpPct;
        var damageText = Instantiate(valueText, valueText.transform.parent, false);
        damageText.gameObject.SetActive(true);
        damageText.text = $"-{damageActionExecution.Damage}";
        damageText.color = Color.red;
        Destroy(damageText.gameObject, 0.2f);
    }
}
```

## 场景

![image-20230423223141803](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304232233699.png)

![image-20230423223151334](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304232231366.png)

## 测试

<video style="width: 30%; height: auto; object-fit: contain;" src="https://www.logarius996.icu/assets/videos/2023年4月23日223233.mp4" controls=""></video>







































