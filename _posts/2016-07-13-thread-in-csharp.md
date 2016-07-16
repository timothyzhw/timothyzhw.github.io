---
layout:     post
title:      "Threading in C# "
subtitle:   "part 1 getting started"
date:       2016-07-13
author:     "Joseph Albahari, Trans by Tim"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - c#
    - Threading
---
> c#里有好几个「多线程」实现，到底用哪个更合理，带着这个问题，一起来学习一下多线程吧。
首先是Joseph Albahari的Threading in C# 的翻译。

# 概念
c# 通过多线程来实现代码并行运行，一个线程相对独立，多个线程可以同时一起运行。

一个c#程序自动运行在一个主线程（main）里，可以增加线程实现多线线程。例如：
{%highlight cSharp %}
class ThreadTest
{
  static void Main()
  {
    Thread t = new Thread (WriteY);   //新建一个线程，指定线程运行WriteY函数

    t.Start();                        // 线程执行，WriteY开始执行

    for (int i = 0; i < 1000; i++)
      Console.Write ("x");           // 同时，主线程也可以做一些其他事情

  }

  static void WriteY()
  {
    for (int i = 0; i < 1000; i++) Console.Write ("y");
  }
}
{%endhighlight%}

主线程里新建了线程t，在线程t里运行函数WriteY打印字母 *y* 。同时主线程打印字母 *x* :

``` console
xxxxxxxxxxyyyyyyyyxxxyxyxyx
xyyyyyyyyxxxyxyxyxxyyyyyyyy
```

![线程](/img/post/7-13-threading/newthread.png)
当线程启动后，其IsAlive属性为true，直至线程结束。
当线程构造函数中传入的delegate执行完成，线程就结束了，并且不会被重启。

下面的例子里，一个有局部变量的方法在主线程和子线程中同时调用：
{%highlight cSharp %}
tatic void Main()
{
  new Thread (Go).Start();      // Call Go() on a new thread

  Go();                         // Call Go() on the main thread

}

static void Go()
{
  // Declare and use a local variable - 'cycles'

  for (int cycles = 0; cycles < 5; cycles++) Console.Write ('?');
}
{%endhighlight%}

每个线程都独立的内存栈，所以结果可以看到是10个 *？* 。

``` console
??????????
```

多个线程也可以同时引用一个对象，如：

{%highlight cSharp %}
class ThreadTest
{
  bool done;

  static void Main()
  {
    ThreadTest tt = new ThreadTest();   // Create a common instance

    new Thread (tt.Go).Start();
    tt.Go();
  }

  // Note that Go is now an instance method
  void Go()
  {
     if (!done) { done = true; Console.WriteLine ("Done"); }
  }
}
{%endhighlight%}

两个线程都调用了同一个对象的 *Go()*, 所以字段done只有一个，结果 *Done* 只打印了一次。

``` console
Done
```
这两个例子也说明一个关键概念：线程安全。程序的输出是不确定的，有可能输出两个 *Done*。
如果改一下Go方法的顺序，*Done* 就会被又打印出来一次。

{%highlight cSharp %}
static void Go()
{
  if (!done) { Console.WriteLine ("Done"); done = true; }
}
{%endhighlight%}

问题是当判断if条件时，另个正在执行WriteLine，还没有来得及把 done 设置为 true。

补救的方法是使用排它锁 exclusive lock。
{%highlight cSharp %}
lass ThreadSafe
{
  static bool done;
  static readonly object locker = new object();

  static void Main()
  {
    new Thread (Go).Start();
    Go();
  }

  static void Go()
  {
    lock (locker)
    {
      if (!done) { Console.WriteLine ("Done"); done = true; }
    }
  }
}
{%endhighlight%}

当两个线程同时竞争一个锁，那一个线程需要阻塞等待锁可用。这保证了只有一个线程执行在执行这段代码。
这种多线程环境下的避免不确定性的方法，就是线程安全的。

当线程被阻塞时，不会占用CPU资源。

# Join和Sleep

如果想等一个线程结束，可以调用这个线程的t.Join()方法。

static void Main()
{
  Thread t = new Thread (Go);
  t.Start();
  t.Join();
  Console.WriteLine ("Thread t has ended!");
}

static void Go()
{
  for (int i = 0; i < 1000; i++) Console.Write ("y");
}

结果会打印1000次 **y**, 然后打印 **Thread t has ended!** .
*t.Join()* 还可以加上参数，表示在线程执行一段时间后，继续执行后续代码。

Thread.Sleep是暂停当前线程一段时间。

在等待 Sleep 或 Join 时，线程被阻塞，所以不会耗用CPU的资源。

>>  Thread.Sleep(0) 和 Thread.Yield() 会将当前线程交出CPU，让CPU处理其他线程。
> 可以用来更加高深的性能调优，同时也可以测试是否有线程安全的bug。

# 线程的工作原理

多线程是由一个线程调度器进行管理。CLR的调度器是操作系统代理函数。
线程调度器保证所有活动的线程分配合适的执行时间，并且在线程等待和阻塞不会占用CPU。

单核
