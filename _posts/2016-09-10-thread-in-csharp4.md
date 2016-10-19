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



# 优化PLINQ
## 输出的优化

PLINQ的优点之一就是可以方便地从并行工作中收集结果到一个单一的输出序列。如果仅仅是在序列的每个元素上执行一些函数:
foreach (int n in parallelQuery)
  DoSomething (n);

这种情况下不关心执行顺序，可以使用ForAll方法来提高效率。

ForAll方法在ParallenQuery的每个输出元素上执行一个代理。它直接钩在PLINQ内部，绕过了收集和列举结果的步骤

"abcdef".AsParallel().Select (c => char.ToUpper(c)).ForAll (Console.Write);

![forall](/img/post/9-10-theadprogram/ForAll.png)

收集和枚举结果不是非常复杂的操作，所以ForAll优化的结果最好的是那些数据量大并要求快速执行的输入。

## 输入端优化

PLINQ有三个讲输入元素分配到线程的划分策略：
Strategy 	Element allocation 	Relative performance
Chunk partitioning 	Dynamic 	Average
Range partitioning 	Static 	Poor to excellent
Hash partitioning 	Static 	Poor

对于那些需要比较元素的查询操作（GroupBy，Join，GroupJoin，Intersect，Except，Union，Distinct），没有选择只能使用 Hash划分。
Hash 划分相对低效因为要预先计算每个元素的hashcode，河阳有个hashcode标识的元素可以在同一线程上运行。如果发现执行太慢，可以调用AsSequential来取消并行。

对于其他的查询操作，需要选择是使用range还是chunck策略。默认上：

* 如果输入的序列是可索引的（数组或者IList<T>）,PLINQ选择的是range划分
* 否则是chunk划分

简单的说，range划分对于那些长的序列，序列中的元素执行花费的CPU时间差不多，那相对执行较快。否则chunk执行的快一点。

强制使用range划分

* 如果一个输入一Enumerable.Range开始，那么要使用ParallelEnumerable.Range代替。
* 另外可以在输入序列上简单调用ToList或ToArray（当然这个操作会引发性能的损失需要考虑到）

要强制使用chunk划分，序列使用Partitioner.Create做一下包装：

int[] numbers = { 3, 4, 5, 6, 7, 8, 9 };
var parallelQuery =
  Partitioner.Create (numbers, true).AsParallel()
  .Where (...)

Partitioner.Create的第二个参数指明希望是一个负载均衡的查询，也就是说要用chunk划分。

chunk划分的工作是让每个工作的线程定期的从输入元素里获取一小段元素来执行。PLINQ开始先划分成非常小的块（一次一个或两个元素），然后在查询执行过程中不断增加块的大小：
这保证小的序列可以高效并行，而大的序列又不会过多的来来回回。如果一个线程碰巧获取的一个简单元素（执行很快），那会接着获取更多的块。系统保证么个线程执行同等的工作。
唯一的缺点是才从共享的的输入序列中取元素时要同步（一般是排他锁），这需要额外的花费和内容。

![part](/img/post/9-10-theadprogram/Partitioning.png)

# 并发集合 Concurrent Collections

Framework4.0 提供了一套新的集合在System.Collections.Concurrent 命名空间下。所有的这些都是完完全全线程安全的：

Concurrent collection 	Nonconcurrent equivalent
ConcurrentStack<T> 	Stack<T>
ConcurrentQueue<T> 	Queue<T>
ConcurrentBag<T> 	(none)
BlockingCollection<T> 	(none)
ConcurrentDictionary<TKey,TValue> 	Dictionary<TKey,TValue>

并发集合通常在需要线程安全的场合。然后有几个注意事项：
* 并发集合是用来给并行程序。除了并发的情况，其他条件下，传统的集合表现的更好。
* 线程安全的集合，不能保证使用代码的是线程安全的。
* 如果一个线程读，另一个线程修改，那么不会有异常，但是读到的是新旧都有的。
* 没有并发版本的List<T>
* 并发的栈、队列和包内部都是链表。这使得先对于非并发的Stack和Queue类的内存效率要低，但是对并发获取更好，因为链表实现可以不用锁或者少量锁。（链表插入只需要更新两个引用）

也就是说，并发集合不是在一般集合上加个锁。为了说明这点，可以在一个线程上执行下面的代码

ar d = new ConcurrentDictionary<int,int>();
for (int i = 0; i < 1000000; i++) d[i] = 123;

这段代码要比后面这段慢3倍

var d = new Dictionary<int,int>();
for (int i = 0; i < 1000000; i++) lock (d) d[i] = 123;

（读起来快是因为读的时候不加锁）

并发集合与一般的集合使用上也不一样，如果想要执行一些特殊的方法实现原子的尝试执行，如TryPop。大部分的方法没有在ProducerConsumerCollection<T>接口上实现。

## ProducerConsumerCollection<T>

一个生产/消费的集合主要有两个主要作用：
* 增加一个元素（生产）
* 获取一个元素并移除（消费）

经典的示例就是栈和队列。生产者/消费者集合在并行编程中意义重大，因为这将有益于高效的无锁的实现。

ProducerConsumerCollection<T>接口代表了一个线程安全的 生产者/消费者 集合。下面的类实现了这个接口：

ConcurrentStack<T>
ConcurrentQueue<T>
ConcurrentBag<T>

IProducerConsumerCollection<T> 继承自 ICollection，并增加了一下方法：

void CopyTo (T[] array, int index);
T[] ToArray();
bool TryAdd (T item);
bool TryTake (out T item);

TryAdd和TryTake方法检查是否可以执行增/删操作，如果可以就马上执行。检查和执行是原子的，去除了在一般集合范围内加锁的要求。
int result;
lock (myStack) if (myStack.Count > 0) result = myStack.Pop();

TryTake返回false，如果集合是空，TryAdd总是成功返回true。如果写自己的的同步集合不允许重复，那么久要让TryAdd返回false，当检查到有重复的元素

TryTake拿出来的元素在各个子类中定义：
* 对于栈，TryTake移除最近增加的元素，
* 对于队列，TryTake移除最先增加的元素，
* 对于包，TryTake移除随意一个元素。

三个同步类明确实现了TryTake和TryAdd，并且更通常意义的刚发TryDequeue和TryPop。

#ConcurrentBag<T>

ConcurrentBag<T>存储了一个没有排序的对象集合（可以重复）。ConcurrentBag<T>适合不在意Take或TryTake的是哪个元素。

ConcurrentBag<T>的优点是在多个线程同时调用Add方法时，并不会受到竞争。相反在队列和栈上的Add有竞争（虽然比非同步集合上的包围lock要少）。
同样取也是非常高效的，前提是每个线程不要取得比加入的多。

在同步包内部，每个线程有各自私有的链接。线程上调用Add加入元素到各自私有的列表上，减少了竞争。当枚举包的时候，枚举器会跨越每个线程的私有列表，依次返回元素。

当调用Take，包首先查看当前线程的私有列表，如果有元素，那就直接拿过出去，基本没有竞争。如果列表空着，就要从别的线程里拿一个元素，就会有竞争。

因此，包的Take返回的是本线程最近加入的元素，或者随意另一个线程最新加入的元素。

同步包适用于那些并行操作大部分是Add元素，或者Add和Take在一个线程上均衡的情况。前面的一个例子就是，当使用Parallel.ForEach实现一个检查器时：

var misspellings = new ConcurrentBag<Tuple<int,string>>();
 
Parallel.ForEach (wordsToTest, (word, state, i) =>
{
  if (!wordLookup.Contains (word))
    misspellings.Add (Tuple.Create ((int) i, word));
});

对于一个生产/消费的队列，同步包是个poor choice，因为元素加入和删除在不同的线程上。

# BlockingCollection<T>

如果在上面的生产/消费集合上调用TryTake，并且集合是空的，返回值是false。有的场景下等待一个可用的元素也是很有用的。

除了重写TryTake，PFX把这个设计封装到一个包装类BlockingCollection<T>。一个阻塞集合可以包装在任意实现IProducerConsumerCollection<T>接口的集合，并且可以从被包装的集合取袁术，如果没有的话就阻塞，直到

一个阻塞的集合可以限制集合的大小，如果超出大小就让生产者阻塞。有这种限制的集合称为边界阻塞集合(bounded blocking collection)。

要使用BlockingCollection<T>：

1. 实例化类，可以指定要包装的IProducerConsumerCollection<T>和集合的最大值。
2. 调用Add或TryAdd加入到下层的集合
3. 调用Take或TryTake来使用消费元素

如果在构造函数没有传入一个集合，会自动初始化一个ConcurrentQueue<T>。生产和消费方法可以指定一个取消标识和超时时间。Add和TryAdd在集合边界大小会阻塞；Take和TryTake集合在集合空 的时候会阻塞。

另一个消费元素的方法是GetConsumingEnumerable。这个方法会无限得返回生产的元素。可以强制停止顺序通过调用CompleteAdding，同样也阻止更多元素加入。

前面已经写了一个生产\消费队列的例子，用的是[wait and pulse](http://)。 现在使用BlockingCollection<T>来重构这个类：

public class PCQueue : IDisposable
{
  BlockingCollection<Action> _taskQ = new BlockingCollection<Action>(); 
  public PCQueue (int workerCount)
  {
    // Create and start a separate Task for each consumer:
    for (int i = 0; i < workerCount; i++)
      Task.Factory.StartNew (Consume);
  }
 
  public void Dispose() { _taskQ.CompleteAdding(); }
 
  public void EnqueueTask (Action action) { _taskQ.Add (action); }
 
  void Consume()
  {
    // This sequence that we’re enumerating will block when no elements
    // are available and will end when CompleteAdding is called. 
    foreach (Action action in _taskQ.GetConsumingEnumerable())
      action();     // Perform task.
  }
}

由于没有传入任何参数到BlockingCollection的构造函数，它会自动实例化一个同步队列。如果传入的是ConcurrentStack，那就要在最后停止一个栈。

BlockingCollection同时提供了静态方法叫AddToAny和TakeFromAny，这允许增加获取一个元素当指定几个阻塞序列。这个动作会在第一个可以服务请求的集合上执行。

#TaskCompletionSource的优势

上面写的生产/消费程序僵硬，因为不能跟踪入队的工作项。如果能够的话就好了：

* 知道工作项已经结束
* 取消一个未开始的工作项
* 优雅的处理工作项抛出的异常

一个理想的方案是让EnqueueTask返回一些对象来实现上面说的功能。好消息是已经有这么个类啦——Task类。需要做的是劫持task的控制权通过TaskCompletionSource：

public class PCQueue : IDisposable
{
  class WorkItem
  {
    public readonly TaskCompletionSource<object> TaskSource;
    public readonly Action Action;
    public readonly CancellationToken? CancelToken;
 
    public WorkItem (
      TaskCompletionSource<object> taskSource,
      Action action,
      CancellationToken? cancelToken)
    {
      TaskSource = taskSource;
      Action = action;
      CancelToken = cancelToken;
    }
  }
 
  BlockingCollection<WorkItem> _taskQ = new BlockingCollection<WorkItem>();
 
  public PCQueue (int workerCount)
  {
    // Create and start a separate Task for each consumer:
    for (int i = 0; i < workerCount; i++)
      Task.Factory.StartNew (Consume);
  }
 
  public void Dispose() { _taskQ.CompleteAdding(); }
 
  public Task EnqueueTask (Action action) 
  {
    return EnqueueTask (action, null);
  }
 
  public Task EnqueueTask (Action action, CancellationToken? cancelToken)
  {
    var tcs = new TaskCompletionSource<object>();
    _taskQ.Add (new WorkItem (tcs, action, cancelToken));
    return tcs.Task;
  }
 
  void Consume()
  {
    foreach (WorkItem workItem in _taskQ.GetConsumingEnumerable())
      if (workItem.CancelToken.HasValue && 
          workItem.CancelToken.Value.IsCancellationRequested)
      {
        workItem.TaskSource.SetCanceled();
      }
      else
        try
        {
          workItem.Action();
          workItem.TaskSource.SetResult (null);   // Indicate completion
        }
        catch (OperationCanceledException ex)
        {
          if (ex.CancellationToken == workItem.CancelToken)
            workItem.TaskSource.SetCanceled();
          else
            workItem.TaskSource.SetException (ex);
        }
        catch (Exception ex)
        {
          workItem.TaskSource.SetException (ex);
        }
  }
}

在EnqueueTask里，入队的工作项封装了目标代理和一个任务完成源，这个源让我们后面可以控制给消费者的任务。

在Consume函数里，首先检查任务是否在出队时被取消。如果没有取消，就运行代理并调用SetResult在类完成源来只是工作项完成。

下面是如何使用这个类：

var pcQ = new PCQueue (1);
Task task = pcQ.EnqueueTask (() => Console.WriteLine ("Easy!"));
...

现在可以等待任务，执行后续操作、在父任务上继续传播异常等等。一句话，在执行我们自己的调度器时，得到了一个更丰富的任务模型。

# SpinLock and SpinWait

在并行编程里，一个简单的spinning通常要比blocking更好，因为避免了上下文的切换和内核转换。SpinLock and SpinWait就是来干这个的。
通常是用来写定制化的同步构造函数。

>>> 注意： SpinLock and SpinWait 是结构，不是类！这样设计是醉倒程度的优化，来避免间接引用？？和垃圾回收。当然这就意味着必须小心不要进行无疑义的拷贝——
方法之间传递的时候没有ref修饰，定义为readonly字段。这在用SpinLock尤其要注意。

## SpinLock

SpinLock结构体可以加锁但不会有上下文切换的开销，花费在保持一个线程轮询（无疑义的忙）。这个方法可以用在高竞争的场景，当加锁的操作很简单。

>>> 如果使用spinlock竞争时间很长（微秒级），那他会产生时间切片，导致上下文切换，就像常规锁。当重新调度后，它会重新产生-在一个不断的 spin yileding。
这样的消费CPU资源比彻底的spinning要少，但是比blocking要多。

使用SpinLock和使用平常锁一样，除了以下注意点：

* Spinlock自旋锁是结构体
* 自旋锁不能再次进入，就是说不能一个线程的一行上对同一个SpinLock抵用两次Enter。如果违反这个规则，会引发一个异常或死锁。可以在初始化的时候设置是否启用owner tracking。这会有性能损失。
* SpinLock可以查询锁是否使用，通过IsHeld属性，如果owner tracking启用了，那查看IsHeldByCurrentThread属性。
* 没有SpinLock语法糖和C#的lock

另一个不同是当调用Enter，必须遵循一个[lockTaken参数的强壮模式]() （就是总是在一个try/finally块里工作）

一个例子：

var spinLock = new SpinLock (true);   // Enable owner tracking
bool lockTaken = false;
try
{
  spinLock.Enter (ref lockTaken);
  // Do stuff...
}
finally
{
  if (lockTaken) spinLock.Exit();
}

在一般的锁，lockTaken会是false在调用Enter后而Enter方法跑出了一个异常，所以锁没有真的发生。这种情况非常罕见，使用这个变量可以让后续是否调用Exit更确定。

SpinLock也提供了一个TryEnter方法可以接收一个超时参数。

>>> 提供SpinLock没有得到语义的价值，并且缺少语法糖。几乎是每次使用都要收到折磨！仔细来决定是否要废除普通的锁。

SpinLock意义在于写自己的可重用的同步构造方法。即使这样，自旋锁也不是听起来那样有用。通常更好的方法是花点时间来时做更投机的——使用SpinWait.

# SpinWait

SpinWait可以写出没有锁的代码使用spin代替阻塞。它工作在实现保护来防止资源接和优先级翻转这些有spin依法的危机。

>>> Lock-free 编程用SpinWaite是和多线程一样的硬骨头，并且没有更高层的构造再来做这个事情。





















