---
title: "Unity与Android通信"
tags: SDK
---

### 前言

目前网上找到的相关文章都是互相转发抄袭，真的毫无阅读价值，如此简单的通信竟然花了我好几天的功夫学习，故做此记录

### Android层

#### 准备

环境配置相关的内容就不介绍了

创建一个空项目

![1](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/1.png)

下面配置可以随便写，不会用到，但还是规范一点吧！

![1](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/2.png)

创建项目，等待Gradle Sync，切记不要使用代理，不要使用代理，也不要开梯子！

右键项目，创建一个new module，选择library

这里的package name要和unity中设置一致，API等级大于11

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/7.png)

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/8.png)

然后我们把创建工程自带的app module删掉，当然直接删是删不掉的，得在这

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/10.png)

同事删除MySDK的所有依赖

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/11.png)

删除安卓自带的一些资源文件，只留下main就好

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/16.png)

到此为止安卓方面的一个空项目算是准备好了

#### 编写SDK

安卓肯定是不认识UNIYT的，所以我们得导入UNITY官方给我们的一个JAR包作为桥梁，找到对应编辑器的安装目录

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/12.png)

这其中有2个文件夹，得看你的UNITY项目选择打包成mono还是il2cpp，这里我们选择mono，则继续往下找

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/13.png)

将他复制到AS对应目录，并添加为Library

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/14.png)

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/15.png)

在main目录中的package中创建一个JAVA类，集成自Fragment，有同名类，注意命名空间

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/17.png)

编写代码

```java
package com.banging.mysdk;

import android.app.Fragment;
import android.os.Bundle;

import com.unity3d.player.UnityPlayer;

public class MyPlugin extends Fragment
{
    private static final String TAG = "MyPlugin";
    private static MyPlugin Instance = null;
    private String gameObjectName;

    public static MyPlugin GetInstance(String gameObject)
    {
        if(Instance == null)
        {
            Instance = new MyPlugin();
            Instance.gameObjectName = gameObject;
            // 大概是默认写法，这里不是很明白
            UnityPlayer.currentActivity.getFragmentManager().beginTransaction().add(Instance, TAG).commit();
        }
        return Instance;
    }

    @Override
    public void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setRetainInstance(true);  // 这一句很重要，保存对该Fragment的引用，防止在旋转屏幕等操作时时丢失引用（Fragment隶属于Activity）
    }
    
    //示例方法一：简单的向Unity回调  
    public void SayHello()
    {
        UnityPlayer.UnitySendMessage(gameObjectName,"PluginCallBack","Hello Unity!");
    }
    
    //示例方法二：计算传入的参数并返回计算结果
    public int CalculateAdd(int one, int another)
    {
        return one + another;
    }
}
```

很简单的能够发现，这是一个单利类，提供给Unity调用

Build项目，找到编译好的arr，复制到Unity项目的Assets/Plugins/Android中

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/18.png)

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/21.png)

### Unity层

用解压缩软件打开arr，删除两个东西

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/19.png)

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/20.png)

新建一个测试场景

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/22.png)

新建一个脚本挂到Canvas上

```c#
using System;
using UnityEngine;
using UnityEngine.UI;
 
public class Test : MonoBehaviour
{
    public Text callbackText = null;
    public Text resultText = null;
    

    private AndroidJavaClass jc;
    private AndroidJavaObject jo;
 
    void Start()
    {
        // 这里就是我们的定义在安卓中的的包名+类名
        // 获取到一个类，只是类，不是实例
        jc = new AndroidJavaClass("com.banging.mysdk.MyPlugin");
        // 调用静态方法获取单例类
        jo = jc.CallStatic<AndroidJavaObject>("GetInstance", gameObject.name);

        try
        {
            jo.Call("SayHello");                                                                               
        }
        catch (Exception e)
        {
            callbackText.text = e.ToString();
        }

        try
        {
            resultText.text = jo.Call<int>("CalculateAdd", 2, 3).ToString(); 
        }
        catch (Exception e)
        {
            resultText.text = e.ToString();
        }
    }
 
    // 安卓向Unity发送的消息
    public void PluginCallBack(string text)
    {
        callbackText.text = text;
    }
}
```

<img src="https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/23.png" alt="3" style="zoom:50%;" />

打包设置，公司名必须改一个，使用默认的可能打包出错

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/24.png)

这两个需要和安卓那边设置一样，不一样好像也可以，没有过多测试，目前先保持一致！

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/25.png)

打包，测试

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/26.png)

### 更新

现在可以直接把JAVA文件丢到Android/Plugins中...根本不需要AS打包...

### 易错点

#### Gradle Sync 特别慢

不要使用代理，首先检查是否打开了代理

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/3.png)

这里分两种情况，如果你之前曾经设置过代理，哪怕你取消了，仍然会有残留配置，检查gradle.properties，图中红框内应该会有对应的代理信息，删掉

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/4.png)

其次，如果你配置过代理并且主动Sync过，AS应该会有一个弹窗让你确定代理设置，此时会在 C:\用户\\(你的用户名)\\.gradle 中生成一个gradle.properties，其中也会有代理信息，删掉，然后重启AS，注意一定要重启

如果你之前没有配置过代理，但是Build也很慢，那可以考虑随便设置一个代理，然后主动Sync，让AS保留代理配置，生成相关文件，然后你再删掉，重复上一步的操作，然后重启

#### Android Gradle plugin requires Java 11 to run.

![3](https://www.logarius996.icu/images/UnityStudy/Unity与Android通信/5.png)

本地JAVA环境版本太低了，直接点击图中的Gradle Settings设置成对应版本就好，似乎AS默认就是安装了JAVA11的

![3](D:\__Project\Gasskin\images\UnityStudy\Unity与Android通信\6.png)
