---
title: Unity GUID和AssetBundle Hash
date: 2020-06-10 17:27:49
tags: 
- Unity
---

### 资源GUID的计算方法

做Unity开发的应该都知道Unity对每个资源文件都会生成一个GUID存在对应的meta文件中。但是Unity似乎并没有在在什么地方明确说过这个GUID是怎么生成的。于是我做了一些小实验，发现了一些有用而且重要的性质。

找到一篇 [官方文档](https://learn.unity.com/tutorial/assets-resources-and-assetbundles?_ga=2.194913709.1026850075.1591773718-1618049787.1515739666#5c7f8528edbc2a002053b5a6)，基本上和我做的实验能对上。

> The Unity Editor has a map of specific file paths to known File GUIDs. A map entry is recorded whenever an Asset is loaded or imported. The map entry links the Asset's specific path to the Asset's File GUID. *If the Unity Editor is open when a .meta file goes missing and the Asset's path does not change, the Editor can ensure that the Asset retains the same File GUID.*
>
> If the .meta file is lost while the Unity Editor is closed, or the Asset's path changes without the .meta file moving along with the Asset, then all references to Objects within that Asset will be broken.

经过试验，GUID的生成算法基本跟文件内容无关，跟文件Import的时候的路径有关，当然也跟文件名有关。**最最最重要的是，还跟一个种子有关，这个种子应该是Unity启动的时候生成的（比如根据时间戳什么的）。**

知道了这个之后就能解释很多很坑的东西。

比如说美术同学提交的时候总是忘了提交meta文件，结果其他人更新下来资源之后所有引用都失效了。最可怕的是可能有人生成了meta然后又提交了覆盖了美术的meta，这样基本上美术就要返工了，而且你还跟他解释不清为什么（主要是讲了也听不懂）。

### 文件没有更改，AssetBundle的Hash为什么会变

下面讲一下另一个最近遇到的很坑的点。

我们项目一直以来打包Lua代码的时候都会造成Lua的AssetBundle的Hash值变化。但是其实我们什么都没有改。按照我们的理解理论上不应该这样子。然后搜索一番之后，网上确实有很多说Unity在每次打AssetBundle之后Hash和MD5都在变。于是很长一段时间我们以为就是这样了，接受了这个“事实”。

但是这里面有一些解释不清的东西，比如为什么只有Lua的AssetBundle老在变，Prefab的就好好的呢？

首先我测试了很多次，至少在同一个BuildTarget下，多次打包AssetBundle，Hash和MD5都是不变的。这是可以肯定的。然后我仔细看了我们的代码，原来在Lua打包之前我们把所有Lua代码Copy一份到另一个目录，并且加上.bytes后缀，这样Unity才能打进AssetBundle中（Lua文件不是Unity认识的TextAsset文件，没法打到AssetBundle里）。这样相当与在项目中创建了很多TextAsset文件，每一个都会有一个GUID。打完包之后会删除这些.bytes文件。但是根据前面的理论，同样的文件下次再拷贝过来，GUID是不会变的，那对应的AssetBundle的Hash也不会变。

可是问题是我们使用Jenkins调用Unity的命令行打包，打包完成之后会退出！这一点被我们忽略了！而我们是没有保存这些.bytes文件到版本控制的。因此每次启动命令行再Copy这些文件后他们的GUID都变了。那打出来的AssetBundle当然就变啦！

### 文件明明被改了，AssetBundle的Hash为什么没有变

最近还发现一个比较坑的点。

SpriteAtlas有一个Include In Build选项。这个选项不知道大家有没有研究过。基本意思就是，如果你勾上，那么所有引用了这个SpriteAtlas中图片的资源在打包的时候会自动把这个图集引用进来打进AssetBundle而你不用操心，并且你在Manifest文件里也找不到它的踪影，除非你用AssetStudio或者别的工具解开AssetBundle二进制去看里面的文件列表。

那么现在我有一堆Prefab引用了一个图集，但是图集的压缩格式选错了导致很模糊。我改好压缩格式，在打包，咦，为什么AssetBundle一点变化都没有？加上了`BuildAssetBundleOptions.ForceRebuildAssetBundle`也没用。

纠结了很久才发现，原来AssetBundle还是重新打了，MD5也变了，但是Hash没有变。而我们的AssetBundle对比依赖的Hash值，因此客户端更新不到。结论就是AssetBundle的Hash是根据Manifest中的资源文件的生成的。

解决了以上这些问题，相信做资源更新就不难了。