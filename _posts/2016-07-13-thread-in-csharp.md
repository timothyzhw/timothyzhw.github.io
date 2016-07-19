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

一个c#程序自动运行在一个主线程（main）里，可以通过增加线程实现多线线程。例如：
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

![线程](/img/post/7-13-threading/thread.png)
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
{%highlight cSharp %}
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
{%endhighlight%}

结果会打印1000次 **y**, 然后打印 **Thread t has ended!** .
*t.Join()* 还可以加上参数，表示在线程执行一段时间后，继续执行后续代码。

Thread.Sleep是暂停当前线程一段时间。

在等待 Sleep 或 Join 时，线程被阻塞，所以不会耗用CPU的资源。

> Thread.Sleep(0) 和 Thread.Yield() 会将当前线程交出CPU，让CPU处理其他线程。
> 可以用来更加高深的性能调优，同时也可以测试是否有线程安全的bug。

# 线程的工作原理

多线程是由一个线程调度器进行管理。CLR的调度器是操作系统代理函数。
线程调度器保证所有活动的线程分配合适的执行时间，并且在线程等待和阻塞不会占用CPU。
在多处理器的计算机上，不同的线程同时在不同CPU上运行，多线程就是混合时间切片，并做到可靠同步。

# 线程和进程

不多说了，计算机上运行多个进程，一个进程并行多个线程。进程是完全独立的，线程相对独立，但是可以共享内存。

# 线程的正确使用和错误使用

多进程有很多应用，经常使用在下面的场合：

* 保持一个可响应的用户界面
* 让遭遇各种阻塞的CPU能高效利用
* 并行执行
* 特殊的执行方式，提前加载、预加载
* 让多个请求同时执行

多线程增加了工作的复杂性，线程交互导致开发周期长和各种bug。
因此要注意做到尽量少的交互，并使用已知可用的模式。

多线程在调度和切换时使用了额外的资源。多线程也不总是加快执行命令。

# 创建和开始线程

前面已经提到，线程使类Thread类构造，传入参数 **ThreadStart** 定义了线程开始位置。

{%highlight cSharp %}
public delegate void ThreadStart();
{%endhighlight%}

调用线程的**Start**方法使线程开始执行，直到方法返回后，线程结束。

{%highlight cSharp %}
class ThreadTest
{
  static void Main()
  {
    Thread t = new Thread (new ThreadStart (Go));

    t.Start();   // Run Go() on the new thread.

    Go();        // Simultaneously run Go() in the main thread.

  }

  static void Go()
  {
    Console.WriteLine ("hello!");
  }
}

{%endhighlight%}

在上面例子中，线程 t 执行 Go( ) 的同时，主线程也调用了 Go( )。几乎同时打印出两个*hello*

``` console
hello!
hello!
```

还可以用更简单的写法，不用定义delegate，直接出入一个方法，让C#自己推断出ThreadStart代理。

{%highlight cSharp %}
Thread t = new Thread (Go);    // No need to explicitly use ThreadStart
{%endhighlight%}
还有更更简单的，直接使用lambda表达式或匿名方法：
{%highlight cSharp %}
static void Main()
{
  Thread t = new Thread ( () => Console.WriteLine ("Hello!") );
  t.Start();
}
{%endhighlight%}
# 给线程传入数据

最简单的方法是使用lambda表达式，给调用的方法传入合适的参数：
{%highlight cSharp %}
static void Main()
{
  Thread t = new Thread ( () => Print ("Hello from t!") );
  t.Start();
}

static void Print (string message)
{
  Console.WriteLine (message);
}
{%endhighlight%}
也可以在Start的时候传入参数：
{%highlight cSharp %}
static void Main()
{
  Thread t = new Thread (Print);
  t.Start ("Hello from t!");
}

static void Print (object messageObj)
{
  string message = (string) messageObj;   // We need to cast here
  Console.WriteLine (message);
}
{%endhighlight%}
因为线程Thread的构造函数可以接收下面的任一个代理
{%highlight cSharp %}
public delegate void ThreadStart();
public delegate void ParameterizedThreadStart (object obj);
{%endhighlight%}
这里有一个限制，就是ParameterizedThreadStart只能传入一个参数，并且类型是object，需要自己做转换。

## Lambda表达式和捕获变量

使用强大的lambda表达式可以给线程出入参数。然而必须小心，在线程开始前修改了捕获变量的值，那结果就麻烦了。
{%highlight cSharp %}
for (int i = 0; i < 10; i++)
  new Thread (() => Console.Write (i)).Start();
{%endhighlight%}
这段程序输出是不确定的，如：

``` console
0223557799
```

问题出在变量 **i** 在循环中指向同一处内存。因此每个线程打印出来的值是变化的。
解决方法是使用一个临时变量
{%highlight cSharp %}
for (int i = 0; i < 10; i++)
{
  int temp = i;
  new Thread (() => Console.Write (temp)).Start();
}
{%endhighlight%}
变量temp是每个循环内的局部变量，每个线程捕获到的是不同内存的变量。
用更浅显的例子说明一下：
{%highlight cSharp %}
string text = "t1";
Thread t1 = new Thread ( () => Console.WriteLine (text) );

text = "t2";
Thread t2 = new Thread ( () => Console.WriteLine (text) );

t1.Start();
t2.Start();
{%endhighlight%}
由于lambda表达式捕获的是同样的text变量，所以"t2" 打印了两次

``` console
t2
t2
```

# 线程命名
每个线程有个 **Name** 属性，可以用来调试。在Visual Studio中调试时可以显示出来线程名字。
Name属性只能设置一次，如果想再次修改会抛出异常。

静态的 **Thread.CurrentThread** 属性可以获得当前执行的线程：
{%highlight cSharp %}
class ThreadNaming
{
  static void Main()
  {
    Thread.CurrentThread.Name = "main";
    Thread worker = new Thread (Go);
    worker.Name = "worker";
    worker.Start();
    Go();
  }

  static void Go()
  {
    Console.WriteLine ("Hello from " + Thread.CurrentThread.Name);
  }
}
{%endhighlight%}
# 前台线程和后台线程
默认创建的线程是前台线程。只要有正在运行的前台线程，程序就一直活着。后台线程则不然。
当所有的前台线程执行完，程序就会结束，所有运行中的后台进程都会意外中止。

可以通过 **IsBackground** 属性来设置线程 是否是后台进程。
{%highlight cSharp %}
class PriorityTest
{
  static void Main (string[] args)
  {
    Thread worker = new Thread ( () => Console.ReadLine() );
    if (args.Length > 0) worker.IsBackground = true;
    worker.Start();
  }
}
{%endhighlight%}
在上面的例子中，如果程序运行时，如果没有输入参数，那worker进程是前台进行，就会停在ReadLine等待用户按下回车键。此时主线程执行完了退出，但是程序依然在运行。

而如果Main输入一个参数，worker线程就变成后台线程，主线程执行完成后，程序就退出了，worker的ReadLine被中止。

当一个线程被这样中止，在后台线程中的**finally**代码会被绕过。这样就有个问题，如果在finally（或者using）里写了释放资源或删除文件等操作，那就不会被执行了。
为了避免这个问题，需要保证等待这些线程执行完了后在退出程序，由两个方法：

* 如果是自己创建的线程，可以使用线程的Join方法
* 如果是线程池的线程，使用event wait handle

不管用那种方法，最好设置上超时时间，抛弃那些无法完成的线程，这样可以保证最终程序可以被关闭。
前台线程不需要这样的处理，但是最好针对无法结束的情况做处理。有时候程序不能退出的原因就是前台线程一直在运行没有结束。

# 线程优先级

线程的优先级决定了操作系统分配给他的执行时间。
优先级的枚举：
{%highlight cSharp %}
enum ThreadPriority { Lowest, BelowNormal, Normal, AboveNormal, Highest }
{%endhighlight%}
优先级只有在多线程并行的情况下有意义。
提高线程的优先级，并不能让线程一直工作，因为他还受到进程优先级的约束。略去一千个字。。。

# 处理异常

创建线程时使用 **try/catch/finally** ，不会捕捉到线程执行中的异常。
{%highlight cSharp %}
public static void Main()
{
  try
  {
    new Thread (Go).Start();
  }
  catch (Exception ex)
  {
    // We'll never get here!

    Console.WriteLine ("Exception!");
  }
}

static void Go() { throw null; }   // Throws a NullReferenceException
{%endhighlight%}
**try/catch** 部分不起作用，新创建的线程被 **NullReferenceException** 连累。
正确的做法是将错误处理移到Go方法里
{%highlight cSharp %}
public static void Main()
{
   new Thread (Go).Start();
}

static void Go()
{
  try
  {
    throw null;    // The NullReferenceException will get caught below

  }
  catch (Exception ex)
  {
    // Typically log the exception, and/or signal another thread

    // that we've come unstuck

  }
}
{%endhighlight%}
和你在main线程最高层上进行trycatch一样，所有的线程都要有异常处理。未处理的异常会让整个程序垮掉。
有些情况你不用处理异常，.NET Framework已经帮忙处理好了：

* 异步代理
* BackgroundWorker
* TPL Task Parallel Library

# 线程池

当开始一个新的线程，几千微秒被用来阻止新的私有变量栈。每个线程大概需要1M内存。
线程池 Thread Pool 通过共享和循环利用线程来降低开销，这使得多线程高可靠并且没有性能损失。
这在多处理器的电脑上使用分而治之的方法做并行运算时，非常有效果。

线程池限定了可同时运行的线程数量。太多活动的线程会加重线程管理的负担，降低CPU缓存的效率，从而阻塞操作系统。在线程池中，当达到一定数量后，任务就要等待，在别的任务完成后才能启动。

有几个方法来使用进程池:

* 使用TPL
* 调用 ThreadPool.QueueUserWorkItem
* 使用异步代理
* 使用BackgroundWorker

TPL和PLINQ是相当强大和高级的，即使没有引入进程池，也可以协助你完成多线程，稍后会讨论到。
现在要看简单看一下如何使用 **Task** 类，实现在池化线程上执行代理。

可以通过 **Thread.CurrentThread.IsThreadPoolThread** 属性来检查当前线程是否是池化线程。

# 通过TPL来实现线程池

使用Task类可以快速进入线程池。Task类是 Framework 4.0 加入的。可以这么看，Task用来替代ThreadPool.QueueUserWorkItem，而泛型 Task\<TResult\> 替代 asynchronous delegates。

使用非泛型的Task，直接调用Task.Factory.StartNew，传入代理方法：
{%highlight cSharp %}
static void Main()
{
  Task.Factory.StartNew (Go);
}
{%endhighlight%}
Task.Factory.StartNew 返回一个Task对象，使用这个对象来监视任务，如调用Wait来等待任务完成。

> 注意：如果调用了Wait方法，未处理的异常会被抛到宿主（调用Wait的）线程。如果不是调用Wait，而是任由其自动执行，那么未处理的异常就会是整个程序停止。

泛型Task\<TResult\> 是Task的子类，使用它可以在task运行完成后得到一个返回值。下面的例子中，使用了Task\<TResult\> 来下载一个网页：
{%highlight cSharp %}
static void Main()
{
  // Start the task executing:
  Task<string> task = Task.Factory.StartNew<string>
    ( () => DownloadString ("http://www.linqpad.net") );

  // We can do other work here and it will execute in parallel:

  RunSomeOtherMethod();

  // When we need the task's return value, we query its Result property:

  // If it's still executing, the current thread will now block (wait)

  // until the task finishes:

  string result = task.Result;
}

static string DownloadString (string uri)
{
  using (var wc = new System.Net.WebClient())
    return wc.DownloadString (uri);
}
{%endhighlight%}
未处理的异常在查询返回值Result属性时，被包装到AggregateException里并转发。然而如果没有到检查结果，那么未处理的异常会让进程down掉。

TPL是个好东西，特别适合多核心处理器，后续还有介绍。

# 不通过TPL来实现线程池

如果使用的是低版本的.NET Framework，就没法用TPL啦。那就要用两个老东西来实现进程池：ThreadPool.QueueUserWorkItem 和 asynchronous delegates。二者的区别就是后者会有返回值，并且未处理的异常会同步返回给调用者。

## QueueUserWorkItem

使用 **QueueUserWorkItem**，简单的通过传入一个代理就可以：
{%highlight cSharp %}
static void Main()
{
  ThreadPool.QueueUserWorkItem (Go);
  ThreadPool.QueueUserWorkItem (Go, 123);
  Console.ReadLine();
}

static void Go (object data)   // data will be null with the first call.

{
  Console.WriteLine ("Hello from the thread pool! " + data);
}
{%endhighlight%}
输出结果：

``` console
Hello from the thread pool!
Hello from the thread pool! 123
```
目标函数Go必须有个object参数。和ParameterizedThreadStart那样，可以传参数进去。但是使用QueueUserWorkItem不会有返回对象来帮助查看执行状态，而且未处理的异常也会导致程序退出。

## Asynchronous delegates
QueueUserWorkItem没有提供一个获取线程返回值的机制，异步代理可以双向传入传出多个参数，并且未处理异常也不会返回给原线程（更准确的说，叫EndInvoke线程）。

使用的方法如下：

1. 实例化一个希望并行处理的目标函数，通常是 **Func** 代理
2. 调用代理的 **BeginInvoke**，并保存 **IAsyncResult** 返回值
    **BeginInvoke**会立即返回给调用方，调用方可以在线程池运行的同时，再继续执行其他任务。
3. 当需要结果时，调用代理的**EndInvoke**，并传入上一步得到的 **IAsyncResult** 对象

下面的例子会得到传入字符串的长度：
{%highlight cSharp %}
static void Main()
{
  Func<string, int> method = Work;
  IAsyncResult cookie = method.BeginInvoke ("test", null, null);

  // ... here's where we can do other work in parallel...

  int result = method.EndInvoke (cookie);
  Console.WriteLine ("String length is: " + result);
}

static int Work (string s) { return s.Length; }
{%endhighlight%}
**EndInvoke**做了三件事情：
* 首先要等待代理执行完成（加入当时还没有完成的话）
* 接收到返回值
* 抛出任何未处理的异常

> 注意：如果没有调用**EndInvoke**，编译器不会出错，但是运行时有可能出问题。

可以定义个回调函数来实现**EndInvoke**，回调函数会在线程完成后自动执行。当然有一些额外的工作：
{%highlight cSharp %}
static void Main()
{
  Func<string, int> method = Work;
  method.BeginInvoke ("test", Done, method);

}

static int Work (string s) { return s.Length; }

static void Done (IAsyncResult cookie)
{
  var target = (Func<string, int>) cookie.AsyncState;
  int result = target.EndInvoke (cookie);
  Console.WriteLine ("String length is: " + result);
}
{%endhighlight%}

BeginInvoke 最后一个参数是用户定义值，可以通过IAsyncResult.AsyncState 属性获取到。这里传入的就是代理方法本身，所以可以再调用**EndInvoke**。

# 线程池优化
线程池开始时就有一个线程。当有任务分配进来时，线程池管理器就插入新的线程，直到最大限制值。在一段非活动期后，线程管理器会销毁一部分线程，来达到更高的效率。

可以用**ThreadPool.SetMaxThreads**属性，设置线程池的最大线程数，默认值如下：

* 1023 in Framework 4.0 in a 32-bit environment
* 32768 in Framework 4.0 in a 64-bit environment
* 250 per core in Framework 3.5
* 25 per core in Framework 2.0

当然这个值要根据具体情况不同。为什么数值会这么大，是因为总是会有一些线程被阻塞。

当然也可以通过ThreadPool.SetMinThreads来设置一个下限。提高这个最小值，可以在很多线程blocked时，仍能并行同步工作。

最小值一般是一个处理器一个线程。在服务器环境下，最小值会设置为50多

> 最小线程是怎么工作的呢？

>>设置了线程池的最小值x后，并不是马上就创建x个线程，线程池管理器会按需逐渐创建线程。不立马创建，是为了防止对应用造成冲击。假设有40个任务，每个任务执行10ms，在一台四核计算机上，需要执行100ms。理论上需要40个线程来执行。少了不能充分利用CPU，多了也没有必要。
如果假设线程不是执行10ms，而是请求网页，需要半秒钟等待回应，这时CPU时空闲的。线程池的管理策略就不行了，需要更多的线程，使得请求同时进行。
幸好，池管理器有个备份计划，如果他的队列半秒不动，就会在创建新的新城，每半秒一个，直到最大值。
半秒钟的延迟是个双刃剑，一方面保证不会马上消耗40M内存，另一方面造成不必要的等待。因此可以告诉池管理器，不要拖延，直接分配前x个线程，ThreadPool.SetMinThreads (50, 50);
