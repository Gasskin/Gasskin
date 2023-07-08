---
title: "Job System"
tags: Unity模块
---

### 前言

本文章学习自[这里](https://www.raywenderlich.com/7880445-unity-job-system-and-burst-compiler-getting-started)，[这是](https://koenig-media.raywenderlich.com/uploads/2020/03/Introduction_to_Job_System.zip)素材文件的下载链接

多半内容都是文章内容加上自身理解，受限于时间以及自身能力水平，谨慎参考

首先需要3个Unity的官方包，请打开Unity的预览包选项，不然很多预览阶段的包你是无法在包管理器中看到的，具体如图

![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/1.png)

随后在Package Manager中安装Jobs、Burst以及Mathematics

其中前两个可以直接在包管理器中搜到，而数学库Mathematics需要通过gir url安装，具体如图

![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/2.png)

在出现的搜索栏中输入com.unity.mathematics@1.1并点击Add就可以添加数学库了

顺带一提，Burst似乎是随着Jobs一起安装的，目前好像不需要额外安装了，具体关系我没有研究，反正保证这三个包都装上了就行

### 关于Job System

Jobs System（任务系统）在我看来就是一种安全的多线程使用方式，不过目前看来使用方法也比较局限，正常情况下咱们PC端都是多核心的CPU了（这都2021年了），但是很多时候做游戏写C#脚本很少会直接调用多线程，一是我没那水平，二是多线程也容易翻车...

Jobs就是Unity官方提供的一种操作多线程的方式...虽然说必须得按照官方给的”模板“来写代码，有点难受，但不论如何，用起来是很方便快捷的，也很安全...这就是我对Jobs的理解...

### Starter

打开Introduction to Job System Starter项目，并打开主场景，是一个水面，本节的目的就是通过移动水面的顶点来制造波浪，这个模型的顶点还是不少的，如果通过传统的遍历顶点的方法，一个一个移动，那可想而知FPS肯定是非常之低的...

![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/3.png)

打开脚本WaveGenerator.cs

其中[Header("Wave Parameters")]下的3个参数用于生成随机数，也就是随机波浪，并不关键

而[Header("References and Prefabs")]下的参数则是对于波浪的引用，也不是关键，等下直接拖Prefab进去就行了

**首先添加两个参数**

```c#
NativeArray<Vector3> waterVertices; // 顶点
NativeArray<Vector3> waterNormals;  // 法线
```

NativeArray是Job System所提供的的特殊容器类型，为什么多线程不安全？一个原因就是各个线程之间同时访问某个数据时，容易引起竞争，从而导致很多的BUG，而这个NativeArray类型的目的就是为了防止竞争导致的问题，Job System提供了以下几种容器类型：

- NativeList                       一个可以调整长度的数组（Job版List)
- NativeHashMap            Job版HasMap
- NativeMultiHashMap   一个键可以对应多个值
- NativeQueue                 Job版Queue

当两个Job同时写入某一个相同的NativeContainer时，系统会自动报错，保证了线程安全

这里还有2个点需要注意

一、不可以向Native容器中传入引用类型的数据，那会导致线程安全问题，所以只能传值类型

二、如果某一个Job线程只需要访问Native容器，而不需要写入数据，那需要将容器标记为[ReadOnly]，下文会提到

**在Start()中初始化**

```c#
private void Start()
{
        waterMesh = waterMeshFilter.mesh; 

        waterMesh.MarkDynamic(); // 1

        waterVertices = 
            new NativeArray<Vector3>(waterMesh.vertices, Allocator.Persistent); // 2

        waterNormals = 
            new NativeArray<Vector3>(waterMesh.normals, Allocator.Persistent);
}
```

1. 让顶点能够被动态修改，和Job无关

2. 初始化Native容器，第一个参数是数据，第二个参数是分配方式，有3种分配方式，如下：

   ![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/4.png)

目前来看我还是喜欢第三种，毕竟生命周期长，随时能用...前两种分配方式的作用没有搞明白...

**在OnDestroy()中销毁**

```c#
private void OnDestroy()
{
    waterVertices.Dispose();
    waterNormals.Dispose();
}
```

就当做是必须操作就好...

**实现Job System**

Job本质是一个一个的Struct，每一个Struct就是一个小的Job，但Job必须实现自以下3个接口之一：

- IJob，标准任务，可以和其他所有定义的Job并行工作
- IJobParallelFor，并行任务，可以对某个Native容器中的所有数据并行的执行某个操作
- IJobParallelForTransform，类似于第二种，但专门为Transform类型的Native容器工作

我们的目的是动态修改所有的顶点数据，那肯定是采用第二种方式了，顶点数据不涉及Transform哦！

定义如下结构体

```c#
private struct UpdateMeshJob : IJobParallelFor
{
        // 1
        public NativeArray<Vector3> vertices;

        // 2
        [ReadOnly]
        public NativeArray<Vector3> normals;

        // 3
        public float offsetSpeed;
        public float scale;
        public float height;

        // 4
        public float time;
        
        // 5
        private float Noise(float x, float y)
        {
            float2 pos = math.float2(x, y);
            return noise.snoise(pos);
        }

		// 6
        public void Execute(int i)
        {
            // 7
            if (normals[i].z > 0f) 
            {
                // 8
                var vertex = vertices[i]; 
    
                // 9
                float noiseValue = 
                    Noise(vertex.x * scale + offsetSpeed * time, vertex.y * scale + 
                                                                 offsetSpeed * time); 
    
                // 10
                vertices[i] = 
                    new Vector3(vertex.x , vertex.y, noiseValue * height + 0.3f); 
            }

        }
}
```

咱们一个个解析

1. 定义一个原生容器，定义在结构体内部的容器是用于复制或者修改外部容器的数据的，此处的容器没有定义为[ReadOnly]说明这个容器会修改某个容器的数据

2. 同上，但注意这里是[ReadOnly]，说明这个容器只是用于访问某个主线程上的数据

   P.S.这里我的理解是，可以吧定义在类内的容器理解为主线程上的数据，定义在结构体内的容器理解为子线程上的数据，子线程想要访问主线程的数据，只可以通过这样的方式进行复制，而且如果两个子线程同时在修改主线程上某个容器的数据，那么Unity会报错。

3. 这些数据用于生成随机数，但我们没有初始化，因为这些数据也是可以通过外面传入进来的，由于是值类型，所以无所谓，但如果你传一个Transform进来就不行了

   P.S.这里也有个疑惑，第一个容器需要定义为NativeArray可以理解，因为他会修改主线程的数据，但是第二个容器定义为了[ReadOnly]说明他只是访问数据而不修改，那我们为什么不定以为普通的List，然后从外面传入呢？事实上按我的理解，这是可行的，但是！加入这个List有1000的长度，我们想从外面拷贝一个List进来，那就需要一个一个复制，这是很浪费空间的，而NativeArray本质是一个共享指针，NativeArray之间的复制是没有消耗的！

4. 不能传Time.time进来，所以就用一个参数暂存了，也是用于生成随机数的
5. 生成随机数的一个方法，很常见的噪声，具体不赘述了，和Job无关
6. 这个方法来自于接口IJobParallelFor，属于必须实现的接口，咱们不是在上面传入了一个NativeArray<Vector3> vertices嘛？这个原生数组中包含了咱们水模型的所有顶点信息，而我们的目标就是同时修改他的顶点数据，这个Execute(i)方法代表着我们将对每一个顶点所做的操作，其中i代表着这个顶点在数组中的序号，看起来这个函数类似于一个For循环有木有？但For循环是一个一个往后执行的，而这个Execute方法会被Unity自动分配到多个线程中
7. vertices是所有顶点的信息，而normals则是所有顶点的法线信息，通过这个i，我们很轻易的就能够访问到某个顶点的法线信息，这个法线可能需要一点点图形学知识，咱们不多说，这里的意思就是说，如果某个顶点是朝上的，那么我们才修改他的信息，再说的通俗一点，咱们只修改水面的上一侧，下侧不管
8. 获取噪声，与Job无关，就是个随机数
9. 修改顶点的数据

**执行任务**

上面只是定义好了任务，但还没执行，我们想每一帧都修改数据，这样才能形成连续的波浪！

先定义两个参数

```c#
JobHandle meshModificationJobHandle; // 1
UpdateMeshJob meshModificationJob; // 2
```

1. 句柄，比较抽象我也不知道该怎么解释，可以当做是某一个任务的当前信息，拟执行了某个Job他就会返回一个句柄给你，里面可以访问当前这个Job的执行情况，多数情况下我们需要等一个Job执行完成，才执行下一个Job，句柄的作用就是等待
2. 就是上面定义的那个Job结构体

在Update()中执行任务

```c#
private void Update()
{
        // 1
        meshModificationJob = new UpdateMeshJob()
        {
            vertices = waterVertices,
            normals = waterNormals,
            offsetSpeed = waveOffsetSpeed,
            time = Time.time,
            scale = waveScale,
            height = waveHeight
        };

        // 2
        meshModificationJobHandle = 
            meshModificationJob.Schedule(waterVertices.Length, 64);

}
```

1. 初始化任务，其中的参数就是咱们定义好的各种参数
2. 执行任务，有两个参数，第一个参数的数据长度，第二个参数是并行长度，比如顶点数是128，咱们并行长度是64，那么这个Job就会被划分为2部分，分别在2个子线程中进行，这个解释很模糊，大概是这个意思，着重在理解，表述可能是错的，具体还得看官方给的线程图，很难几句话说清楚

**等待任务结束**

```c#
private void LateUpdate()
{
    // 1
    meshModificationJobHandle.Complete();

    // 2
    waterMesh.SetVertices(meshModificationJob.vertices);
    
    // 3
    waterMesh.RecalculateNormals();

}
```

1. 句柄的作用就是这个，表示在这里需要等待上面Schedule()的Job结束，如果不结束我们怎么能获取新的顶点数据呢...
2. 设置顶点数据
3. 重新计算法线

### 测试一下

设置好脚本数据

![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/5.png)

点击运行，应该是有用的，此时查看一下Status

![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/6.png)

这么多的顶点还能有这个数据，也是挺喜人的，查看一下任务管理器，发现所有核心确实是都在跑的

![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/7.png)

### Burst

轻松一步，瞬间超神，具体原理我没有研究，总之咱们现在给刚刚定义的Job加上一个标签

![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/8.png)

然后再运行

![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/9.png)



直接超神！

### IJobParallelForTransform

上面我们还提到过一种特殊的Job，专门针对于Transform类型，因为Transform类型是引用类型，所以必须通过特定的方法实现Job化

打开脚本FishGenerator.cs，这个脚本会生成大量自由移动的小鱼

**首先添加两个属性**

```c#
// 1
private NativeArray<Vector3> velocities;

// 2
private TransformAccessArray transformAccessArray;
```

1. 移动速度，和上面提到的容器是一样的，只不过储存的是Vector3类型
2. 专门针对于Transform的特殊容器，只能输出Transform类型的数据，且是一个数组

**在Start()中初始化**

```csharp
private void Start()
{
    // 1
    velocities = new NativeArray<Vector3>(amountOfFish, Allocator.Persistent);

    // 2
    transformAccessArray = new TransformAccessArray(amountOfFish);

    for (int i = 0; i < amountOfFish; i++)
    {

        float distanceX = 
            Random.Range(-spawnBounds.x / 2, spawnBounds.x / 2);

        float distanceZ = 
            Random.Range(-spawnBounds.z / 2, spawnBounds.z / 2);

        // 3
        Vector3 spawnPoint = 
            (transform.position + Vector3.up * spawnHeight) + new Vector3(distanceX, 0, distanceZ);

        // 4
        Transform t = 
            (Transform)Instantiate(objectPrefab, spawnPoint, 
                Quaternion.identity);

        // 5
        transformAccessArray.Add(t);
    }

}
```

1. 和上面没啥区别，就是初始化容器，第一个参数是鱼的数量，第二个参数是分配方式
2. 也是初始化容器，不过初始化Transform容器和其他容器稍微有一点区别，只要写好大小就行

   3.4. 生成随机数罢了，和Job无关，目的是随机生成鱼的位置，随便看看就行

5. 把生成的随机位置添加到容器中

**同样需要在OnDestroy()时销毁容器**

```csharp
private void OnDestroy()
{
        transformAccessArray.Dispose();
        velocities.Dispose();
}
```

然后填写好参数，测试一下

![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/10.png)

会发现随机生成了很多小鱼，但是是静止的

![1](https://www.logarius996.icu/images/UnityStudy/JobSystem/11.png)

**创建小鱼移动Job**

```c#
[BurstCompile]
struct PositionUpdateJob : IJobParallelForTransform
{
    public NativeArray<Vector3> objectVelocities;

    public Vector3 bounds;
    public Vector3 center;

    public float jobDeltaTime;
    public float time;
    public float swimSpeed;
    public float turnSpeed;
    public int swimChangeFrequency;

    public float seed;

    public void Execute (int i, TransformAccess transform)
    {
        // 1
        Vector3 currentVelocity = objectVelocities[i];

        // 2            
        random randomGen = new random((uint)(i * time + 1 + seed));

        // 3
        transform.position += 
            transform.localToWorldMatrix.MultiplyVector(new Vector3(0, 0, 1)) * 
            swimSpeed * 
            jobDeltaTime * 
            randomGen.NextFloat(0.3f, 1.0f);

        // 4
        if (currentVelocity != Vector3.zero)
        {
            transform.rotation = 
                Quaternion.Lerp(transform.rotation, 
                    Quaternion.LookRotation(currentVelocity), turnSpeed * jobDeltaTime);
        }
        
        Vector3 currentPosition = transform.position;

        bool randomise = true;

        // 5
        if (currentPosition.x > center.x + bounds.x / 2 || 
            currentPosition.x < center.x - bounds.x/2 || 
            currentPosition.z > center.z + bounds.z / 2 || 
            currentPosition.z < center.z - bounds.z / 2)
        {
            Vector3 internalPosition = new Vector3(center.x + 
                                                   randomGen.NextFloat(-bounds.x / 2, bounds.x / 2)/1.3f, 
                0, 
                center.z + randomGen.NextFloat(-bounds.z / 2, bounds.z / 2)/1.3f);

            currentVelocity = (internalPosition- currentPosition).normalized;

            objectVelocities[i] = currentVelocity;

            transform.rotation = Quaternion.Lerp(transform.rotation, 
                Quaternion.LookRotation(currentVelocity), 
                turnSpeed * jobDeltaTime * 2);

            randomise = false;
        }

        // 6
        if (randomise)
        {
            if (randomGen.NextInt(0, swimChangeFrequency) <= 2)
            {
                objectVelocities[i] = new Vector3(randomGen.NextFloat(-1f, 1f), 
                    0, randomGen.NextFloat(-1f, 1f));
            }
        }

    }
}
```

这个Job和上面那个波浪Job很像，区别是实现自接口IJobParallelForTransform，只有这个接口可以访问Transform容器，同样也有一个Execute方法需要实现，其中i代表容器内的序号，transform则是引用

具体算法其实并不重要，主要目的就是让小鱼的Transform线性变化，如果不使用Job而在Update中写，相信很多人随便就能写了

1. 第i条小鱼当前的速度
2. 随机种子，需要using random = Unity.Mathematics.Random;
3. 随机移动距离，会根据上面几个指定的参数决定
4. 旋转
5. 6. 防止小鱼移出水面的范围

**执行任务**

同样需要先创建一个句柄以及任务

```csharp
private PositionUpdateJob positionUpdateJob;

private JobHandle positionUpdateJobHandle;
```

在Update()中初始化并执行

```csharp
private void Update()
{
    // 1
    positionUpdateJob = new PositionUpdateJob()
    {
        objectVelocities = velocities,
        jobDeltaTime = Time.deltaTime,
        swimSpeed = this.swimSpeed,
        turnSpeed = this.turnSpeed,
        time = Time.time,
        swimChangeFrequency = this.swimChangeFrequency,
        center = waterObject.position,
        bounds = spawnBounds,
        seed = System.DateTimeOffset.Now.Millisecond
    };

    // 2
    positionUpdateJobHandle = positionUpdateJob.Schedule(transformAccessArray);

}
```

1. 初始化任务，注意我们没并有传入一个Transform数组
2. 执行任务，此时可以传入咱们的transformAccessArray

同样在LateUpdate()中等待任务结束

```csharp
private void LateUpdate()
{
    positionUpdateJobHandle.Complete(); 
}
```

点击运行，小鱼就动了起来！























