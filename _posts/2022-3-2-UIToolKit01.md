---
title: "UIToolKit|01 初识"
categories: UIToolKit
tags: Unity
---

# 基础使用

右键Asset -> Create -> UI Toolkit -> UI Document

![image-20220302161703212](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302161703212.png)

然后双击打开，可以进入UIBuilder面板

### 1.创建USS描述文件

![image-20220302164304080](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302164304080.png)

### 2.创建一个UI元素

![image-20220302164842371](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302164842371.png)

### 3.创建一个样式选择器

![image-20220302164919934](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302164919934.png)

选中USS描述文件，然后在右侧的选择器里创建

"test"属于修饰名称，没啥实际作用

可以以任意".x"结尾，"x"才属于是真正的名字，和CSS很像

![image-20220302165450696](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302165450696.png)

选中我们新建的样式选择器，再右侧Inspector面板可以调节属性，比如我们把背景设置为红色

![image-20220302165523113](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302165523113.png)

可以发现选择器的描述文件发生了变化

![image-20220302165542644](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302165542644.png)

### 4.创建focus和hover选择器

.x是默认选择器，我们可以通过".x:hover"和".x:focus"来实现特殊的聚焦和悬停功能

![image-20220302170446361](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302170446361.png)

并设置好不同的颜色

![image-20220302170519571](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302170519571.png)

### 5.将样式选择器绑定到UI元素

选中我们的元素

![image-20220302165251439](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302165251439.png)

​	在右侧样式选择器中可以发现已经和我们的.a绑定了

![image-20220302165745141](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302165745141.png)

### 6.预览

点击preview可以进入实时预览模式

![image-20220302170536726](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220302170536726.png)

可以发现我们的样式选择器都已经生效了

![动画](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/%E5%8A%A8%E7%94%BB.gif)

# 对比传统UGUI

我们随便创建一个UI，然后保存为Prefab

![image-20220304100835619](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220304100835619.png)

用文本编辑器打开本地的Prefab文件

```c#
%YAML 1.1
    %TAG !u! tag:unity3d.com,2011:
//...
```

尽管我们的Prefab里只有一个Image，但会有非常长的一串代码，非常长，就不全贴出来了...

因为UGUI是以传统GameObject的方式运行的，所以Prefab上的所有信息都需要序列化保存，哪怕我们不需要修改Image的坐标和颜色，这些信息也需要保存

同时我们观察一下UXML的代码量，只有这么一丢丢

```c#
<ui:UXML xmlns:ui="UnityEngine.UIElements" xmlns:uie="UnityEditor.UIElements" xsi="http://www.w3.org/2001/XMLSchema-instance" engine="UnityEngine.UIElements" editor="UnityEditor.UIElements" noNamespaceSchemaLocation="../UIElementsSchema/UIElements.xsd" editor-extension-mode="False">
    <Style src="project://database/Assets/USS/TestUSS.ussfileID=7433441132597879392&amp;guid=82b3f1e7acd825545bfe058793efb0bb&amp;
    type=3#TestUSS" />
    <ui:Button text="Button" 
        display-tooltip-when-elided="true" 
        class="a" 
        style="height: 71px;" />
</ui:UXML>
```

# 其他面板

### 替换Canvas背景

选中我们的UXML后可以设置当前的背景，设置为摄像机之后就和UGUI的显示一样了

![image-20220304101800318](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220304101800318.png)

![image-20220304101823787](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220304101823787.png)

### Inspector面板

以Laber组件为例

![image-20220304103108588](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220304103108588.png)

- **Name：**标识组件的名称，代码中通过这个名称来查找组件

- **View Data Key：**表示组件的唯一性

- **Picking Mode：**是否接受点击，设置为Ignore后无法接受点击事件

- **ToolTip：**人如其名，当hover时会显示一段提示

  ![image-20220304103545449](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220304103545449.png)

- **Usage Hints：**不影响显示效果，影响性能

  - **DynamicTransform：**经常改变的元素应该设置为此项
  - **GroupTransform：**类似于子Canvas的动静分离
  - **Mask Contianer：**官方文档缺失
  - **Dynamic Color：**官方文档缺失

- **TabIndex：**对于Focusable的焦点进行排序，大于0
- **BindingPath：**用于和Monobehaviour的序列化数据相绑定，主要用于脚本自定义面板与数据绑定

### StyleSheet

![image-20220304104752702](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220304104752702.png)

### InlinedStyles

这里包含了所有可设置属性

![image-20220304104844871](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220304104844871.png)

#### DisPlay

![image-20220304105646451](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220304105646451.png)

- **Opacity：**透明度

- **Display：**是否显示

- **Visibility：**是否可见，第一个选项表示显示，第二个选项表示不显示，但保留占位

  将Button1设置为不显示但可见

  ![image-20220304105923484](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220304105923484.png)

  将Button1设置为显示但不可见

  ![image-20220304131831701](https://raw.githubusercontent.com/Gasskin/CloudImg/master/img/image-20220304131831701.png)

- **Overflow：**裁剪，父节点对于子节点的裁剪

  将父节点设置为裁剪后：
  
  ![image-20220305202348485](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203052023533.png)

#### Position

- **Relative：**相对定位，是相对于他自己的Layout，再定位
- **Absolute：**绝对定位，和自己没关系，相对于父节点进行定位

#### Flex

伸缩性

- **Direction：**排列方向，针对于子物体

  向右排

  ![image-20220306154620331](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061546374.png)

  向下排

  ![image-20220306154642601](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061546634.png)

- **Wrap：**换行，针对于子物体，分别是不换行/向下换行/向上换行

  当Button3排不下的时候就会换行

  ![image-20220306154758666](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061547700.png)

- **Basis：**基础大小，针对于自身，意思是基础大小，根据父节点Direction的不同，会适当放大自身的宽/高

  如图Button1的Basis设置为了300，2和3都是auto，同时父节点的排布方向是竖直排布，所以Button1的高度被设置为了300

  ![image-20220306154945242](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061549284.png)

- **Shrink/Grow：**缩小权重/放大权重，当排布大小不够/盈余时，是否进行自动的缩小/放大

  如图，Button1和2的放大权重都是1，当画布大小有明显盈余时，他们两个就会同等放大

  ![image-20220306155206515](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061552549.png)

- **Background：**背景，已经支持使用Textrue/Sprite
  - **Scale Mode：**缩放选项：拉伸/裁剪/等比缩放

- **Transition Animations：**过渡动画，特别吊

  比如给背景色添加一个1秒的过渡动画

  ![image-20220306164333355](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061643377.png)

  ![1](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061646448.gif)

# 显示UI

首先需要创建PanelSetting文件

![image-20220306165257286](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061652307.png)

主要是这部分设置，和UGUI是一样的

![image-20220306165753565](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061657587.png)

然后我们创建一个GameObject，添加一个UIDocument的组件，拖入UXML和PanelSetting

![image-20220306165826542](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061658577.png)

这样就可以在Game视图可以看到UI了

![image-20220306165849144](https://cdn.jsdelivr.net/gh/Gasskin/CloudImg/img/202203061658186.png)
