---
title: "ECGameplay|04 效果和状态"
categories: ECGameplay
tags: 战斗
---

# 配置文件

这一章往后很多效果依赖于数据，所以需要配表

直接使用Luban，不多讲述了（略）

# 效果

效果依附于能力，比如普攻，最基础的普攻自带是带有伤害效果，但普攻还可以附带比如减速效果

所以说，效果是基于能力的再一次抽象

万物皆效果，效果可以看做是一种BUFF，比如普攻会造成伤害，但我们的普攻不直接进行伤害结算，而是给目标添加一个效果BUFF，具体逻辑由BUFF去触发

## EffectComponent

和我们的ActionPointComponent一样，本身没有太多复杂的逻辑，主要是为了集中处理Entity上的Effect

```c#
namespace ECGameplay
{
    public class EffectComponent : Component
    {
        public CombatEntity OwnerEntity => Entity.As<CombatEntity>();
		// 所有的Effect字典
        public Dictionary<int,List<EffectAbility>> Id2Effects { get; set; } = new Dictionary<int, List<EffectAbility>>();

        // 添加一个Effect
        public EffectAbility AttachEffect(EffectConfig effectConfig)
        {
            var effect = OwnerEntity.AddChild<EffectAbility>(effectConfig);
            if (!Id2Effects.ContainsKey(effectConfig.Id))
            {
                Id2Effects.Add(effectConfig.Id,new List<EffectAbility>());
            }
            Id2Effects[effectConfig.Id].Add(effect);
            return effect;
        }

        // 销毁一个Effect
        public void RemoveEffect(EffectAbility effect)
        {
            var id = effect.EffectConfig.Id;
            if (Id2Effects.TryGetValue(id,out var list))
            {
                list.Remove(effect);
                Entity.Destroy(effect);
            }
        }
        
        public bool TryGetEffect(int id,out EffectAbility effect)
        {
            effect = null;
            if (Id2Effects.TryGetValue(id,out var list) && list.Count > 0)
            {
                effect = list[0];
                return true;
            }
            return false;
        }
    }
}
```

然后CombatEntity再做一些封装

```c#
public EffectAbility AttachEffect(int id)
{
    return GetComponent<EffectComponent>().AttachEffect(TableUtil.Tables.EffectTable[id]);
}

public void RemoveEffect(EffectAbility effect)
{
    GetComponent<EffectComponent>().RemoveEffect(effect);
}
```

## AddEffectAction

添加效果的行为

```c#
namespace ECGameplay
{
    public class AddEffectAction: Entity, IAction
    {
        public CombatEntity OwnerEntity
        {
            get=>GetParent<CombatEntity>() ;
            set{}
        }
        
        public bool Enable { get; set; }

        public bool TryMakeAction(out AddEffectActionExecution actionExecution)
        {
            if (!Enable)
            {
                actionExecution= null;
            }
            else
            {
                actionExecution = OwnerEntity.AddChild<AddEffectActionExecution>();
                actionExecution.Action = this;
                actionExecution.Creator = OwnerEntity;
            }
            return Enable;
        }
    }


    public class AddEffectActionExecution : Entity, IActionExecution
    {
        public IAction Action { get; set; }
        public CombatEntity Creator { get; set; }
        public CombatEntity Target { get; set; }

        public EffectConfig EffectConfig { get; set; }

        public EffectAbility Effect { get; set; }
        
        public void AddEffect()
        {
            // 根据配表判断Effect是否可以叠加
            if (!EffectConfig.CanStack)
            {
                if (Target.GetComponent<EffectComponent>().TryGetEffect(EffectConfig.Id,out var effect))
                {
                    effect.Reset();
                    return;
                }
            }

            // 根据Effect的作用目标，给不同目标添加Effect
            switch (EffectConfig.EffectTarget)
            {
                case EffectTarget.Target:
                    Effect = Target.AttachEffect(EffectConfig.Id);
                    break;
                case EffectTarget.Self:
                    Effect = Creator.AttachEffect(EffectConfig.Id);
                    break;
            }

            Effect.AddEffectActionExecution = this;
            Effect.ActivateAbility();
            
            Creator?.TriggerActionPoint(ActionPointType.AfterGiveEffect, this);
            Target?.TriggerActionPoint(ActionPointType.AfterReceiveEffect, this);   
            
            FinishAction();
        }
        
        public void FinishAction()
        {
            Destroy(this);
        }
    }
}
```

## EffectAbility

效果能力，主要存储Effect的配表信息，以及添加具体逻辑的Component

```c#
namespace ECGameplay
{
    public interface IEffectComponent
    {
        public void Reset();
    }

    public class EffectAbility : Entity
    {
        public CombatEntity OwnerEntity
        {
            get=> Parent.As<CombatEntity>(); 
            set{}
        }
        public EffectConfig EffectConfig { get; set; }
        public AddEffectActionExecution AddEffectActionExecution { get; set; }
        public Component EffectComponent { get; set; }

        public override void Awake(object initData)
        {
            EffectConfig = initData as EffectConfig;

            if (EffectConfig == null)
                return;

            // 根据配置，添加不同的组件，具体的逻辑再组件内
            switch (EffectConfig.EffectType)
            {
                case EffectType.Damage:
                    EffectComponent = AddComponent<EffectDamageComponent>();
                    break;
                case EffectType.Cure:
                    EffectComponent = AddComponent<EffectCureComponent>();
                    break;
            }
        }

        public void ActivateAbility()
        {
            EffectComponent.Enable = true;
        }

        public void DeactivateAbility()
        {
            EffectComponent.Enable = false;
        }
        
        public void EndAbility()
        {
            OwnerEntity.RemoveEffect(this);
        }

        public void Reset()
        {
            (EffectComponent as IEffectComponent)?.Reset();
        }
    }
}
```

### EffectDamageComponent

伤害效果组件

```c#
namespace ECGameplay
{
    public class EffectDamageComponent : Component, IEffectComponent
    {
        // 默认是不激活的
        public override bool DefaultEnable { get; set; } = false;

        private EffectAbility EffectAbility => Entity.As<EffectAbility>();
        private IGameTimer Timer { get; set; }

        // 激活时触发
        public override void OnEnable()
        {
            // 根据不同的类型触发不同的逻辑
            switch (EffectAbility.EffectConfig.EffectTiming)
            {
                case EffectTiming.Immediate:
                    ApplyDamage();
                    EffectAbility.EndAbility();
                    break;
                case EffectTiming.Duration:
                    break;
                case EffectTiming.Interval:
                    Timer = new IntervalTimer(EffectAbility.EffectConfig.Duration / 1000f,
                        EffectAbility.EffectConfig.Interval / 1000f, ApplyDamage, EffectAbility.EndAbility);
                    break;
            }
        }

        public override void OnDisable()
        {
            Debug.LogError("伤害结束");
        }

        public override void Update()
        {
            Timer?.Update(Time.deltaTime);
        }

        public void Reset()
        {
        }

        // 最终还是调用伤害行为
        private void ApplyDamage()
        {
            if (EffectAbility.AddEffectActionExecution.Creator.DamageAction.TryMakeAction(out var actionExecution))
            {
                actionExecution.Target = EffectAbility.OwnerEntity;
                actionExecution.EffectAbility = EffectAbility;
                actionExecution.ApplyDamage();
            }
        }
    }
}
```

# IGameTimer

一个接口，效果执行期间可能需要各种时间判断（比如循环，倒计时结束）

继承这个接口实现自己的Timer

```c#
public interface IGameTimer
{
    public bool IsFinish { get; }
    public void Update(float delta);
    public void Reset();
}
```

比如倒计时Timer，会在倒计时结束时触发回调

```c#
public class DurationTimer : IGameTimer
{
    public bool IsFinish => time >= maxTime;
    private Action action;
    private readonly float maxTime;
    private float time;

    public DurationTimer(float maxTime, Action action)
    {
        this.maxTime = maxTime;
        this.action = action;
        time = 0;
    }

    public void Update(float delta)
    {
        if (!IsFinish)
        {
            time += delta;
            if (IsFinish)
            {
                action?.Invoke();
            }
        }
    }

    public void Reset()
    {
        time = 0;
    }
}
```
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

# 公式解析


```c#
namespace ECGamePlay
{
    public class ExpressionUtil
    {
        private static ExpressionParser ExpressionParser { get; set; } = new ExpressionParser();   
        public static double 			
            
        TryEvaluate(string expressionStr, AttributeComponent attr)
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

# 测试

增加一个配置

![image-20230425223926982](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304252239029.png)

进行攻击

<video style="width: 30%; height: auto; object-fit: contain;" src="https://www.logarius996.icu/videos/2023年4月25日224014.mp4" controls=""></video>

<video style="width: 30%; height: auto; object-fit: contain;" src="https://www.logarius996.icu/videos/2023年4月27日224914.mp4" controls=""></video>

# 技能

其实普攻也被我们作为技能来写了，不过普攻比较特殊

## SpellSkillAction

释放技能行为

```c#
namespace ECGameplay
{
    public class SkillSpellAction : Entity, IAction
    {
        public CombatEntity OwnerEntity {get => GetParent<CombatEntity>();set{}}
        public bool Enable { get; set; }

        public bool TryMakeAction(out SkillSpellActionExecution actionExecution)
        {
            if (!Enable)
            {
                actionExecution = null;
            }
            else
            {
                actionExecution = new SkillSpellActionExecution();
                actionExecution.Action = this;
                actionExecution.Creator = OwnerEntity;
            }
            return Enable;
        }
    }

    public class SkillSpellActionExecution : Entity, IActionExecution
    {
        public IAction Action { get; set; }
        public CombatEntity Creator { get; set; }
        // 技能可能是指定目标，也可能是指定方向，也可能是多个目标
        public CombatEntity Target { get; set; }
        public List<Component> Targets { get; set; } = new List<Component>();
        public Vector3 Point { get; set; }

        public SkillAbilityExecution SkillAbilityExecution { get; set; }
        public SkillAbility SkillAbility { get; set; }
        

        public void SpellSkill()
        {
            Creator.TriggerActionPoint(ActionPointType.BeforeSpell, this);
            
            SkillAbilityExecution = SkillAbility.CreateExecution() as SkillAbilityExecution;
            SkillAbilityExecution.ActionExecution = this;
            SkillAbilityExecution.BeginExecute();
        }
        
        public void FinishAction()
        {
            Creator.TriggerActionPoint(ActionPointType.AfterSpell, this);
            Destroy(this);
        }
    }
}
```

## SkillAbility

技能能力

```c#
using System.Collections.Generic;
using cfg.Skill;

namespace ECGameplay
{
    public class SkillAbility : Entity, IAbility
    {
        public CombatEntity OwnerEntity
        {
            get => GetParent<CombatEntity>();
            set { }
        }
        
        public bool Enable { get; set; }

        public SkillConfig SkillConfig { get; set; }

        public override void Awake(object initData)
        {
            SkillConfig = initData as SkillConfig;
        }

        public void TryActivateAbility()
        {
        }

        public void ActivateAbility()
        {
        }

        public void DeactivateAbility()
        {
        }

        public void EndAbility()
        {
        }

        public Entity CreateExecution()
        {
            var execution = OwnerEntity.AddChild<SkillAbilityExecution>(this);
            execution.Ability = this;
            execution.OwnerEntity = OwnerEntity;
            return execution;
        }
    }

    public class SkillAbilityExecution : Entity, IAbilityExecution
    {
        public IAbility Ability { get; set; }
        public CombatEntity OwnerEntity { get; set; }
        public IActionExecution ActionExecution { get; set; }

        public void BeginExecute()
        {
        }

        public void EndExecute()
        {
        }
    }
}
```

## SpellSkillComponent

添加一个释放技能的组件，集中一下逻辑，两种释放技能的方式，其中范围技能都通过Point实现，内置了朝向

```c#
namespace ECGameplay
{
    public class SpellSkillComponent : Component
    {
        private CombatEntity CombatEntity => GetEntity<CombatEntity>();
        public override bool DefaultEnable { get; set; } = true;
        
        public void SpellWithTarget(SkillAbility spellSkill, CombatEntity targetEntity)
        {
            if (CombatEntity.SkillAbility != null)
                return;

            if (CombatEntity.SkillSpellAction.TryMakeAction(out var actionExecution))
            {
                actionExecution.SkillAbility = spellSkill;
                actionExecution.Target = targetEntity;
                actionExecution.Point = targetEntity.Position;
                spellSkill.OwnerEntity.Rotation 
                    = Quaternion.LookRotation(targetEntity.Position - spellSkill.OwnerEntity.Position);
                actionExecution.InputDirection = spellSkill.OwnerEntity.Rotation.eulerAngles.y;
                actionExecution.SpellSkill();
            }
        }

        public void SpellWithPoint(SkillAbility spellSkill, Vector3 point)
        {
            if (CombatEntity.SkillAbility != null)
                return;

            if (CombatEntity.SkillSpellAction.TryMakeAction(out var actionExecution))
            {
                actionExecution.SkillAbility = spellSkill;
                actionExecution.Point = point;
                spellSkill.OwnerEntity.Rotation = Quaternion.LookRotation(point - spellSkill.OwnerEntity.Position);
                actionExecution.InputDirection = spellSkill.OwnerEntity.Rotation.eulerAngles.y;
                actionExecution.SpellSkill();
            }
        }
    }
}
```



























