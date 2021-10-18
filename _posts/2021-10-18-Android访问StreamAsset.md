---
layout: post
title: "Android访问StreamAsset"
categories: [Unity学习记录]
tags: [Unity,学习记录]
---

安卓平台下JAR包变成了压缩包，直接通过File来读取StreamAsset下的文件是不行的

通过UnityWebRequest来读取，以某一个XML文件为例

```c#
var             path    = Path.Combine (Application.streamingAssetsPath, "xxx");
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

