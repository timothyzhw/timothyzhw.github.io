---
layout:     post
title:      "Threading in C# 5"
subtitle:   "PART 5 PARALLEL PROGRAMMING"
date:       2016-09-10
author:     "Joseph Albahari, Trans by Tim"
header-img: "img/post/9-10-theadprogram/bk.jpg"
catalog: true
tags:
    - c#
    - Threading
---
>多线程实践

本节讲涵盖Framework 4.0新引入的多线程API，可以更好适用多核处理器：
* Parallel Linq or PLinq
* Parallel 类
* task
* concurrent 集合
* SpinLock 和 SpinWait

这些API总的称作PFX（Parallel Framework）. 其中的Parallel类和task一起叫做并行任务库TPL（Task Parallel Library）。
Framework 4.0 还增加了几个底层的线程构造，目标和传统的多线程相同，前面已经提到了。

* The low-latency signaling constructs (SemaphoreSlim, ManualResetEventSlim, CountdownEvent and Barrier)
* Cancellation tokens for cooperative cancellation
* The lazy initialization classes
* ThreadLocal<T>

# Why PFX

近一段时间，CPU的时钟频率停滞了，制造商转而更关注增加CPU核数。这对编程人员是个问题，因为以前的单线程代码，不能自动在这些多核上加速。
利用多核CPU对于大多数服务器应用是很简单的，因为每个线程可以独立处理一个客户端的请求，但是在桌面应用却很困难，因为需要对计算敏感的代码做如下改造：

* 划分为小块
* 通过多线程并行执行这些小块
* 使用线程安全并高效的方法，在合适的时机收集执行结果

虽然这些都可以通过使用经典的多线程架构，但是却是非常笨拙的，尤其是在划分和收集阶段。更深层的问题是一般的 lock 和 线程安全会占据大部分内容，当很多线程工作在同一份数据上。

PFX库设计的初衷，就是为了帮助提升这些场景

```
  利用多核和多处理器而进行的编程，称为并行编程，是广义多线程的一个子集。
```
## PFX的概念

在线程间分配工作有两个策略：数据并行和任务并行。

当一组任务必须在许多数据上执行，可以并行化为让每个线程执行任务，每个任务执行一个数据子集。这个就是数据并行，就是将数据在线程间划分开。相反，任务并行是切分任务，就是说每个线程执行不同的任务。

一般数据并行比较容易，在高并行硬件上缩放更好，因为可以减少和消除共享数据。同时数据并行显示相对于离散的任务，有更多的数据并行需求。

数据并行便于构造并行机制，并行工作开始和结束的时间点是相同的。相反任务病史是不能非结构化的，并行工作可能出现在程序的任意地方。结构化的并行简单并不容易出错，并可以讲划分的工作和线程同步的工作提取到类库里。

## PFX组件

PFX有两个层次功能。上层是两个结构化的数据并行API：PLINQ和并行类；下层包括任务并行类加上一组附加的结构来帮助并行编程。

PLINQ提供了非常丰富的功能：能够自动化并行的所有步骤，包括讲工作划分为任务，在线程执行任务，讲结果收集到一个单独的输出序列。
这称为声明性，因为可以简单的声明说希望进行并行化工作，而让Framework来接管实现的细节。
相反，其他的方法是命令式，这里需要明确写出划分和整理的代码。在Parallel类里，必须要自己收集结果，在任务并行机制里，必须自己划分工作。

Partitions work 	Collates results
PLINQ 	Yes 	Yes
The Parallel class 	Yes 	No
PFX’s task parallelism 	No 	No

# PLINQ

PLINQ可以自动将局部的LINQ查询并行化。PLINQ的有点是将工作划分和结果收集的重担转移到框架上。

要使用PLINQ，只需要在输入的序列上调用AsParallel()方法，然后向平常一样继续LINQ查询就行了。
下面的查询计算3-100,000之间的所有质数，将会充分利用所有的机器内核：

{%highlight cSharp %}
// Calculate prime numbers using a simple (unoptimized) algorithm.

IEnumerable<int> numbers = Enumerable.Range (3, 100000-3);
 
var parallelQuery = 
  from n in numbers.AsParallel()
  where Enumerable.Range (2, (int) Math.Sqrt (n)).All (i => n % i > 0)
  select n;
 
int[] primes = parallelQuery.ToArray();
{%endhighlight%}

AsParallel是System.Linq.ParallelEnumerable类中的一个扩展方法，
它将输入的序列绑定到 ParallelQuery<TSource>，这将使得后续的LINQ查询操作绑定到ParallelEnumerable。
这些提供了标准查询操作的并行实现。本质上，是将输入的序列划分成小段并在不同线程执行，然后收集返回结果到一个输出序列：

调用AsSequential()方法会将解绑ParallelQuery，后续的查询将绑定在标准的查询操作，顺序执行。
在调用那些有副作用或者非线程安全的方法前，必须要先调用AsSequential()。

对于接受两个输入序列的查询操作（如Join,GroupJoin,Concat,Union,Intersect,Except和Zip），必须在两个输入序列上都使用AsParallel()，否则会有异常抛出。
当然，不必在查询执行过程中一直使用AsParallel，因为PLINQ操作的输出仍然是ParallelQuery。事实上，再次调用AsParallel会降低效率，因为这将强制合并再划分查询。

{%highlight cSharp %} 
mySequence.AsParallel()           // Wraps sequence in ParallelQuery<int>
          .Where (n => n > 100)   // Outputs another ParallelQuery<int>
          .AsParallel()           // Unnecessary - and inefficient!
          .Select (n => n * n)
{%endhighlight%}

不是所有的查询操作都能用并行加速。对于不能的那些，PLINQ将替换为顺序操作。如果PLINQ认为并行实际将减缓局部执行速度，可能会按照顺序来执行。

PLINQ只能作用于本地集合，LINQ to SQL 和 Entity Framework 不能使用，因为这些情况下，LINQ会转换成SQL并在数据库服务器执行。当然，可以使用PLINQ对数据库结果上执行额外的本机操作。

```
如果PLINQ执行抛出异常，是一个AggregateException，其中的InnerExceptions属性包括真实的异常。参考 Working with AggregateException。 
```



'''
为什么不默认使用AsParallel

'''

# Parallel执行基本轨迹

PLINQ和一般LINQ查询相同，是懒加载的。也就是说当开始消费结果时才触发真正的查询操作，典型的是用foreach循环。

当枚举结果

# PLINQ和排序

并行操作的副作用是收集的结果，不必和提交的顺序一致，在上图已经示意。也就是说LINQ的保证按照顺序输出，不在被持有。
如果真的需要保证顺序，可以强制在AsParallel()之后调用AsOrdered():

myCollection.AsParallel().AsOrdered()...

调用AsOrdered会引发大量元素的性能冲击，因为PLINQ必须跟踪每个元素的初始位置。

可以在后续查询中使用AsUnordered来消除AsOrdered的影响，

inputSequence.AsParallel().AsOrdered()
  .QueryOperator1()
  .QueryOperator2()
  .AsUnordered()       // From here on, ordering doesn’t matter
  .QueryOperator3()
  ...

AsOrdered没有设置为默认，是因为对于一般的查询并不关心原始输入的顺序。也就是说，如果将AsOrdered设置为默认，为了达到最佳性能，还要使用AsUnordered在大部分查询上，那也是一种负担。

# PLINQ 局限

当前在PLINQ并行中有一些实现上的限制。这些限制在以后的sp或新版Framework中会取消。

下面的查询操作将阻止并行，直到源数据在原来的索引位置。

* Take,TakeWhile,Skip,SkipWhile
* Select,SelectMany,ElementAt （有索引的版本）

大部分的操作改变了元素的索引位置（包括Where这样移除元素的查询）。这就意味着如果要使用操作，通常是要在查询一开始就用。

下面的查询是并行的，但是使用复杂的划分策略有时会比顺序执行还要慢：
* Join，Groupby,GroupJoin,Distinct,Union,Intersect,Except 

Aggregate操作的seeded多态在标准情况下不会并行，PLINQ提供了特殊的解决方法。

所有其他的操作都是并行的，然而不能保证这些操作在执行时是并行的。PLINQ如果判断并行会减慢部分查询操作的速度，那它会执行顺序操作。
可以重写这个操作，AsParallel()使用下面的方法强行并行化：

.WithExecutionMode (ParallelExecutionMode.ForceParallelism)

# 示例： Parallel Spellchecker

假设要实现一个能在超大文档上进行快速的拼写检查，并能充分使用多核CPU。将算法写成LINQ后，可以很容易并行。

第一步是下载一个英文词典到一个HashSet来实现高效查表：
if (!File.Exists ("WordLookup.txt"))    // Contains about 150,000 words
  new WebClient().DownloadFile (
    "http://www.albahari.com/ispell/allwords.txt", "WordLookup.txt");
 
var wordLookup = new HashSet<string> (
  File.ReadAllLines ("WordLookup.txt"),
  StringComparer.InvariantCultureIgnoreCase);

下面使用单词查表来构造一个测试文档，文档随机取了1,00,000个单词，并引入两个错误的单词。

var random = new Random();
string[] wordList = wordLookup.ToArray();
 
string[] wordsToTest = Enumerable.Range (0, 1000000)
  .Select (i => wordList [random.Next (0, wordList.Length)])
  .ToArray();
 
wordsToTest [12345] = "woozsh";     // Introduce a couple
wordsToTest [23456] = "wubsie";     // of spelling mistakes.

下面可以使用wordLookup来测试wordsToTest，进行并行检查。PLINQ做起来很容易：

var query = wordsToTest
  .AsParallel()
  .Select  ((word, index) => new IndexedWord { Word=word, Index=index })
  .Where   (iword => !wordLookup.Contains (iword.Word))
  .OrderBy (iword => iword.Index);
 
query.Dump();     // Display output in LINQPad

下面是输出结果

IndexedWord 是一个自定义的结构体：

struct IndexedWord { public string Word; public int Index; }

wordLookup.Contains方法在执行中让查询有点“肉”，所以值得并行化。

···
可以使用匿名类型来代替IndexedWord结构，但是会降低性能，因为匿名类型会分配堆上，并需要垃圾回收。

这个差别在顺序执行时无所谓，但是在并行查询是，基于栈的分配将有很大的优势。因为基于栈的分配高度并行（每个线程有自己的栈），相反所有线程必须竞争一个堆，这个堆只有单一的内存管理器和垃圾回收器。
···

## 使用ThreadLocal<T>

继续扩展上面的例子，让测试单词随机生成的过程并行化。上面构造测试数据使用了LINQ，所以很容易并行执行。

这是原来的代码：

string[] wordsToTest = Enumerable.Range (0, 1000000)
  .Select (i => wordList [random.Next (0, wordList.Length)])
  .ToArray();

  不幸的的是 random.Next 不是线程安全的，简单的将 AsParallel()加入到查询中是不行的。可能的解决方法是在random.Next使用锁，然而这样会限制并发。
  好的方法是使用ThreadLocal<Random>来新建一个独立的Random对象给每个线程。下面是并行的代码

  var localRandom = new ThreadLocal<Random>
 ( () => new Random (Guid.NewGuid().GetHashCode()) );
 
string[] wordsToTest = Enumerable.Range (0, 1000000).AsParallel()
  .Select (i => wordList [localRandom.Value.Next (0, wordList.Length)])
  .ToArray();

在工厂函数中，使用了Guid的哈希值来初始化Random对象，保证短时间内生成的Random对象种子是不同的。
````
 MSDN对Random的线程安全的说明
  Instead of instantiating individual Random objects, we recommend that you create a single Random instance to generate all the random numbers needed by your app. However, Random objects are not thread safe. If your app calls Random methods from multiple threads, you must use a synchronization object to ensure that only one thread can access the random number generator at a time. If you don't ensure that the Random object is accessed in a thread-safe way, calls to methods that return random numbers return 0.
```
什么时候用PLINQ
你可能试着在你的应用里搜寻已经有的LINQ查询，并试着改成并行。但是似乎没有什么效果，因为大部分问题用LINQ就执行的很快，并行没有好处。
更好的方式是找一个CPU敏感的瓶颈，并考虑是否可以用LINQ来实现？

PLINQ适合那些烦人的并行问题，同样适用于

# 纯粹的功能

因为PLINQ并行线程上执行查询，所以必须小心不要执行线程不安全的操作，特别是写到变凉里的副作用而引发的线程不安全。

// The following query multiplies each element by its position.
// Given an input of Enumerable.Range(0,999), it should output squares.
int i = 0;
var query = from n in Enumerable.Range(0,999).AsParallel() select n * i++;

当然可以用锁或者是Interlock，但是问题依然存在，没有在对应的输入上有正确的值。加上AsOrdered也不管用，因为AsOrdered只是保证了输出的顺序，而不是执行的顺序，实际上也不是顺序执行。

因此执行可以替换为使用indexed版本的Select
var query = Enumerable.Range(0,999).AsParallel().Select ((n, i) => n * i);

为了最好的性能，所有查询的方法中都应该线程安全，通过不写字段和属性的方式。如果通过锁的方式，并行化的潜能被限制，被划分为函数包含锁的执行时间总。

#调用阻塞或者I/O密集型的函数

有时一个长时间执行的查询不是CPU密集型的，而是在等待一些东西——例如下载网页或硬件反馈。PLINQ可以高效的并行化这样的查询，通过在AsParallel后面调用WithDegreeOfParallelism。
例子：假设要同时ping六个网站，除了使用繁琐的asynchronous delegates或者手工起6个线程。然而也可以使用PLINQ轻松完成：
from site in new[]
{
  "www.albahari.com",
  "www.linqpad.net",
  "www.oreilly.com",
  "www.takeonit.com",
  "stackoverflow.com",
  "www.rebeccarey.com"  
}
.AsParallel().WithDegreeOfParallelism(6)
let p = new Ping().Send (site)
select new
{
  site,
  Result = p.Status,
  Time = p.RoundtripTime
}

WithDegreeOfParallelism强制PLINQ同时运行一定数量的任务。在调用阻塞的方法（如Ping.Send）时，必须这样用。因为PLINQ假设查询都是CPU密集型的而分配相应的任务。这时一个双核机器，PLINQ默认会先启动两个任务，这明显是不符合预期的。

```
PLINQ的每个任务是在一个线程上服务，受制于线程池。可以通过ThreadPool.SetMinThreads.加速初始化线程。

```

另一个例子，假设开发一个监控系统，需要不断将四个安保摄像机上的图片从合并成一个图片，并显示在CCTV。其中摄像机的类：

class Camera
{
  public readonly int CameraID;
  public Camera (int cameraID) { CameraID = cameraID; }
 
  // Get image from camera: return a simple string rather than an image
  public string GetNextFrame()
  {
    Thread.Sleep (123);       // Simulate time taken to get snapshot
    return "Frame from camera " + CameraID;
  }
}

为了得到合成图像，必须在四个camera对象上依次调用GetNextFrame。假设操作是I/O相关的，使用PLINQ可以很容易实现四个框同时并行：
Camera[] cameras = Enumerable.Range (0, 4)    // Create 4 camera objects.
  .Select (i => new Camera (i))
  .ToArray();
 
while (true)
{
  string[] data = cameras
    .AsParallel().AsOrdered().WithDegreeOfParallelism (4)
    .Select (c => c.GetNextFrame()).ToArray();
 
  Console.WriteLine (string.Join (", ", data));   // Display data...
}

GetNextFrame 是一个阻塞的方法，所以使用WithDegreeOfParallelism 来实现期望的同步。

## 修改并行化的值

在一个PLINQ查询中只能使用一次WithDegreeOfParallelism 。如果想再次调用，需要再次调用AsParallel：

"The Quick Brown Fox"
  .AsParallel().WithDegreeOfParallelism (2)
  .Where (c => !char.IsWhiteSpace (c))
  .AsParallel().WithDegreeOfParallelism (3)   // Forces Merge + Partition
  .Select (c => char.ToUpper (c))

# Cancellation 取消

# 任务并行机制

任务并行机制是用PFX实现并行化的最底层方法。这一层的类在System.Threading.Tasks命名空间，由下面的类组成：

Class         	                  | Purpose
Task	                            | For managing a unit for work
Task<TResult>            	| For managing a unit for work with a return value
TaskFactory	                  | For creating tasks
TaskFactory<TResult>  |	For creating tasks and continuations with the same return type
TaskScheduler               | For managing the scheduling of tasks
TaskCompletionSource | For manually controlling a task’s workflow

本质上，任务是轻量化管理并行单元的对象。task使用CLR的线程池来避免开启一个专用线程的开销，这个线程池和ThreadPool.QueueUserWorkItem用的是一样的。
在CLR 4.0 做了调整，使得Tasks更高效。

Task可以在想并行执行时使用。不过Task为了充分利用多核做了调整。实际上Parallel类和PLINQ内部都是建立在task并行上。

Task不仅仅是提供一个简单有效的方式来使用线程池，而且还提供了很多强大功能来管理工作单元，这包括：

* 协调任务安排
* 建立一个父子关系，当一个任务启用了一个任务
* 实现了联合取消
* 等待一组任务，而不需要一个信号量
* 加到继续执行的任务
* 在多个先行任务后安排一个后续任务
* 将异常传播到 父级、后续、和任务消费者

Task也实现了局部工作队列，可以快速的执行子任务，而不需要额外的花费，在那些自有单一工作队列。

## 建立和启动任务

第一篇提到可以使用Task.Factory.StartNew来创建并启动任务，同时传入一个Action代理：
Task.Factory.StartNew (() => Console.WriteLine ("Hello from a task!"));

泛型版本的Task<TResult>可以得到任务的执行完成后的返回值：

Task<string> task = Task.Factory.StartNew<string> (() =>    // Begin task
{
  using (var wc = new System.Net.WebClient())
    return wc.DownloadString ("http://www.linqpad.net");
});
 
RunSomeOtherMethod();         // We can do other work in parallel...
 
string result = task.Result;  // Wait for task to finish and fetch result.

Task.Factory.StartNew 将创建和开始任务一步就做完。当然可以先初始化一个Task对象，然后调用Start方法：

var task = new Task (() => Console.Write ("Hello"));
...
task.Start();

这种方式下创建的任务可以用RunSynchronously 代替Start，使得Task在调用者的线程上同步运行。（不开新的线程）
 
### 指定一个状态对象

Task.Factory.StartNew 初始化任务时，可以指定一个状态对象，它将传入到目标方法。当不想用lambda表达式，而想用个方法的时候，可以这么做：

static void Main()
{
  var task = Task.Factory.StartNew (Greet, "Hello");
  task.Wait();  // Wait for task to complete.
}
 
static void Greet (object state) { Console.Write (state); }   // Hello

假设已经用了lambda表达式，state object 可以赋值到一个有意义的名字给任务，然后使用AsyncState来查询名字。

static void Main()
{
  var task = Task.Factory.StartNew (state => Greet ("Hello"), "Greeting");
  Console.WriteLine (task.AsyncState);   // Greeting
  task.Wait();
}
 
static void Greet (string message) { Console.Write (message); }

###TaskCreationOptions

可以通过在任务StartNew的时候指定TaskCreationOptions 枚举值来协调任务执行。TaskCreationOptions 是位标识的枚举：

LongRunning
PreferFairness
AttachedToParent

LongRunning 要求调度器专门拿出一个线程给任务。这对于一直运行的任务很有好处，因为不这样的话它们会占据队列，
让短时运行的任务无意义的等待。

PreferFairness 告诉调度器尽量保证任务按照创建的顺序执行。调度器通常不会保证顺序，因为它在内部使用了局部的工作暂存队列。这个优化对那些小任务非常有利。

AttachedToParent 是建立子任务

### 子任务

当一个任务重启动另一个子任务，可以通过设置TaskCreationOptions.AttachedToParent在二者之间建立父子关系：

Task parent = Task.Factory.StartNew (() =>
{
  Console.WriteLine ("I am a parent");
 
  Task.Factory.StartNew (() =>        // Detached task
  {
    Console.WriteLine ("I am detached");
  });
 
  Task.Factory.StartNew (() =>        // Child task
  {
    Console.WriteLine ("I am a child");
  }, TaskCreationOptions.AttachedToParent);
});

子任务的作用就是当等待一个父任务完成时，会同时等待所有的子任务完成。

## 等待任务

可以明确等待任务完成用以下两种方法：

* 调用Wait方法（超时参数可选）
* 访问Result

也可以一次等待多个任务，使用Task.WaitAll方法和Task.WaitAny方法。

WaitAll和一次等待多个任务相似，但是效率更高，大概只需要一个上下文切换。同样如果任意一个任务抛出异常，WaitAll仍会等待所有的任务完成并且抛出一个 AggregateException异常，包含了每个失败任务的异常。等效下面的操作：

// Assume t1, t2 and t3 are tasks:
var exceptions = new List<Exception>();
try { t1.Wait(); } catch (AggregateException ex) { exceptions.Add (ex); }
try { t2.Wait(); } catch (AggregateException ex) { exceptions.Add (ex); }
try { t3.Wait(); } catch (AggregateException ex) { exceptions.Add (ex); }
if (exceptions.Count > 0) throw new AggregateException (exceptions);

调用WaitAny等效于等待[ManualResetEventSlim]()， 这个标记了每个任务完成。

有超时和传入取消标识到Wait方法，可以取消等待（不是取消任务）

## 错误处理

当等待任务完成时，任何为处理的异常都方便的抛到调用方，包装成一个AggregateException 异常。这通常无需在task里写异常处理的代码，可以这么写

int x = 0;
Task<int> calc = Task.Factory.StartNew (() => 7 / x);
try
{
  Console.WriteLine (calc.Result);
}
catch (AggregateException aex)
{
  Console.Write (aex.InnerException.Message);  // Attempted to divide by 0
}

对于那些分离的任务还是需要进行异常处理，否则会破坏整个应用，当任务脱离处理范围到垃圾回收时。同样在WaitAll超时后，任务会被脱离出来变成未处理的。

对于父任务，等待父任务的状态在子任务等待，任何子任务的异常都被冒泡到父任务：

TaskCreationOptions atp = TaskCreationOptions.AttachedToParent;
var parent = Task.Factory.StartNew (() => 
{
  Task.Factory.StartNew (() =>   // Child
  {
    Task.Factory.StartNew (() => { throw null; }, atp);   // Grandchild
  }, atp);
});
 
// The following call throws a NullReferenceException (wrapped
// in nested AggregateExceptions):
parent.Wait();

>>> 有趣的是当你检查Task的Exception 属性的时候，读取的动作会阻止异常破坏应用。关联关系是PFX的设计人员希望你不要忽略异常，但是当你知道会发生异常时，也不会说非要中断你的程序。

>>> 未处理的异常不会马上就导致应用中止。这会延迟到垃圾回收机捕捉到任务并调用销毁。中止被延迟是因为系统不知道你是否在垃圾回收前有计划调用Wait或Result或Exception属性。这个延迟可能让你的错误分析误入歧途。

下面还有个异常处理的方法 [continuations]().

## 取消任务

在新建任务的时候，允许传入一个[取消标识]()。这可以使得你有机会取消任务，使用公用的取消模式

var cancelSource = new CancellationTokenSource();
CancellationToken token = cancelSource.Token;
 
Task task = Task.Factory.StartNew (() => 
{
  // Do some stuff...
  token.ThrowIfCancellationRequested();  // Check for cancellation request
  // Do some stuff...
}, token);
...
cancelSource.Cancel();

要检查被取消的任务，捕捉AggregateException 并且检查内部的异常

try 
{
  task.Wait();
}
catch (AggregateException ex)
{
  if (ex.InnerException is OperationCanceledException)
    Console.Write ("Task canceled!");
}

如果任务没有开始就取消掉了，那任务不会被调度，直接就抛出一个 OperationCanceledException 。
因为 取消标识 被别的API认识，可以将他们传入别的构造函数，取消动作会无缝蔓延：

var cancelSource = new CancellationTokenSource();
CancellationToken token = cancelSource.Token;
 
Task task = Task.Factory.StartNew (() =>
{
  // Pass our cancellation token into a PLINQ query:
  var query = someSequence.AsParallel().WithCancellation (token)...
  ... enumerate query ...
});

在cancelSource上调用Cancel将取消PLINQ查询，并抛出OperationCanceledException 在task的内部，随后将引发task取消。

## Continuations

有时要在一个任务完成后，马上开始一个新任务。使用ContinueWith 方法就是干这个用的：

Task task1 = Task.Factory.StartNew (() => Console.Write ("antecedant.."));
Task task2 = task1.ContinueWith (ant => Console.Write ("..continuation"));

当task1完成后（成功、失败、取消），task2自动开始。如果task1在未运行到第二行的时候就执行完了，那task2会马上执行。ant参数是对前一个任务的引用。

例子里的情况和下面的相同：

Task task = Task.Factory.StartNew (() =>
{
  Console.Write ("antecedent..");
  Console.Write ("..continuation");
});






















