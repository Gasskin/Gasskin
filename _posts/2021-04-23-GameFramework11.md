---
title: "GameFramework|11 多语言"
categories: GameFramework 
tags: 框架
---

多语言也是很常用的一个功能，如果游戏想要支持多种语言，那就不得不设计一个多语言的框架。比如中文的“确定”，英文要叫“OK”，你不可能创建多个语言版本的UI，那会十分繁琐，难不成有二十种语言，你就创建二十种UI？那也太折磨人了...

目前只考虑到文字的多语言，实际上还有其他资源也需要支持本地化，比如众所周知国内的血是绿色的，而国外是红色的，这其实也算是一种本地化...只不过对象不是文字，而是动画或者贴图资源

但目前我这里只学习了文字相关的本地化

其实现很简单，本质是靠CSV表配置，我们给每个Text组件额外加上一个ID，然后根据ID去对应的语言CSV里找对应的语言文字就好

### CSV的读取规则

读表是很常见的需求，但我们还是必须制定好表格的规则，首先，我们的KEY单独为一张表，这张表里指定了所有的关键字，而我们之后查找对应文字，就是依靠的KEY

![1](https://www.logarius996.icu/images/SimpleGameFramework/Language/1.png)

第一列是关键字，**行号代表序列号**，关键字会被自动生成为枚举类型，而它的行号会被生成为枚举的序列号

第二列是备注

注意，行号代表序列号，我这里是靠行号来实现关键字与语言的对应的，比如我现在有一张中文表

![1](https://www.logarius996.icu/images/SimpleGameFramework/Language/2.png)

你会发现，第二行取消，对应了KEY表中的第二行Cancel，第五行语种对应了KEY表中的第五行Language

这必须是一一对应的关系！不然两张表要如何联系到一起呢？

当然有小伙伴会觉得这么写会很麻烦，可以把语言和KEY写到同一张CSV表中，那当然是没有问题而且也确实更清楚，但是一般来说，我们的游戏只会加载一种语言，如果你把所有的语言都写到同一张表里，那加载的时候岂不是会把所有语言都加载一遍？

好我们再新建一张英文表，目前一共三张表，如下

![1](https://www.logarius996.icu/images/SimpleGameFramework/Language/3.png)

### 自动生成LanguageID

自动生成咱们应该已经轻车熟路了，新建一个Editor脚本

![1](https://www.logarius996.icu/images/SimpleGameFramework/Language/4.png)

csv格式是以逗号分隔列的，知道这一点我们就很轻松的能够获取信息了

```c#
public static class Generate
{
        // LanguageID.csv 所在位置
        public static string filePath = Application.dataPath + "/SimpleGameFramework/Language/CSV/LanguageID.csv";
        // 自动生成的保存路径
        public static string generatePath = Application.dataPath + "/SimpleGameFramework/Language/Generated/LanguageID.cs";
        
        [MenuItem("SGFTools/Language/Generate LanguageID.cs")]
        public static void GenerateLanguageID()
        {
            StringBuilder sb = new StringBuilder();
            
            // 打开csv读取所有字符
            var datas = File.ReadAllLines(filePath, Encoding.UTF8);

            sb.AppendLine("//==========================================");
            sb.AppendLine("// 这个文件是自动生成的...");
            sb.AppendLine($"// 生成日期：{DateTime.Now.Year}年{DateTime.Now.Month}月{DateTime.Now.Day}日{DateTime.Now.Hour}点{DateTime.Now.Minute}分");
            sb.AppendLine("//");
            sb.AppendLine("//");
            sb.AppendLine("//==========================================");
            sb.AppendLine();
            sb.AppendLine();
            sb.AppendLine("namespace SimpleGameFramework.Language.Editor");
            sb.AppendLine("{");
            sb.AppendLine("    public enum LanguageID");
            sb.AppendLine("    {");
            sb.AppendLine("        Invalid = 0,");
            sb.AppendLine();
            for (int i = 0; i < datas.Length; i++)
            {
                if (string.IsNullOrEmpty(datas[i])) 
                    continue;
                
                var temp = datas[i].Split(',');
                if (string.IsNullOrEmpty(temp[0]))
                    continue;
                
                var enumName = temp[0].Substring(0, 1).ToUpper() + temp[0].Substring(1).ToLower();
                sb.AppendLine($"        /// {temp[1]}");
                sb.AppendLine($"        {enumName} = {i + 1},");
                sb.AppendLine();
            }
            sb.AppendLine("    }");
            sb.AppendLine("}");

            File.WriteAllText(generatePath, sb.ToString());

            AssetDatabase.Refresh();
}
```

运行过后会生成如下文件

![1](https://www.logarius996.icu/images/SimpleGameFramework/Language/5.png)

### 扩展Text组件

现在我们已经有了文字的KEY值了，那么如何与对应的Text组件绑定呢？

在这里，我们额外扩展一个组件TextID，用这个组件来作为我们KEY值和语言的中间件

```c#
[DisallowMultipleComponent]
[RequireComponent(typeof(Text))]
public class TextID : MonoBehaviour
{
        #region Field

        public Text text;
        public LanguageID id;

        #endregion

        #region Mono

        private void Awake()
        {
            text = GetComponent<Text>();
            if (text == null)
                throw new Exception("挂在Text组件失败...");
            id = LanguageID.Invalid;
        }

        #endregion
}
```

然后咱们重写一下TextID的编辑器

```c#
[CustomEditor(typeof(TextID))]
public class TextIDEditor : UnityEditor.Editor
{
        /// <summary>
        /// 添加Text组件
        /// </summary>
        [MenuItem ("GameObject/UI/TextID")]
        private static void CreateTextMultiLanguage ()
        {
            var go = new GameObject {name = "Text"};
            go.transform.SetParent (Selection.activeTransform, false);
            go.AddComponent<TextID> ();
            Selection.activeObject = go;
        }
        
        private TextID inst => target as TextID;

        public override void OnInspectorGUI()
        {
            inst.id = (LanguageID)EditorGUILayout.EnumPopup("ID", inst.id);
        }
}
```

此时右键UI中会有一个新的组件，咱们创建一下试试，应该如下，会自动添加Text组件

![1](https://www.logarius996.icu/images/SimpleGameFramework/Language/6.png)

好了，到此为主，我们算是可以给我们的Text绑定ID了，但是文字呢？文字呢？

...

文字其实还没读CSV...

### LanguageManager

Manager用于管理语种的切换，但在此之前，我们还需要有一个地方能够读取我们的语种CSV

```c#
public class LanguageReader
{
        /// <summary>
        /// 读取对应语种的字典，不存在时，读取 LanguageManager.DEFAULT_LANGUAGE 的语种字典
        /// </summary>
        public void Read (Dictionary<LanguageID, string> dic, string language)
        {
            // 目标语种
            var filePath = Application.dataPath + $"/SimpleGameFramework/Language/CSV/Languages/{language}.csv";

            // 如果目标语种不存在，那读取默认语种，注意，我们的CSV表的表名，一定要和SystemLanguage的枚举类型名称一致
            if (!File.Exists(filePath))
                filePath = Application.dataPath + $"/SimpleGameFramework/Language/CSV/Languages/{LanguageManager.DEFAULT_LANGUAGE.ToString ()}.csv";
            if (!File.Exists(filePath))
                throw new Exception($"不存在对应语种{language}的CSV表，也不存在对应默认语种{LanguageManager.DEFAULT_LANGUAGE.ToString()}的CSV表...");

            dic.Clear ();
            dic.Add(LanguageID.Invalid, "#Invalid#");

            StreamReader sr = new StreamReader(filePath);

            int i = -1;
            var data = sr.ReadLine();
            while (data != null) 
            {
                i++;
                if (!string.IsNullOrEmpty(data))
                {
                    dic.Add((LanguageID) (i + 1), data);
                }
                data = sr.ReadLine();
            }
        }
}
```

此方法会返回一个字典类型，里面就是我们所需语种的信息

好了然后我们来写我们的LanguageManager

```c#
public class LanguageManager:ManagerBase
{
        #region Filed

        public const SystemLanguage DEFAULT_LANGUAGE = SystemLanguage.English;
    
        private string currentLanguage;
        private LanguageReader languageReader;
        private StringBuilder  textMaker;
        
        /// 当前语种的字典，切换语种需要重新加载这个字典
        private Dictionary<LanguageID, string> textData;
        private const string CURRENT_LANGUAGE = "LanguageFramework.Current_Language";
        
        #endregion
        
        public override int Priority
        {
            get { return ManagerPriority.LanguageManager.GetHashCode(); }
        }

        #region 生命周期

        public override void Init()
        {
            // 首先我们会尝试获取本地的设置，看看本地是否有设置过的语言类型，如果有的话就以本地语种为准，没有的话就以系统默认语种为准
            currentLanguage = PlayerPrefs.GetString (CURRENT_LANGUAGE);
            if (string.IsNullOrEmpty(currentLanguage)) 
                currentLanguage = Application.systemLanguage.ToString();

            textMaker      = new StringBuilder ();
            languageReader = new LanguageReader ();
            textData       = new Dictionary<LanguageID, string> ();
            
            // 获取对应语种的信息
            languageReader.Read (textData, currentLanguage);
        }

        #endregion
}
```

设置语种的方法，需要注意的是，如果我们重新设置了语种，那么对应的Text应该也需要改变，所以这里需要注册一个事件， 并在TextID中监听

```c#
public class LanguageManager:ManagerBase
{
    #region Event
        public event Action onLanguageChange;
    #endregion  
    
    #region Language Method
        /// <summary>
        /// 设置语种，并刷新页面上的文字 
        /// </summary>
        public void SetCurrentLanguage (SystemLanguage newLanguage)
        {
            var language = newLanguage.ToString ();
            if (currentLanguage == language)
                return;

            PlayerPrefs.SetString (CURRENT_LANGUAGE, language);
            currentLanguage = language;
            textData.Clear ();
            languageReader.Read (textData, language);
            onLanguageChange?.Invoke ();
        }
    #endregion
    
    #region Text Method
        /// <summary>
        /// 获取与 TextID 对应的 String ，语种等于当前 language
        /// </summary>
        public string GetText (LanguageID id)
        {
            if (textData.TryGetValue (id, out var content))
                return content;

            return "#Invalid#";
        }

        /// <summary>
        /// 获取与 TextID 对应的 String ，语种等于当前 language
        /// </summary>
        public string GetText (LanguageID id, object arg0)
        {
            if (textData.TryGetValue (id, out var content))
            {
                textMaker.Clear ();
                textMaker.AppendFormat (content, arg0);
                return textMaker.ToString ();
            }

            return "#Invalid#";
        }

        /// <summary>
        /// 获取与 TextID 对应的 String ，语种等于当前 language
        /// </summary>
        public string GetText (LanguageID id, object arg0, object arg1)
        {
            if (textData.TryGetValue (id, out var content))
            {
                textMaker.Clear ();
                textMaker.AppendFormat (content, arg0, arg1);
                return textMaker.ToString ();
            }

            return "#Invalid#";
        }

        /// <summary>
        /// 获取与 TextID 对应的 String ，语种等于当前 language
        /// </summary>
        public string GetText (LanguageID id, object arg0, object arg1, object arg2)
        {
            if (textData.TryGetValue (id, out var content))
            {
                textMaker.Clear ();
                textMaker.AppendFormat (content, arg0, arg1, arg2);
                return textMaker.ToString ();
            }

            return "#Invalid#";
        }

        /// <summary>
        /// 获取与 TextID 对应的 String ，语种等于当前 language
        /// </summary>
        public string GetText (LanguageID id, params object[] args)
        {
            if (textData.TryGetValue (id, out var content))
            {
                textMaker.Clear ();
                textMaker.AppendFormat (content, args);
                return textMaker.ToString ();
            }

            return "#Invalid#";
        }
    #endregion
}
```

这几个GetText的作用是，有一些文字的格式会是{0}获得了{1}个{2}，像这样的文字，其中的0 1 2处可以自己填入信息

TextID中需要有对应的刷新方法

```c#
    public class TextID : MonoBehaviour
    {
        #region Field

        public Text text;
        public LanguageID id;

        private LanguageManager languageManager;
        #endregion

        #region Mono

        private void Awake()
        {
            languageManager = SGFEntry.Instance.GetManager<LanguageManager>();
            
            text = GetComponent<Text>();
            if (text == null)
                throw new Exception("挂在Text组件失败...");
            RefreshText ();
            languageManager.onLanguageChange += RefreshText;
        }

        private void OnDestroy()
        {
            languageManager.onLanguageChange -= RefreshText;
        }

        #endregion
        
        #region Event 方法
    
        private void RefreshText ()
        {
            if (text != null)
                text.text = languageManager.GetText (id);
        }
    
        #endregion
    }
```

测试一下，新建一个Text如下

![1](https://www.logarius996.icu/images/SimpleGameFramework/Language/7.png)

![1](https://www.logarius996.icu/images/SimpleGameFramework/Language/8.png)

运行游戏

![1](https://www.logarius996.icu/images/SimpleGameFramework/Language/9.png)

### 改进TextID的编辑器面板

到此为止其实工作已经完成了，但是在Editor模式下，咱们是没法看到我们选中的文字到底是啥的，所以稍微再改进一下编辑器

```c#
	[CustomEditor(typeof(TextID))]
    public class TextIDEditor : UnityEditor.Editor
    {
        private Dictionary<LanguageID, string> textData;

        protected void OnEnable ()
        {
            textData = new Dictionary<LanguageID, string> ();
            new LanguageReader ().Read (textData, SystemLanguage.ChineseSimplified.ToString());
            inst.text = inst.gameObject.GetComponent<Text> ();
        }

        public override void OnInspectorGUI()
        {
            inst.id = (LanguageID)EditorGUILayout.EnumPopup("ID", inst.id);
            
            if (GUI.changed)
            {
                if (inst.text != null)
                    inst.text.text = textData[inst.id];
                EditorUtility.SetDirty(target);
            }
        }
    }
```

好了，这下全部完成了，具体的测试就不截图了，应该木有问题

















