---
title: "GameAbilitySystem|01 Tag 游戏标签"
tags: 战斗
---

Tag，是一个纯标记位，自身不带有任何逻辑

```c#
namespace GameAbilitySystem
{
    /// <summary>
    /// 游戏标签，通过父子标签实现的一种树状结构，仅标记，没有逻辑也没有数据
    /// </summary>
    [CreateAssetMenu(menuName = "GameAbilitySystem/Tag")]
    public class GameTag : ScriptableObject
    {
        [SerializeField]
        private GameTag parent;
        
        /// <summary>
        /// 是否是另一个标签的孩子
        /// </summary>
        /// <param name="other">另一个标签</param>
        /// <param name="depth">深度</param>
        /// <returns></returns>
        public bool IsDescendantOf(GameTag other, int depth = 8)
        {
            if (this == other)
                return true;
            
            int i = 0;
            var tag = parent;
            while (depth > i++)
            {
                if (!tag) 
                    return false;

                if (tag == other) 
                    return true;

                tag = tag.parent;
            }
            return false;
        }
    }
}
```

标签是链式父子结果，一个标签可以有他的父标签，也可以通过接口判断某个标签是否是另一个标签的子节点

这有什么用呢？

举例来说，燃烧可以是一个标签，**Burning**，当然他可以有一个父标签，比如是**Debuff**

这很好理解，燃烧是异常状态下的一个子项，异常状态还可以有其他很多子项

![image-20231113113016271](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/202311131130336.png)

此时如果有一个技能需要清除所有的异常状态，那只需要筛选父标签**Debuff**即可

当然标签还有其他作用，可以说标签就是GAS的灵魂

比如说技能冷却，即释放技能A时，必须不存在A的CD标签
