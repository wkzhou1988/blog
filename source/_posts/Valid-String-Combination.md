---
title: 算法面试：检查一个字符串是否是另外两个字符串的组合
date: 2020-05-14 10:10:49
tags: 
- 算法
- C#
---

最近面试遇到一个算法题，感觉应该是LeetCode上面的原题或者微变形，想在这分析一下。

题目说有字符串str1，str2， anim，其中anim必须由str1，str2中的字母组成，且必须保证所以字符在anim中的相对顺序和原本的相对顺序不变。比如str1=abc，str2=def，那么adbefc是一个合理的anim，defabc也是，还有很多解。由于是视频面试，原题我没有读的很清楚，并且有点紧张，思考也不是很严谨。当时简单看了一眼题目底下的测试用例，我想了一两分钟就开始写了下面这个版本。


```csharp
public class Solution {
    public bool IsValid(string str1, string str2, string anim) {
        if (str1.Length + str2.Length != anim.Length) return false;
        if (str1.Length == 0) return str2 == anim;
        if (str2.Length == 0) return str1 == anim;

        int index1 = 0, index2 = 0;
        for (int i = 0; i < anim.Length; i++) {
            if (index1 < str1.Length && str1[index1] == anim[i]) {
                index1++;
            } else if (index2 < str2.Length && str2[index2] == anim[i]) {
                index2++;
            } else {
                return false;
            }
        }
        return true;
    }
}
```

可以看到上面的算法假定的是anim由str1，str2的全部字符组成（我不确定题目中有没有说了）。但是最大的问题是，算法中永远优先取str1。那么如果当前位置str1，str2相等，那么就会优先选str1。那这显然是不对的。比如ab，ac，acab这样的组合，第一位都是a，但是算法会从str1去取，然后到第二位判断的时候会认为必须是b或者a，然而现在是c，结果会认为这不是一个合理的答案。其实只要第一位从str2开始取就行了。

那么看到这里就很明显了，如果str1，str2中有相同的字符，那么就有可能会遇到这个字符不知道是从str1还是str2来取。那么算法上，这就是典型的回溯了。知道了这一点，代码还是比较好写的，并没有太多难点。至于anim的长度会不会比str1+str2要短，这个其实很好解决。这里就不管了，假定`anim.Length==str1.Length+str2.Length`吧。回溯的思想就是一旦走不通就返回上次的地方，那对于这个题，需要回溯的就是str1和str2的index，即上次anim的字符是从哪个string里面的哪个index取过来的。

```csharp
public class Solution {
    string str1;
    string str2;
    string anim;
    int index1;
    int index2;
    bool success;

    void Trackback() {
        if (success) return;
        if (index1 + index2 == anim.Length) {
            success = true;
        } else {
            if (index1 < str1.Length && anim[index1 + index2] == str1[index1]) {
                index1++;
                Trackback();
                index1--;
            }
            if (index2 < str2.Length && anim[index1 + index2] == str2[index2]) {
                index2++;
                Trackback();
                index2--;
            }
        }
    }

    public bool IsValid(string str1, string str2, string anim) {
        this.str1 = str1;
        this.str2 = str2;
        this.anim = anim;
        Trackback();
        return success;
    }
}

```

结语：虽然算法现在写出来了，但是第一个方案显然不是太好。现在回想来看，面试的时候的大忌应该是拿到就写。实际上题目虽然有些东西没说清楚，但这正是面试官考察的东西。面试时我们还是应该不要想当然，要多去问题目的边界在哪里，是否有隐藏的坑点不要掉进去，而不是一看就觉得理解的题目然后开始按照自己的想法去写。现在回头看，面试可能更多的不是看你能不能写的多么的好，而是你的思路是否正确。毕竟很少有面试时就把你的代码拿去跑一下看看的，最终看的也是你的思路和基本功，所以有些细节可能写的不对，但是不影响你表达你会这道题。