---
title: "GameAbilitySystem|07 蓝图"
tags: 战斗
---

# FlowCanvas

蓝图直接使用FlowCanvas即可，但需要自己添加一些节点

## OnAbilityStart

```c#
namespace GameAbilitySystem
{
    [Category("GameAbilitySystem")]
    public class OnAbilityStart : RouterEventNode<GraphOwner>
    {
        private FlowOutput onEnter;
        private GameAbilitySpec spec;
        private GameObject owner;

        protected override void RegisterPorts()
        {
            onEnter = AddFlowOutput(" ");
            AddValueOutput("AbilitySpec", () => spec);
            AddValueOutput("Owner", () => owner);
        }

        protected override void Subscribe(EventRouter router)
        {
            router.onCustomEvent += OnAbilityStartAction;
        }

        protected override void UnSubscribe(EventRouter router)
        {
            router.onCustomEvent -= OnAbilityStartAction;
        }

        private void OnAbilityStartAction(string eventName, IEventData data)
        {
            if (data is EventData<GameAbilitySpec> eventData)
            {
                owner = eventData.sender as GameObject;
                spec = eventData.value;
                onEnter.Call(new Flow());
            }
        }
    }
}
```

那么，我们一个简单的演示技能蓝图就如下，流程很简单，创建一个GES，然后找到目标的ASC，然后应用GES

![image-20231115173751717](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311151737862.png)

此时这个技能的配置如下，没有消耗，但有一个冷却GE，同时，释放条件里有一个标签要求（这个标签就是冷却时间标签）

![image-20231115174934870](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311151749912.png)

冷却GE如下，没什么修饰器，但自带一个标签（也就是冷却时间标签），持续5秒

![image-20231115175002731](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311151750775.png)

蓝图中的伤害GE如下

![image-20231115175102726](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311151751760.png)







