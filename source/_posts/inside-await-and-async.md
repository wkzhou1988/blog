---
title: 深入await和async
date: 2020-11-11 22:24:43
tags:
- Unity
- C#
---

上次我们讲了用await/async如何能在游戏开发过程中写出更加好维护的异步代码。实际上，上一篇文章中我们已经提到了，异步代码的基本思想就是使用状态机。只不过上一篇的代码的状态机写得非常的简单，可扩展性可读性等等都比较差，并不是什么通用的代码。

那么这篇文章当中我们就来看看C#编辑器到底给我们生成了什么代码。

首先贴上一段原始代码。

```csharp
using System;
using System.Collections;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;

namespace TaskExample {
    class Program {
        static void Main(string[] args) {
            StartAsync();
            Thread.CurrentThread.Join();
        }

        static async void StartAsync() {
            var result = await DoSomethingAsync(99);
            Console.WriteLine(result);
        }

        static async Task<string> DoSomethingAsync(int someParam) {
            Console.WriteLine("Before delay");
            await Task.Delay(TimeSpan.FromSeconds(3));
            Console.WriteLine("After delay");
            return $"the param is {someParam}";
        }
    }
}

```

可以看到代码很简单，有两个async的方法，其中主要区别是一个是async void，另一个是async Task<T>类型的。这两个有什么区别，有很多文档详细解释，这里就不做介绍了。

那么我们再来看看C#编译器处理过的中间代码是什么样的。

```csharp
// Decompiled with JetBrains decompiler
// Type: TaskExample.Program
// Assembly: Task, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null
// MVID: B3D3C194-54A1-4EE1-A493-99E41A7519D8
// Assembly location: C:\Users\Admin\source\repos\Task\Task\bin\Debug\netcoreapp3.1\Task.dll
// Compiler-generated code is shown

using System;
using System.Diagnostics;
using System.Runtime.CompilerServices;
using System.Threading;
using System.Threading.Tasks;

namespace TaskExample
{
  internal class Program
  {
    private static void Main(string[] args)
    {
      Program.StartAsync();
      Thread.CurrentThread.Join();
    }

    [AsyncStateMachine(typeof (Program.<StartAsync>d__1))]
    [DebuggerStepThrough]
    private static void StartAsync()
    {
      Program.<StartAsync>d__1 stateMachine = new Program.<StartAsync>d__1();
      stateMachine.<>t__builder = AsyncVoidMethodBuilder.Create();
      stateMachine.<>1__state = -1;
      stateMachine.<>t__builder.Start<Program.<StartAsync>d__1>(ref stateMachine);
    }

    [AsyncStateMachine(typeof (Program.<DoSomethingAsync>d__2))]
    [DebuggerStepThrough]
    private static Task<string> DoSomethingAsync(int someParam)
    {
      Program.<DoSomethingAsync>d__2 stateMachine = new Program.<DoSomethingAsync>d__2();
      stateMachine.someParam = someParam;
      stateMachine.<>t__builder = AsyncTaskMethodBuilder<string>.Create();
      stateMachine.<>1__state = -1;
      stateMachine.<>t__builder.Start<Program.<DoSomethingAsync>d__2>(ref stateMachine);
      return stateMachine.<>t__builder.Task;
    }

    public Program()
    {
      base..ctor();
    }

    [CompilerGenerated]
    private sealed class <StartAsync>d__1 : IAsyncStateMachine
    {
      public int <>1__state;
      public AsyncVoidMethodBuilder <>t__builder;
      private string <result>5__1;
      private string <>s__2;
      private TaskAwaiter<string> <>u__1;

      public <StartAsync>d__1()
      {
        base..ctor();
      }

      void IAsyncStateMachine.MoveNext()
      {
        int num1 = this.<>1__state;
        try
        {
          TaskAwaiter<string> awaiter;
          int num2;
          if (num1 != 0)
          {
            awaiter = Program.DoSomethingAsync(99).GetAwaiter();
            if (!awaiter.IsCompleted)
            {
              this.<>1__state = num2 = 0;
              this.<>u__1 = awaiter;
              Program.<StartAsync>d__1 stateMachine = this;
              this.<>t__builder.AwaitUnsafeOnCompleted<TaskAwaiter<string>, Program.<StartAsync>d__1>(ref awaiter, ref stateMachine);
              return;
            }
          }
          else
          {
            awaiter = this.<>u__1;
            this.<>u__1 = new TaskAwaiter<string>();
            this.<>1__state = num2 = -1;
          }
          this.<>s__2 = awaiter.GetResult();
          this.<result>5__1 = this.<>s__2;
          this.<>s__2 = (string) null;
          Console.WriteLine(this.<result>5__1);
        }
        catch (Exception ex)
        {
          this.<>1__state = -2;
          this.<result>5__1 = (string) null;
          this.<>t__builder.SetException(ex);
          return;
        }
        this.<>1__state = -2;
        this.<result>5__1 = (string) null;
        this.<>t__builder.SetResult();
      }

      [DebuggerHidden]
      void IAsyncStateMachine.SetStateMachine(IAsyncStateMachine stateMachine)
      {
      }
    }

    [CompilerGenerated]
    private sealed class <DoSomethingAsync>d__2 : IAsyncStateMachine
    {
      public int <>1__state;
      public AsyncTaskMethodBuilder<string> <>t__builder;
      public int someParam;
      private TaskAwaiter <>u__1;

      public <DoSomethingAsync>d__2()
      {
        base..ctor();
      }

      void IAsyncStateMachine.MoveNext()
      {
        int num1 = this.<>1__state;
        string result;
        try
        {
          TaskAwaiter awaiter;
          int num2;
          if (num1 != 0)
          {
            Console.WriteLine("Before delay");
            awaiter = Task.Delay(TimeSpan.FromSeconds(3.0)).GetAwaiter();
            if (!awaiter.IsCompleted)
            {
              this.<>1__state = num2 = 0;
              this.<>u__1 = awaiter;
              Program.<DoSomethingAsync>d__2 stateMachine = this;
              this.<>t__builder.AwaitUnsafeOnCompleted<TaskAwaiter, Program.<DoSomethingAsync>d__2>(ref awaiter, ref stateMachine);
              return;
            }
          }
          else
          {
            awaiter = this.<>u__1;
            this.<>u__1 = new TaskAwaiter();
            this.<>1__state = num2 = -1;
          }
          awaiter.GetResult();
          Console.WriteLine("After delay");
          result = string.Format("the param is {0}", (object) this.someParam);
        }
        catch (Exception ex)
        {
          this.<>1__state = -2;
          this.<>t__builder.SetException(ex);
          return;
        }
        this.<>1__state = -2;
        this.<>t__builder.SetResult(result);
      }

      [DebuggerHidden]
      void IAsyncStateMachine.SetStateMachine(IAsyncStateMachine stateMachine)
      {
      }
    }
  }
}

```

这段代码很长，但是我们可以拆开来看。

首先我们来看两个async方法变成什么样了。

```csharp
    [AsyncStateMachine(typeof (Program.<StartAsync>d__1))]
    [DebuggerStepThrough]
    private static void StartAsync()
    {
      Program.<StartAsync>d__1 stateMachine = new Program.<StartAsync>d__1();
      stateMachine.<>t__builder = AsyncVoidMethodBuilder.Create();
      stateMachine.<>1__state = -1;
      stateMachine.<>t__builder.Start<Program.<StartAsync>d__1>(ref stateMachine);
    }

    [AsyncStateMachine(typeof (Program.<DoSomethingAsync>d__2))]
    [DebuggerStepThrough]
    private static Task<string> DoSomethingAsync(int someParam)
    {
      Program.<DoSomethingAsync>d__2 stateMachine = new Program.<DoSomethingAsync>d__2();
      stateMachine.someParam = someParam;
      stateMachine.<>t__builder = AsyncTaskMethodBuilder<string>.Create();
      stateMachine.<>1__state = -1;
      stateMachine.<>t__builder.Start<Program.<DoSomethingAsync>d__2>(ref stateMachine);
      return stateMachine.<>t__builder.Task;
    }
```

首先，async void函数变成了void函数，但是上面多了一个标签`[AsyncStateMachine(typeof (Program.<StartAsync>d__1))]`。这个标签有两个作用。第一，告诉编译器这是一个异步函数，第二，告诉编译器这个函数对应的AsyncStateMachine类是谁。[文档在这](https://docs.microsoft.com/zh-cn/dotnet/api/system.runtime.compilerservices.asyncstatemachineattribute?view=xamarinandroid-7.1)

其次，这两个方法里面的代码已经完全不是我们写的代码了，而是被编译器替代成了生成的代码。这两个代码应该能看懂是什么意思。基本上就是创建了一个状态机stateMachine，而这个状态机的原型就是上面标签里面的那个类。那个类也是又编译器生成的一个类，是不可见的。其中还有一个关键类型就是AsyncTaskMethodBuilder<T>和非泛型的AsyncVoidMethodBuilder类。这两个类是为了生成stateMachine里面的builder字段而存在的。我们可以看到这里状态机的state初始化为-1，并且调用了builder的Start方法，把状态机传了进去。之所以这里是ref传递，因为stateMachine是一个结构体值类型。

我们再观察下代码，发现我们两个方法里面的内容实际上被放到了对应的stateMachine的MoveNext方法里面去了。熟悉IEnumerator的朋友对MoveNext应该不陌生。MoveNext就是实现状态机的状态切换的方法。

比如我们先看看DoSomethingAsync这个方法的代码变成什么样了。

```csharp

    [CompilerGenerated]
    private sealed class <DoSomethingAsync>d__2 : IAsyncStateMachine
    {
      public int <>1__state;
      public AsyncTaskMethodBuilder<string> <>t__builder;
      public int someParam;
      private TaskAwaiter <>u__1;

      public <DoSomethingAsync>d__2()
      {
        base..ctor();
      }

      void IAsyncStateMachine.MoveNext()
      {
        int num1 = this.<>1__state;
        string result;
        try
        {
          TaskAwaiter awaiter;
          int num2;
          if (num1 != 0)
          {
            Console.WriteLine("Before delay");
            awaiter = Task.Delay(TimeSpan.FromSeconds(3.0)).GetAwaiter();
            if (!awaiter.IsCompleted)
            {
              this.<>1__state = num2 = 0;
              this.<>u__1 = awaiter;
              Program.<DoSomethingAsync>d__2 stateMachine = this;
              this.<>t__builder.AwaitUnsafeOnCompleted<TaskAwaiter, Program.<DoSomethingAsync>d__2>(ref awaiter, ref stateMachine);
              return;
            }
          }
          else
          {
            awaiter = this.<>u__1;
            this.<>u__1 = new TaskAwaiter();
            this.<>1__state = num2 = -1;
          }
          awaiter.GetResult();
          Console.WriteLine("After delay");
          result = string.Format("the param is {0}", (object) this.someParam);
        }
        catch (Exception ex)
        {
          this.<>1__state = -2;
          this.<>t__builder.SetException(ex);
          return;
        }
        this.<>1__state = -2;
        this.<>t__builder.SetResult(result);
      }
```

可以看到里面有几个状态：-1是刚进入的状态，打印Before Delay并且等3秒，0是After Delay的状态，在这里执行Delay 3秒的逻辑；然后-2是退出的状态。最后整个函数的返回值由builder的SetResult来传给awaiter。而这个返回值会在调用这个函数的地方由这个函数返回的Task的awaiter来取得。这里可以查看StartAsync这个方法。StartAsync调用了DoSomethingAsync得到一个Task，然后调用了GetAwaiter来拿到awaiter对象。通过awaiter的GetResult最后拿到异步函数的返回值。

好啦。这篇文章扒开了编译器生成的代码的外衣，我们看到了C#编译器帮我们生成了哪些代码来实现异步函数的状态机。这对我们理解异步编程有很大帮助。如果是自己来写异步代码，也完全可以参考这个思路。例如在monoBehaviour里面，Update里面完全就可以用几个状态切换不同的更新代码。

下一篇文章我们将看看如何自己定义一个可以await的类，以及如何使这个类支持async关键字。