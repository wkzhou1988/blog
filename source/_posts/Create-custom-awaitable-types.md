---
title: 创建自己的类型，实现await关键字的
date: 2020-11-15 20:09:53
tags:
- C#
- Unity
---


前面的文章介绍了await/async的实现原理和用法。这篇文章来介绍如何写出自己的基类来实现await关键字的功能。

###### 首先，为什么要自己实现awaitable的类呢？

答案就是，我们需要自己控制异步代码的步进，就是Update。

如果我们使用Task类，那么是没有办法控制它是怎么update的。游戏中我们经常需要实现的功能就是加速和减速，甚至是暂停。可是我们只想针对游戏逻辑中的一部分实现这些流程控制，而不是针对整个应用级别。这个时候通常来说我们需要自己写一个逻辑的Update入口，然后在这里处理逻辑的时间步进的计算。如果我们用Task来做，你是很难实现这样的功能的。

另一个设计上的需求是（当然这个需求比较弱），对于一个比较复杂的异步逻辑，我们希望将它封装成一个类，传入相应的参数就行。如果我们用Task来做，最后的结果就是入口一个async Task<T>函数。那么结果可能是我们会有一个非常大的类，里面全部都是async Task<T>的函数，每个函数封装的是一个异步逻辑。其实这样也可以，但是不能体现出一段逻辑的整体性。因为大家都是async的函数，不能一眼看出来层级关系。

好，那我们开始实现自己的类，我们叫它AwaitableTask。如果这时候我们写`await new AwaitableTask();`，我们会发现编译器告诉我们，这不是一个可以await的类型。那么什么样的类型是可以await的呢？

[首先贴上文档](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#awaitable-expressions)

1. 类型的编译时类型是dynamic

2. 类型实现一个GetAwaiter函数，这个函数**没有形参**，**返回一个awaiter类型**，而这个awaiter类型需要满足以下3个条件：

   1. awaiter实现`System.Runtime.CompilerServices.INotifyCompletion`接口

   2. awaiter拥有一个可读的bool类型的IsComplete**成员**属性

   3. awaiter拥有一个GetResult函数，没有形参，没有类型参数

又上面的要求可以看出，我们应该首先定义Awaiter类，然后再定义AwaitableTask类就行了。

```csharp
    public class AwaitableTask {
        public virtual void Update(float dt) {

        }

        public Awaiter GetAwaiter() {
            return new Awaiter();
        }

        public struct Awaiter : INotifyCompletion {
            public void OnCompleted(Action continuation) {
                
            }

            public bool IsCompleted => true;

            public object GetResult() {
                return null;
            }
        }
    }
```

当我们做到这里就会发现，`await new AwaitableTask();`就已经不报错了。是不是很简单呢？

那么问题来了。Awaiter应该怎么实现这几个函数呢？

显然这里面有几个关键性的函数：

1. `public Awaiter GetAwaiter()`

2. `public void OnCompleted(Action continuation)`

3. `public object GetResult()`

我们需要回顾前面文章里面贴的代码，C#编译器生成的代码来理解这几个函数有什么用。

`Console.WriteLine("Before delay");
 awaiter = Task.Delay(TimeSpan.FromSeconds(3.0)).GetAwaiter();`

所以`await Task.Delay(...)`实际上被翻译成了`awaiter = Task.Delay(TimeSpan.FromSeconds(3.0)).GetAwaiter()`。因此当我们写下await xxxClass的时候，等价的调用的GetAwaiter。

然后awaiter被传递给了builder，调用的函数是AwaitUnsafeOnCompleted。这里UnsafeOnCompleted实际上就是OnCompleted的unsafe版本。如果你的Task里面需要调用unsafe的代码，你就需要提供unsafe的版本。而AwaitOnCompleted里面会调用awaiter.OnCompleted(continuation)，这里的continuation其实就是stateMachine的MoveNext函数。也就是说在Task完成时，会调用MoveNext，而MoveNext就是我们进入我们异步逻辑的下一个状态。这里就和协程和IEnumerator是一样的了。

最后就是GetResult。当我们的Task完成后，会调用GetResult来拿到这个Task的返回值。

```csharp
		  TaskAwaiter<string> awaiter;
          int num2;
          if (num1 != 0)
          {
            awaiter = Program.DoSomethingAsync(99).GetAwaiter(); // 看这里
            if (!awaiter.IsCompleted)
            {
              this.<>1__state = num2 = 0;
              this.<>u__1 = awaiter;
              Program.<StartAsync>d__1 stateMachine = this;
              this.<>t__builder.AwaitUnsafeOnCompleted<TaskAwaiter<string>, Program.<StartAsync>d__1>(ref awaiter, ref stateMachine); // 看这里
              return;
            }
          }
          else
          {
            awaiter = this.<>u__1;
            this.<>u__1 = new TaskAwaiter<string>();
            this.<>1__state = num2 = -1;
          }
          this.<>s__2 = awaiter.GetResult(); // 看这里
          this.<result>5__1 = this.<>s__2;
          this.<>s__2 = (string) null;
          Console.WriteLine(this.<result>5__1);
```

现在代码其实已经很清晰了。编译器帮我们完成了什么呢？

1. 编译器帮我们生成了状态机

2. 编译器帮我们生成了一套框架代码，它会在builder里面将continuation传给我们的代码，以便在Task完成时能够调用到MoveNext

3. 编译器帮我们生成了返回值逻辑，它从GetResult里面拿到返回值，最后作为async函数的返回值，最后被await拿到。

那我们需要做什么呢？

1. 编译器并不知道我们的Task什么时候完成，因此要求我们提供IsCompleted接口。因此我们需要在这个接口正确返回Task的运行状态。
2. 编译器生成的代码并没有替我们调用MoveNext，而是把它回传给我们了，需要我们在完成之后调用MoveNext，保证状态机的正确前进。
3. 编译器不知道我们的返回值怎么拿到，因此需要我们实现GetResult。

知道了这些，我们的AwaitableTask是不是很好写了呢？在Update里面处理逻辑，当逻辑处理完了之后，awaiter需要能告知外面IsCompleted返回true。并且我们需要有一个task的调度器负责Update所有task，并且在完成之后设置状态，并且调用continuation。最后，我们需要实现GetResult。

下面贴上我的最终实现。更详细的代码请移步我们[仓库](https://github.com/wkzhou1988/Awaitable)。

```csharp
    // 我们希望去继承这个类，在子类中实现各种逻辑。
	public abstract class AwaitableTask : ITask {
        protected bool _isInQueue = false;

        public bool IsDestroyed { get; protected set; }

        public bool IsCompleted { get; protected set; }

        public bool IsStarted { get; protected set; }

        private Action Continuation { get; set; }

        // 用来捕捉执行过程中的异常信息
        public ExceptionDispatchInfo ExceptionDispatchInfo { get; set; }

        public bool IsCancelled { get; private set; }

        // 在任务开始前，我们希望能够自定义一些逻辑
        protected virtual void OnStart() {
        }

        public void Start() {
            IsStarted = true;
            OnStart();
        }

        // 对于非泛型的任务，返回null即可。
        public virtual object GetResult() {
            return null;
        }

        // 每个子类实现Update来处理逻辑。
        public virtual void Update(float dt) {

        }

        // 在调用await之后，任务就会进入队列，并开始Update。因此
        // 在这里我们希望能够在执行前自定义一些逻辑
        protected virtual void BeforeRun() {

        }

        // 由于一个task是可以被await多次的（对于已经完成的task，await直接返回result），
        // 这里需要防止被多次加入队列
        public Awaiter GetAwaiter() {
            if (!_isInQueue) {
                BeforeRun();
                _isInQueue = true;
                AwaitableTaskQueue.Instance.Add(this);
            }

            return new Awaiter(this);
        }

        // 在完成前执行一些逻辑
        protected virtual void OnComplete() {

        }

        // 完成任务，同时调用MoveNext（continuation）
        public void Complete() {
            OnComplete();
            Continuation?.Invoke();
            IsDestroyed = true;
        }

        protected virtual void OnCancel() {

        }

        // 取消之后也要调用continuation
        public void Cancel() {
            IsCancelled = true;
            OnCancel();
            Continuation?.Invoke();
            IsDestroyed = true;
        }

        public struct Awaiter : INotifyCompletion {
            private AwaitableTask _awaitableTask;

            public Awaiter(AwaitableTask awaitableTask) {
                _awaitableTask = awaitableTask;
            }

            // 将MovenNext传递给task
            public void OnCompleted(Action continuation) {
                _awaitableTask.Continuation = continuation;
            }

            public bool IsCompleted => _awaitableTask.IsCompleted;

            public object GetResult() {
                // 对于在update里面捕捉到的异常，在这里重新扔出去，
                // 这样做能保证在await语句的地方能通过try..catch捕捉到task内部的异常。
                if (_awaitableTask.ExceptionDispatchInfo != null) {
                    _awaitableTask.ExceptionDispatchInfo.Throw();
                }
                return _awaitableTask.GetResult();
            }
        }
    }
	
	// 所有AwaitableTask都通过这个队列来更新，方便控制步进的速度等
    public class AwaitableTaskQueue : MonoSingleton<AwaitableTaskQueue> {

        private LinkedList<ITask> _tasks = new LinkedList<ITask>();

        public void Add(ITask task) {
            _tasks.AddLast(task);
        }

        public bool Empty => _tasks.Count == 0;

        private void Update() {
            OnUpdate(Time.deltaTime, Time.unscaledDeltaTime);
        }

        public void OnUpdate(float deltaTime, float unscaleDeltaTime) {
            if (_tasks.Count == 0) {
                return;
            }

            var cur = _tasks.First;
            while (cur != null) {
                var next = cur.Next;
                var task = cur.Value;

                try {
                    if (task.IsDestroyed) {
                        _tasks.Remove(cur);
                    }
                    else {
                        if (!task.IsStarted) task.Start();

                        if (task.IsCompleted) {
                            _tasks.Remove(cur);
                            task.Complete();
                        }
                        else {
                            task.Update(deltaTime);
                        }
                    }
                }
                catch (Exception ex) {
                    _tasks.Remove(cur);
                    // 捕捉异常信息，然后传递给Task
                    task.ExceptionDispatchInfo = ExceptionDispatchInfo.Capture(ex);
                    task.Cancel();
                }

                cur = next;
            }
        }
    }

```

###### 结语

这一套设计主要解决的问题是：

1. 通过使用await，解决的异步逻辑写起来繁琐，可读性差的问题，在游戏开发中有大量的异步逻辑的地方能极大的提高效率（毕竟状态机的代码不需要写了）
2. 逻辑中的Task抽象可以完全由自己的代码控制，可以加速减速暂停，也可以取消删除，也可以统计数量，等等。



下一篇，我来介绍一下在项目中如何使用这套设计来简化异步逻辑代码。