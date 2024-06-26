---
title: "GameFramework|04 UI管理器02"
categories: GameFramework 
tags: 框架
---

### UI的注册

好了，在上一节结尾，我们已经能打开一个UI了，那么我们的管理器完成了吗？

是的，完成了......个锤子！

最直观的问题是，我们现在打开的UI是固定的，废话，因为Resources加载的路径都被写死了......

```c#
public void OpenUI()
{
    /// 路径是死的！！
    GameObject go = Resources.Load<GameObject>("UI/TestUI");
    var prefab = GameObject.Instantiate(go);
    prefab.transform.SetParent(normalRoot,false);
}
```

所以第一个问题：如何保存我们UI Prefab的路径？

最简单的想法，把所有UI的路径写在一个静态类中，当然，从本质上来说，只有这一种方法，因为我们必然需要UI的路径...所以问题不在这，问题在于，怎么更好更简单的保存这个路径？当然你要是觉得手写没问题...其实也行...希望策划想改路径的时候，你看到那几百个UI，你不会想杀了他......

对于这个问题我没有很深入的研究，所以也只是提供目前我自己的一个解决方案以供参考

首先我们在Config中新建一个enum叫做UIRegister

```c#
/// <summary>
/// UI的注册
/// </summary>
public enum UIRegister
{
    TestUI,
}
```

当然我们现在只有一个TestUI，所以我们只写了一个...等之后有更多的UI了，我们都要把他们注册到这里...

注册了之后有啥用呢？那就是———

**代码自动生成！**

这是我很喜欢的一个方式，本质就是通过C#读写文本文件，来自动生成一些类，当然这是我自己考虑的方法， 因为我也不知道还有什么其他方法能够自动生成类，通过IO接口来读写文件，我觉得是比较“朴素”的做法，因为确实没啥算法上技术含量，也没什么黑魔法...就是很朴素的读，然后很朴素的写...如果大佬们有什么比较吊的方案希望能告诉我！！

好了废话不多说，咱们来写这个类吧！因为要把这个自动生成的方法扩展给Unity，所以我们新建一个Editor脚本

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/11.png)

添加一个工具方法如下

```c#
public static class GenerateUIPath
{
    [MenuItem("SGFTool/UI/GenerateUIPath")]
    public static void Generate()
    {
        Debug.Log(111);
    }
}
```

然后Unity编辑器上就会有一个对应的方法了~当然你可以放在自己想要的位置，主要必须是静态类和静态方法！

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/12.png)

之后就是自动生成算法了，其实这个算法就很简单了，我们在上面不是有一个UIRegiser嘛，只要遍历这个enum，然后给每一个enum都生成一个路径就好啦，但是个人感觉这样还是有一些死，因为很多路径都必须要定死，没法随便，但目前没想到其他方法，暂时只能如此~

```c#
public static class GenerateUIPath
{
    [MenuItem("SGFTools/UI/GenerateUIPath")]
    public static void Generate()
    {
        // Application.dataPath是Assets的路径
        var path = Application.dataPath + "/SimpleGameFramework/UI/Generated";
        // 用来给txt文件填充字符串
        StringBuilder sb = new StringBuilder();
        // 获取目前注册过的UI的名字
        var names = Enum.GetNames(typeof(UIRegister));
        // 如果不存在目录，新建一个
        if (!Directory.Exists(path))
        {
            Directory.CreateDirectory(path);
        }

        sb.AppendLine("//==========================================");
        sb.AppendLine("// 这个文件是自动生成的...");
        sb.AppendLine($"// 生成日期：{DateTime.Now.Year}年{DateTime.Now.Month}月{DateTime.Now.Day}日{DateTime.Now.Hour}点{DateTime.Now.Minute}分");
        sb.AppendLine("//");
        sb.AppendLine("//");
        sb.AppendLine("//==========================================");
        sb.AppendLine();
        sb.AppendLine();
        sb.AppendLine("public struct UIStruct");
        sb.AppendLine("{");
        sb.AppendLine("    public string name;");
        sb.AppendLine("    public string path;");
        sb.AppendLine("    public UIStruct(string name,string path)");
        sb.AppendLine("    {");
        sb.AppendLine("        this.name = name;");
        sb.AppendLine("        this.path = path;");
        sb.AppendLine("    }");
        sb.AppendLine("}");
        sb.AppendLine();
        sb.AppendLine();
        sb.AppendLine("public static class UIs");
        sb.AppendLine("{");
        for (int i = 0, imax = names.Length; i < imax; i++)
        {
            sb.AppendLine($"    public static UIStruct {names[i]} = new UIStruct(\"{names[i]}\",\"UI/{names[i]}\");");
        }
        sb.AppendLine("}");

        var wirtePath = path + "/UIPath.cs";
        File.WriteAllText(wirtePath,sb.ToString(),Encoding.UTF8);
        
        AssetDatabase.Refresh();
    }
}
```

好了，回到编辑器，点击这个方法，然后对应目录应该就会生成对应的cs文件~

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/13.png)

![1](https://www.logarius996.icu/images/SimpleGameFramework/UI/14.png)

然后把我们的OpenUI方法也改了，先简单测试一下

```c#
public void OpenUI()
{
    GameObject go = Resources.Load<GameObject>(UIPath.TESTUI);
    var prefab = GameObject.Instantiate(go);
    prefab.transform.SetParent(normalRoot,false);
}
```

运行~效果是一样的，我就不放截图啦，下一节正式介绍打开UI



























