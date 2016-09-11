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

调用AsSequential()方法会将


