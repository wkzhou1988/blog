---
title: AssetBundle.LoadAsset的一些小细节
date: 2020-05-08 16:38:52
tags: 
- Unity
---
#### 反复调用LoadAsset会产生什么结果？

之所以这么问是因为我们知道同一个AssetBundle是不允许重复加载的。那么里面的资源是怎么样的呢？如果可以反复加载，那么有什么细节需要注意呢？写一段代码来测试一下吧。

我们应该关注以下两点：

1. 两次加载出来的对象是否是同一个对象

2. 两次加载的时间是否一样

```csharp
using UnityEngine;
using System.IO;
using System.Collections.Generic;
using System.Diagnostics;

public class Test : MonoBehaviour {
    protected void Start() {
        var assetPath = "Assets/Resource/Prefabs/TestPrefab.prefab";
        var dict = new Dictionary<string, UnityEngine.Object>();
        var ab = AssetBundle.LoadFromFile(Path.Combine(Application.streamingAssetsPath, "bundleprefab"));
        var sw = new Stopwatch();
        sw.Start();
        var asset = ab.LoadAsset(assetPath);
        sw.Stop();
        UnityEngine.Debug.Log("First time: " + sw.ElapsedTicks);
        dict.Add(assetPath, asset);
        sw.Restart();
        var asset2 = ab.LoadAsset(assetPath);
        sw.Stop();
        UnityEngine.Debug.Log("Second time: " + sw.ElapsedTicks);
        sw.Restart();
        var asset3 = dict[assetPath];
        sw.Stop();
        UnityEngine.Debug.Log("Read from Dictionary: " + sw.ElapsedTicks);
        UnityEngine.Debug.Log("asset == asset2: " + (asset == asset2));
        UnityEngine.Debug.Log("asset2 == asset3: " + (asset2 == asset3));
    }
}
```

{% asset_img log_output.png 打印结果 %}

可以看到重复调用LoadAsset拿到的对象都是一样的，并且第二次调用的时间比第一次显著的低，接近于自己用Dictionary缓存起来。因此推断AssetBundle内部可能也有这样一个缓存机制。