---
title: "Playable"
tags: 动画
---

有啥好处就不空谈了，直接上干货

# 架构

Playable有很多类型，但是总体上，由两部分组成

- Playable
- PlayableOutput

从名字就看得出来，他两是对应的，一个是输入，一个是输出

## Playable

作为输入节点，一共支持如下类型

![image-20230414214350642](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304142143675.png)

## PlayableOutput

作为输出，类型就少很多了，因为从上面的输入来看，我们也只有三种类型的输出

![image-20230414214454013](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304142144051.png)

## PlayableGraph

输入输出管理器，可以看做是一个管理类，管理我们所有的输入（Playable）和输出（PlayableOutput）

如果使用可视化插件，我们可以发现，一个简单的Graph如下（一个输入，一个输出）

![image-20230414214658265](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304142146290.png)

当然，Graph通常是不止两个节点的

![image-20230414214734466](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304142147500.png)

# 简单使用

一个模型，挂在了Animator，没用Animation也没有AnimatorController

![image-20230414215028761](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304142150814.png)

![image-20230414215021717](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304142150741.png)

```c#
public class PlayableTest : MonoBehaviour
{
    public AnimationClip clip;
    private PlayableGraph graph;

    void Start()
    {
        // 创建管理器
        graph = PlayableGraph.Create("Test");
        // 创建一个AnimationClip输入
        var animationClipPlayable = AnimationClipPlayable.Create(graph, clip);
        // 创建Animation输出
        var animationPlayableOutput = AnimationPlayableOutput.Create(graph, "OutPut", GetComponent<Animator>());
        // 连接输入输出
        animationPlayableOutput.SetSourcePlayable(animationClipPlayable);
        // 播放
        graph.Play();
    }
}
```

<video style="width: 30%; height: auto; object-fit: contain;" src="https://www.logarius996.icu/assets/videos/2023年4月14日215854.mp4" controls=""></video>

当然这么做可能有一点麻烦，官方提供了一个简单的工具类，可以直接播放一个方法

```c#
AnimationPlayableUtilities.PlayClip(GetComponent<Animator>(), clip, out PlayableGraph graph);
```

Graph是可以设置更新频率的

通过方法`PlayableGraph.SetTimeUpdateMode`

- **DSPClock** 基于DSP，与声音同步
- **GameTime** 基于Time.time，如果timeScale是0，那么也会停止
- **UnscaledGameTime** 同上，不过不会停止
- **Manual** 手动更新，需要自己调用PlayableGraph.Evaluate()

# PlayableGraph Visualizer

一个可视化的插件，很好用，Add From Git Url : com.unity.playablegraph-visualizer

然后就可以在Window里打开了

![image-20230414221059224](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202304142210243.png)

<video style="width: 30%; height: auto; object-fit: contain;" src="https://www.logarius996.icu/assets/videos/2023年4月14日221211.mp4" controls="">



















