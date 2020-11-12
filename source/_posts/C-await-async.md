---
title: C# await/async对Unity开发游戏的好处
date: 2020-11-04 23:47:51
tags: 
- Unity
- C#
---



作为一名游戏前端开发，那么肯定会经常需要些一些异步的逻辑，比如在一个回合制的游戏中，需要场上所有的角色挨个释放技能。

异步的函数，在Unity当中有很多种实现方式。比较通用一些的做法，可以通过回调或者派发和监听事件来实现异步的函数调用。回调这种方式的好处是，这种编程思路是非常通用的，不依赖于任何Unity的功能，因为可以用任何语言在任何引擎和环境中实现异步功能。因此你的代码的可移植性会很好。但是缺点也很明显。以下总结几种回调最让人觉得恶心的地方。

1. 回调传递的层级越来越深，越来越容易在一些分支忘记最终的调用，导致上层永远收不到完成的回调

   ```csharp
   public class Soldier {
       bool isDisabled;
       bool isInstantAttack;
       int damage;
       public void Attack(Action<int> callback) {
           if (isDisabled) {
               return;
           }
           if (isInstantAttack) {
               callback?.Invoke(damage);
               return;
           }
           SomeAnimation(() => {
               callback?.Invoke(damage);
           });
       }
   
       void SomeAnimation(Action callback) {
           // play animation for a few seconds
           // ...
           callback?.Invoke();
       }
   }
   
   public class LogicRunner {
       public void SomeLogic() {
           var s = new Soldier();
           s.Attack(damage => {
               Debug.Log($"attack caused {damage} points of damage");
           });
       }
   }
   ```

   比如像上面这种情况，callback被一级一级往下传递，当逻辑变得复杂之后，很容易在某一层忘记调用。并且callback被不停的传递这件事本身就已经很不优雅了。

2. 如果需要在一个循环中调用异步函数，就需要修改代码结构，增加许多逻辑和变量保存状态

   ```csharp
   public class LogicRunner {
       public void SomeLogic() {
           var soldiers = new List<Soldier>();
           // unable to write this way...
           //foreach(var s in soldiers) {
           //    s.Attack(damage => Debug.Log(damage));
           //}
           var index = 0;
           Action nextAttack = null;
           nextAttack = new Action(() => {
               if (index == soldiers.Count) {
                   Debug.Log("All Soldiers have attacked");
                   return;
               }
               var soldier = soldiers[index];
               soldier.Attack(damage => {
                   index++;
                   nextAttack();
               });
           });
       }
   }
   ```

   你的代码大概得类似于上面的例子。反正是无法在一个for循环或者while循环里面解决问题。实际上，上面的这个写法已经形成了一个简单的状态机了。通过index实际上就记录的状态机的状态。所以本质上，这种异步的代码就是用状态机以及回调实现的。

OK。我就举两个例子了吧。实际上这是最简单的情况。在真实的开发者中，异步的逻辑常常很复杂，有非常多的步骤要等待，中间就会产生非常多的回调，这样就产生了所谓的“Callback Hell”。有过几个项目开发经验的同学应该很容易理解。

那么Coroutine能解决这个问题吗？

能一定程度上简化异步代码的编码难度，但是对于需要前一步结果返回值的情况，Coroutine依然要以来很多的成员变量来时间结果的传递。

例如下面这个例子。我们全部用Coroutine实现异步的攻击，但是由于Coroutine没有返回值，因此计算的结果我们不得不通过一个成员变量保存，在经过Get函数拿到。当然可能还有别的实现，不过你都绕不过要用成员变量去存储和传递。

```csharp
public class Soldier : MonoBehaviour {
    bool isDisabled;
    bool isInstantAttack;
    int damage;
    int ret;

    public int GetDamageResult() => ret;

    public IEnumerator Attack() {
        if (isDisabled) {
            yield return null;
        }
        if (isInstantAttack) {
            ret = damage;
            yield return null;
        }
        StartCoroutine(SomeAnimation());
        ret = damage;
    }

    IEnumerator SomeAnimation() {
        yield return new WaitForSeconds(3);
    }
}

public class LogicRunner : MonoBehaviour {
    int totalDamage;

    public int GetTotalDamage() => totalDamage;

    public IEnumerator SomeLogic() {
        var soldiers = new List<Soldier>();
        foreach(var s in soldiers) {
            yield return StartCoroutine(s.Attack());
            totalDamage += s.GetDamageResult();
        }
    }
}
```

所以有办法根本解决这个问题吗？

这里就是await和async的好处了。我们来看看我们怎么改造上面的例子。

```csharp
public class Soldier {
    bool isDisabled;
    bool isInstantAttack;
    int damage;

    public async Task<int> Attack() {
        if (isDisabled) {
            return 0;
        }
        if (isInstantAttack) {
            return damage;
        }
        await SomeAnimation();
        return damage;
    }

    async Task SomeAnimation() {
        await Task.Delay(TimeSpan.FromSeconds(3));
    }
}

public class LogicRunner {

    public async Task<int> SomeLogic() {
        var totalDamage = 0;
        var soldiers = new List<Soldier>();
        foreach(var s in soldiers) {
            totalDamage += await s.Attack();
        }
        return totalDamage;
    }
}

```

现在的代码是不是很优雅很简洁呢？你会发现Attack里面你再也不用担心会忘记回调函数，因为编译器会保证这个函数一定有一个返回值。其次，你再也不用声明一些没什么用的成员变量来传递结果。结果的传递和普通的函数调用一模一样。代码变得无比简洁和清晰。当逻辑变得复杂之后，维护这样的代码会变得十分简单。

下一篇，我们一起来看看await和async关键字到底实现了什么，编译器帮你做了什么来实现这些功能。