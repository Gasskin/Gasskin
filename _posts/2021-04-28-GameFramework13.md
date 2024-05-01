---
title: "GameFramework|13 数据节点"
categories: GameFramework 
tags: 框架
---

用于管理游戏运行时的各种数据信息，以一种树状结构储存

新建如下两个脚本

![1](https://www.logarius996.icu/images/SimpleGameFramework/DataNode/1.png)

其中DataNode是数据节点类，Manager就是管理器了

### DataNode

#### 字段与属性

```c#
namespace SimpleGameFramework.DataNode
{
    public class DataNode
    {
        #region Field

            public static readonly DataNode[] s_EmptyArray = new DataNode[] { };
            public static readonly string[] s_PathSplit = new string[] {".", "/", "\\"};

            /// <summary>
            /// 节点数据
            /// </summary>
            public object m_Data;

            /// <summary>
            /// 子节点
            /// </summary>
            public List<DataNode> m_Childs;

        #endregion

        #region Property
        
            /// <summary>
            /// 父节点
            /// </summary>
            public DataNode Parent { get; private set; }

            /// <summary>
            /// 节点名称
            /// </summary>
            public string Name { get; private set; }

            /// <summary>
            /// 节点全名
            /// </summary>
            public string FullName
            {
                get
                {
                    return Parent == null ? Name : string.Format("{1}{2}{3}", Parent.FullName, s_PathSplit[0], Name);
                }
            }

            /// <summary>
            /// 子节点数量
            /// </summary>
            public int ChildCount
            {
                get
                {
                    return m_Childs != null ? m_Childs.Count : 0;
                }
            }

        #endregion
    }
}
```

其实和链表的构造很像，唯一需要注意的是属性FullName，仔细观察可以发现，FullName是递归调用的，需要注意

#### 检查节点名称是否合法

```c#
namespace SimpleGameFramework.DataNode
{
    public class DataNode
    {
        #region Tool

            /// <summary>
            /// 检测数据结点名称是否合法。
            /// </summary>
            /// <param name="name">要检测的数据节点名称。</param>
            /// <returns>是否是合法的数据结点名称。</returns>
            private static bool IsValidName(string name)
            {
                if (string.IsNullOrEmpty(name))
                    return false;
                foreach (string pathSplit in s_PathSplit)
                {
                    if (name.Contains(pathSplit))
                        return false;
                }
                return true;
            }

        #endregion
    }
}
```

#### 构造方法

```c#
namespace SimpleGameFramework.DataNode
{
    public class DataNode
    {
        #region 生命周期

        public DataNode(string name, DataNode parent = null)
        {
            if (!IsValidName(name))
                Debug.LogError("数据结点名字不合法：" + name);
 
            Name = name;
            m_Data = null;
            Parent = parent;
            m_Childs = null;
        }

        #endregion
    }
}
```

#### 接口方法

```c#
namespace SimpleGameFramework.DataNode
{
    public class DataNode
    {
        #region 接口

        /// <summary>
        /// 获取结点数据
        /// </summary>
        public T GetData<T>()
        {
            return (T)m_Data;
        }
 
        /// <summary>
        /// 设置结点数据
        /// </summary>
        public void SetData(object data)
        {
            m_Data = data;
        }
        
        /// <summary>
        /// 根据索引获取子数据结点
        /// </summary>
        /// <param name="index">子数据结点的索引</param>
        /// <returns>指定索引的子数据结点，如果索引越界，则返回空</returns>
        public DataNode GetChild(int index)
        {
            return index >= ChildCount ? null : m_Childs[index];
        }
 
        /// <summary>
        /// 根据名称获取子数据结点
        /// </summary>
        /// <param name="name">子数据结点名称</param>
        /// <returns>指定名称的子数据结点，如果没有找到，则返回空</returns>
        public DataNode GetChild(string name)
        {
            if (!IsValidName(name))
            {
                Debug.LogError("子结点名称不合法，无法获取");
                return null;
            }
 
            if (m_Childs == null)
                return null;
 
            foreach (DataNode child in m_Childs)
            {
                if (child.Name == name)
                {
                    return child;
                }
            }
 
            return null;
        }
 
        /// <summary>
        /// 根据名称获取或增加子数据结点
        /// </summary>
        /// <param name="name">子数据结点名称</param>
        /// <returns>指定名称的子数据结点，如果对应名称的子数据结点已存在，则返回已存在的子数据结点，否则增加子数据结点</returns>
        public DataNode GetOrAddChild(string name)
        {
            DataNode node = GetChild(name);
            if (node != null)
                return node;
 
            node = new DataNode(name, this);
 
            if (m_Childs == null)
                m_Childs = new List<DataNode>();
 
            m_Childs.Add(node);
 
            return node;
        }
        
        /// <summary>
        /// 根据索引移除子数据结点
        /// </summary>
        /// <param name="index">子数据结点的索引位置</param>
        public void RemoveChild(int index)
        {
            DataNode node = GetChild(index);
            if (node == null)
                return;
 
            node.Clear();
            m_Childs.Remove(node);
        }
 
        /// <summary>
        /// 根据名称移除子数据结点
        /// </summary>
        /// <param name="name">子数据结点名称</param>
        public void RemoveChild(string name)
        {
            DataNode node = GetChild(name);
            if (node == null)
            {
                return;
            }
 
            node.Clear();
            m_Childs.Remove(node);
        }
 
        /// <summary>
        /// 移除当前数据结点的数据和所有子数据结点
        /// </summary>
        public void Clear()
        {
            m_Data = null;
            if (m_Childs != null)
            {
                foreach (DataNode child in m_Childs)
                {
                    child.Clear();
                }
 
                m_Childs.Clear();
            }
        }

        #endregion

    }
}
```

其实就是添加节点和删除节点，很简单，看代码就懂了

### DataNodeManager

首先实现ManagerBase，添加一些字段

```c#
namespace SimpleGameFramework.DataNode
{
    public class DataNodeManager : ManagerBase
    {
        #region Implement

            public override int Priority { get { return ManagerPriority.DataNodeManager.GetHashCode(); } }

            public override void Init()
            {
                Root = new DataNode(RootName, null);
            }

            public override void Update(float time)
            {

            }

            public override void ShutDown()
            {
                Root.Clear();
                Root = null;
            }

        #endregion

        #region Filed

            private static readonly string[] s_EmptyStringArray = new string[] { };

            /// <summary>
            /// 根节点名称
            /// </summary>
            private const string RootName = "<Root>";

        #endregion

        #region Property

            /// <summary>
            /// 根节点
            /// </summary>
            public DataNode Root { get; private set; }

        #endregion
    }
}
```

#### 切割节点路径的工具方法

```c#
namespace SimpleGameFramework.DataNode
{
    public class DataNodeManager : ManagerBase
    {
        #region 工具

            /// <summary>
            /// 数据结点路径切分
            /// </summary>
            /// <param name="path">要切分的数据结点路径</param>
            /// <returns>切分后的字符串数组</returns>
            private static string[] GetSplitPath(string path)
            {
                if (string.IsNullOrEmpty(path))
                {
                    return s_EmptyStringArray;
                }

                return path.Split(DataNode.s_PathSplit, StringSplitOptions.RemoveEmptyEntries);
            }

        #endregion
    }
}
```

关于这里的Split，[具体可以看这里]([C#的String.Split方法 - linFen - 博客园 (cnblogs.com)](https://www.cnblogs.com/luluping/archive/2009/04/30/1446654.html))，一看就懂这个方法在干啥了

#### 添加节点

```c#
namespace SimpleGameFramework.DataNode
{
    public class DataNodeManager : ManagerBase
    {
        #region 接口

        /// <summary>
        /// 获取数据结点。
        /// </summary>
        /// <param name="path">相对于 node 的查找路径</param>
        /// <param name="node">查找起始结点</param>
        /// <returns>指定位置的数据结点，如果没有找到，则返回空</returns>
        public DataNode GetNode(string path, DataNode node = null)
        {
            DataNode current = (node ?? Root);
 
            // 获取子结点路径的数组
            string[] splitPath = GetSplitPath(path);
 
            foreach (string ChildName in splitPath)
            {
                // 根据数组里的路径名获取子结点
                current = current.GetChild(ChildName);
                if (current == null)
                {
                    return null;
                }
            }
 
            return current;
        }
 
        /// <summary>
        /// 获取或增加数据结点
        /// </summary>
        /// <param name="path">相对于 node 的查找路径</param>
        /// <param name="node">查找起始结点</param>
        /// <returns>指定位置的数据结点，如果没有找到，则增加相应的数据结点</returns>
        public DataNode GetOrAddNode(string path, DataNode node = null)
        {
            DataNode current = (node ?? Root);
            string[] splitPath = GetSplitPath(path);
            foreach (string childName in splitPath)
            {
                current = current.GetOrAddChild(childName);
            }
 
            return current;
        }
        
        #endregion
    }
}
```

GetNode这里需要解释一下，首先我们目前的数据结构是这样的

![1](https://www.logarius996.icu/images/SimpleGameFramework/DataNode/2.png)

注重强调，每一个DataNode内部储存的都是一个List<DataNode>，理解了这点，也就理解了全部

假设我们的Path是 Player.Property.HP，从Root开始搜索

那么我们会现在Root的List<DataNode>中找到Player节点，然后从Player的List<DataNode>中找到Property，最后从Property中找到HP

这里的foreach循环，起到的作用类似于递归搜索

#### 删除节点

```c##
namespace SimpleGameFramework.DataNode
{
    public class DataNodeManager : ManagerBase
    {
         #region 接口
             
        /// <summary>
        /// 移除数据结点
        /// </summary>
        /// <param name="path">相对于 node 的查找路径</param>
        /// <param name="node">查找起始结点</param>
        public void RemoveNode(string path, DataNode node = null)
        {
            DataNode current = (node ?? Root);
            DataNode parent = current.Parent;
            string[] splitPath = GetSplitPath(path);
            foreach (string childName in splitPath)
            {
                parent = current;
                current = current.GetChild(childName);
                if (current == null)
                {
                    return;
                }
            }
 
            if (parent != null)
            {
                parent.RemoveChild(current.Name);
            }
        }
        
        #endregion
    }
}
```

#### 获取和设置数据节点

```c##
namespace SimpleGameFramework.DataNode
{
    public class DataNodeManager : ManagerBase
    {
        #region 接口

        /// <summary>
        /// 根据类型获取数据结点的数据
        /// </summary>
        /// <typeparam name="T">要获取的数据类型</typeparam>
        /// <param name="path">相对于 node 的查找路径</param>
        /// <param name="node">查找起始结点</param>
        public T GetData<T>(string path, DataNode node = null)
        {
            DataNode current = GetNode(path, node);
            if (current == null)
            {
                Debug.Log("要获取数据的结点不存在：" + path);
                return default(T);
            }
 
            return current.GetData<T>();
 
        }
 
        /// <summary>
        /// 设置数据结点的数据。
        /// </summary>
        /// <param name="path">相对于 node 的查找路径。</param>
        /// <param name="data">要设置的数据</param>
        /// <param name="node">查找起始结点</param>
        public void SetData(string path, object data, DataNode node = null)
        {
            DataNode current = GetOrAddNode(path, node);
            current.SetData(data);
        }
        
        #endregion
    }
}
```

### 测试

```c##
public class test : MonoBehaviour
{
    void Start()
    {
        var dataNodeManager = SGFEntry.Instance.GetManager<DataNodeManager>();
        
        // 绝对路径设置数据
        dataNodeManager.SetData("Player.Property.Hp",100);
        
        // 相对路径设置数据
        var node = dataNodeManager.GetOrAddNode("Player.Message");
        dataNodeManager.SetData("Name", "logarius996", node);
        
        // 直接通过数据节点设置数据
        var exp = dataNodeManager.GetOrAddNode("Player.Message.Exp");
        exp.SetData(1000);

        // 获取数据
        string name = dataNodeManager.GetNode("Player.Message.Name").GetData<string>();
        int hp = dataNodeManager.GetNode("Player.Property.Hp").GetData<int>();
        int exps = dataNodeManager.GetNode("Player.Message.Exp").GetData<int>();

        Debug.Log($"{name} {hp} {exps}");
    }

}
```

![1](https://www.logarius996.icu/images/SimpleGameFramework/DataNode/3.png)





























