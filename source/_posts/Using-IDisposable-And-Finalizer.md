---
title: C#中using, IDisposable, 析构函数的用法
date: 2020-05-25 11:24:55
tags: C#
---

一直对C#的内存管理可以说是一知半解，今天特地研究了一下。

先说结论吧。对于使用了非托管资源的类，需要实现IDisposable，最好使用using语法来保证正确的释放。最好提供Finalize来保证即使Dispose没有调用也能最后释放资源。因此最好按照下面的模式来实现：

```csharp
using System;

class BaseClass : IDisposable
{
    // To detect redundant calls
    private bool _disposed = false;

    ~BaseClass() => Dispose(false);

    // Public implementation of Dispose pattern callable by consumers.
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    // Protected implementation of Dispose pattern.
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed)
        {
            return;
        }

        if (disposing)
        {
            // TODO: dispose managed state (managed objects).
        }

        // TODO: free unmanaged resources (unmanaged objects) and override a finalizer below.
        // TODO: set large fields to null.

        _disposed = true;
    }
}
```

在开始分析前，先放上StackOverflow上面的[优秀答案](https://stackoverflow.com/questions/538060/proper-use-of-the-idisposable-interface?answertab=votes#tab-top)的链接。此答案写得非常清晰，结合官方文档可以更好的理解上面的模式是怎么来的。

首先说[using](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-statement)和[IDisposable](https://docs.microsoft.com/en-us/dotnet/api/system.idisposable?view=netcore-3.1)，因为一般这两个是一起用的。using是C#提供的一个语法，保证IDisposable被正确的使用。这里说的正确使用，就是说确保Dispose方法得到调用。

那么什么样的类需要实现IDisposable呢？答案是使用了非托管成员的类，或者使用了实现IDisposable的成员。对于一般的完全没有使用非托管数据类型的类是完全不需要使用IDisposable的。因为当一个对象不再被引用，GC会负责所有托管对象的回收。但是对于非托管对象GC是不知道他们的存在和如何释放的，因此需要程序员自己手动释放，于是就产生了Dispose的调用。而为了防止程序员犯错而忘记调用Dispose，因此出现了using语法，保证一个IDisposable对象在自己的作用域结束的时候，编译器会帮你调用Dispose。

然后再说[Finalize](https://docs.microsoft.com/en-us/dotnet/api/system.object.finalize?view=netcore-3.1)。Finalize是对象在释放以前最后一个释放资源的机会。GC在回收内存前，会对所有重写了Finalize的对象调用这个方法。对于C#而言，你只需要写一个析构函数，等同于提供了Finalize。Finalize由GC调用，运行在另一个线程上。根据官方文档，Finalize有几个限制。

1. 执行时间和线程都是不确定的
2. 任何两个对象的Finalize的调用顺序是不确定的

> [Finalize](https://docs.microsoft.com/en-us/dotnet/api/system.object.finalize?view=netcore-3.1) operations have the following limitations:
>
> - The exact time when the finalizer executes is undefined. To ensure deterministic release of resources for instances of your class, implement a `Close` method or provide a [IDisposable.Dispose](https://docs.microsoft.com/en-us/dotnet/api/system.idisposable.dispose?view=netcore-3.1) implementation.
> - The finalizers of two objects are not guaranteed to run in any specific order, even if one object refers to the other. That is, if Object A has a reference to Object B and both have finalizers, Object B might have already been finalized when the finalizer of Object A starts.
> - The thread on which the finalizer runs is unspecified.

那么回过头来再来解释开头提出的pattern。

1. 提供析构函数的目的是，如果代码中没有调用Dispose，那么最终的资源还是会在析构函数中释放，只不过比较晚，总比泄露好。
2. Dispose已经释放了资源，因此其实可以不用调用Finalize了。`GC.SuppressFinalize(this)`就是告诉GC不要再调Finalize了。
3. 由于GC调用Finalize的顺序不确定，因此在Finalize中释放自己的托管成员变量是不安全的（GC会保证托管的对象释放，不用自己管），通过isDisposing来判断。如果是自己手动调用Dispose（包括using的），都可以保证顺序（using后声明的会先释放）。
4. 通过isDisposed来防止多次调用。

好了，更详细的还是看文档吧。我这只做一点总结。