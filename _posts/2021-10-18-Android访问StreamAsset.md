---
title: "Android访问StreamAsset"
tags: SDK
---

安卓平台下JAR包变成了压缩包，直接通过File来读取StreamAsset下的文件是不行的

通过UnityWebRequest来读取，以某一个XML文件为例

```c#
var             path    = Path.Combine (Application.streamingAssetsPath, "xxx");// 使用Combine时，路径前不能带有'/'，直接以文件夹开头
UnityWebRequest request = UnityWebRequest.Get (path);
request.SendWebRequest ();
while (!request.downloadHandler.isDone) { } // 这里可以用携程，现在是同步加载
if (request.result == UnityWebRequest.Result.Success)
{
    TextReader reader   = new StringReader (request.downloadHandler.text);
    XDocument  document = XDocument.Load (reader);
	...
}
```

