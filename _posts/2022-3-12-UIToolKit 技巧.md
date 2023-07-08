---
title: "UIToolKit|03 技巧"
tags: UGUI
---

# 3D坐标转2D坐标

**RuntimePanelUtils.CameraTransformWorldToPanel**

随便新建一个血条

```c#
public class Test : MonoBehaviour
{
    public UIDocument document;
    public Transform head;
    
    private VisualElement visualElement;
    
    private void Start()
    {
        visualElement = document.rootVisualElement.Q<VisualElement>("hp");
    }

    private void Update()
    {
        visualElement.transform.position =
            RuntimePanelUtils.CameraTransformWorldToPanel(visualElement.panel, head.position, Camera.main);
    }
}
```

# 拖动事件

新建一个300x300的图片

![image-20220314154532999](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203141545057.png)

```c#
public class Test : MonoBehaviour
{
    public UIDocument document;

    private VisualElement visualElement;
    private bool moving;

    private void Start()
    {
        visualElement = document.rootVisualElement.Q<VisualElement>("hp");
        
        visualElement.RegisterCallback<PointerDownEvent>(OnPointerDown);
        visualElement.RegisterCallback<PointerMoveEvent>(OnPointerMove);
        visualElement.RegisterCallback<PointerUpEvent>(OnPointerUp);
    }

    private void OnPointerDown(PointerDownEvent evt)
    {
        moving = true;
    }

    private void OnPointerMove(PointerMoveEvent evt)
    {
        if (moving)
        {
            // Unity屏幕坐标的原点是左下角，而UIToolKit的原点是左上角，也就是说Y是相反的
            var pos = new Vector2(Input.mousePosition.x, Screen.height - Input.mousePosition.y);
            //转换自适应后的UI坐标
            pos = RuntimePanelUtils.ScreenToPanel(visualElement.panel, pos);

            //图片尺寸是300，这里取中心点
            float widthAndHeight = 300f;
            pos.x -= widthAndHeight / 2f;
            pos.y -= widthAndHeight / 2f;

            //设置坐标
            visualElement.transform.position = pos;
        }
    }

    private void OnPointerUp(PointerUpEvent evt)
    {
        moving = false;
    }
}
```

# 动态设置样式

使用UGUI时，如果某一个元素需要改变多个样式，比如颜色，坐标，显隐等，那么我们可能需要写很多if else

```c#
if(??)
{
    button.color = xxx;
    image.position = xxx;
    gameobject.active = false;
}
else
{
    button.color = xxx;
    image.position = xxx;
    gameobject.active = true;
}
```

在UIToolKit中，则可以直接采用两套样式设计

新建一个简单的UXML

![image-20220314170137026](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203141701074.png)

创建第一个USS

![img](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203141704209.png)

#text，代表选中所有名称为text的组件

还有其他很多种索引方式

![image-20220314170504098](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203141705136.png)

再创建第二个USS

![image-20220314170525249](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203141705285.png)

回到代码里，很简单

```c#
public class Test : MonoBehaviour
{
    public UIDocument document;
    public StyleSheet sheet1;
    public StyleSheet sheet2;

    private void Start()
    {
        var root = document.rootVisualElement;

        root.Q<Button>("btn1").clicked += () =>
        {
            root.styleSheets.Remove(sheet2);
            root.styleSheets.Add(sheet1);
        };
        
        root.Q<Button>("btn2").clicked += () =>
        {
            root.styleSheets.Remove(sheet1);
            root.styleSheets.Add(sheet2);
        };
    }
}
```

![1](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203141707081.gif)

不过目前USS好像对VisualElement不生效

# 界面动画

相当于内置了DoTween

```c#
public class Test : MonoBehaviour
{
    public UIDocument document;

    private void Start()
    {
        var root = document.rootVisualElement;

        var text = root.Q<Label>("text");

        root.Q<Button>("btn1").clicked += () =>
        {
            text.experimental.animation.Scale(2f, 2000).Ease(Easing.InSine);
        };
        
    }
}
```

但是比较有限，目前来看和DoTween差远了

# 编辑器界面

首先可以打开编辑器专用脚本

![image-20220314172054959](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203141720981.png)

然后随便拼一个简单UXML

![image-20220314173947745](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203141739769.png)

脚本如下

```c#
public class TestEditor : EditorWindow
{
    [MenuItem("UIToolkit/Test")]
    public static void GetWindow()
    {
        TestEditor wnd = GetWindow<TestEditor>();
        wnd.titleContent = new GUIContent("ShowExample16");
    }
    public void OnEnable()
    {
        var visualTree = AssetDatabase.LoadAssetAtPath<VisualTreeAsset>("Assets/UXML.uxml");
        rootVisualElement.Add(visualTree.Instantiate());

        rootVisualElement.Q<Button>().clicked += () =>
        {
            Debug.Log(rootVisualElement.Q<IntegerField>().value);
        };
    }
}
```

# Inspector界面

随便写一个Mono脚本

```c#
public class Test : MonoBehaviour
{
    public int a;
    public bool b;
    public GameObject c;
}
```

拼一个对应的UXML

![image-20220314174840247](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203141748287.png)

Binding Path就是代码中的变量名

对于Object这样的引用类型，还需要设置他的Type，和我们自己写编辑器脚本一样，ObjectField要有TypeOf

然后就大功告成了

![image-20220314175012521](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203141750541.png)

























