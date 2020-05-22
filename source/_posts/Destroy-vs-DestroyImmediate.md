---
title: Destroy vs DestroyImmediate
date: 2020-05-22 11:09:26
tags: Unity
---

> Destroys the object `obj` immediately. You are strongly recommended to use Destroy instead.
>
> This function should only be used when writing editor code since the delayed destruction will never be invoked in edit mode. In game code you should use [Object.Destroy](https://docs.unity3d.com/ScriptReference/Object.Destroy.html) instead. Destroy is always delayed (but executed within the same frame). Use this function with care since it can destroy assets permanently! Also note that you should never iterate through arrays and destroy the elements you are iterating over. This will cause serious problems (as a general programming practice, not just in Unity).

Unity官方文档对于DestroyImmediate的解释。给我们的信息是，Destroy是延迟执行的，但是是同一帧之内，也就是这一帧的结束时执行。第二个重要信息是DestroyImmediate可以删除资源，因此可以用在Editor模式。第三个信息是说不要在循环中Destroy。

但是Destroy和DestroyImmediate到底应该怎么用呢？下面摘一段UGUI的代码。

```csharp
namespace UnityEngine.UI
{
    /// <summary>
    /// Helper class containing generic functions used throughout the UI library.
    /// </summary>

    internal static class Misc
    {
        /// <summary>
        /// Destroy the specified object, immediately if in edit mode.
        /// </summary>

        static public void Destroy(UnityEngine.Object obj)
        {
            if (obj != null)
            {
                if (Application.isPlaying)
                {
                    if (obj is GameObject)
                    {
                        GameObject go = obj as GameObject;
                        go.transform.parent = null;
                    }

                    Object.Destroy(obj);
                }
                else Object.DestroyImmediate(obj);
            }
        }

        /// <summary>
        /// Destroy the specified object immediately, unless not in the editor, in which case the regular Destroy is used instead.
        /// </summary>

        static public void DestroyImmediate(Object obj)
        {
            if (obj != null)
            {
                if (Application.isEditor) Object.DestroyImmediate(obj);
                else Object.Destroy(obj);
            }
        }
    }
}

```

上面代码和官方文档的说明完全吻合。在isPlaying模式下使用Destroy，在isEditor模式下使用DestroyImmediate。

上面代码还有一点值得学习的，就是删除前把GameObject的parent置空了。这样对于很多操作更安全。比如Destroy之后再GetChild就不会拿到了。