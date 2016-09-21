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

# 纯粹的功能

因为PLINQ并行线程上执行查询，所以必须小心不要执行线程不安全的操作，特别是写到变凉里的副作用而引发的线程不安全。

// The following query multiplies each element by its position.
// Given an input of Enumerable.Range(0,999), it should output squares.
int i = 0;
var query = from n in Enumerable.Range(0,999).AsParallel() select n * i++;

当然可以用锁或者是Interlock，但是问题依然存在，没有在对应的输入上有正确的值。加上AsOrdered也不管用，因为AsOrdered只是保证了输出的顺序，而不是执行的顺序，实际上也不是顺序执行。

因此执行可以替换为使用indexed版本的Select
var query = Enumerable.Range(0,999).AsParallel().Select ((n, i) => n * i);

为了达到最高效，所有从











