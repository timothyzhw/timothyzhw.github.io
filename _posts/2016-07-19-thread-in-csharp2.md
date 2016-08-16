---
layout:     post
title:      "Threading in C# "
subtitle:   "PART 2: BASIC SYNCHRONIZATION"
date:       2016-07-13
author:     "Joseph Albahari, Trans by Tim"
header-img: "/img/post/7-19-threading/bk.jpg"
catalog: true
tags:
    - c#
    - Threading
---

> 上篇博客.Net多线程实现上，用Task是个不错的选择。

# 线程同步的要点

目前为止，已经知道如何在线程上开始工作，如何传递数据，如何使用局部变量，如何在线程之间共享交互。
下一步是线程同步：保证线程一致工作，得到预期的结果。在多个线程操作同一份数据时，同步就尤为重要。在这个地方很容易出错。

同步的过程可以分为四类：
1. 简单的阻塞方法
线程可以等一会儿或者等到别的线程完成后再执行。可以使用 **Sleep**、 **Join**、 **Task.Wait** 等简单的阻塞方法。
2. 锁装置
限制同一时间只能有部分线程可以执行。最常用的是排他锁，只有一个进程竞争到资源，可以任意使用公共资源不受其他进程干扰。关于锁，**lock (Monitor.Enter/Monitor.Exit), Mutex, and SpinLock . The nonexclusive locking constructs are Semaphore, SemaphoreSlim, and the reader/writer locks **。 
3. 信号装置
一个线程会暂停，直到接收到一个信号后再执行。这可以避免低效的轮询操作。一般有两种信号装置：**event wait handles 和 Monitor’s Wait/Pulse**。在.Net 4.0又引入了 **CountdownEvent 和 Barrier**
4. 不阻塞的同步装置
这种保护措施是基于处理器的原始功能。CLR和C#提供了以下方法：**Thread.MemoryBarrier, Thread.VolatileRead, Thread.VolatileWrite, the volatile keyword, and the Interlocked class**.

> 看到这么多名词，都要吓哭了。一个一个排着看吧。

## Blocking

一个线程由于某种原因（Sleep，Join，EndInvoke）暂停执行，那就处于阻塞状态。阻塞的线程会立刻交出占用的处理器周期，不耗用处理器，直到阻塞的条件结束。可以通过 **ThreadState** 属性来检查线程是否阻塞。
bool blocked = (someThread.ThreadState & ThreadState.WaitSleepJoin) != 0;
注意：只能在测试时用，因为在条件判定时，线程状态也会随时会变化。

当线程阻塞和解除阻塞时，都会有上下文的切换，这大概要用几微秒。解除阻塞有四种方法（不包括摁电脑的Power键）；
* 阻塞的条件已满足
* 操作时间超时
* Thread.Interrupt 中断
* Thread.Abort 退出

Suspend方法挂起的线程，不认为是阻塞的线程。（Suspend 方法已经不推荐用）

## 阻塞 Vs 轮询

有些情况要求线程暂停，待满足一定条件后才能执行。标志位和锁都可以实现线程阻塞，也可以不停的循环执行，直到条件满足。如：
while (!proceed);

while (DateTime.Now < nextStartTime);

一般来说，这会造成处理器的巨大浪费。CLR和操作系统认为线程在执行非常重要的计算，会分配相应的资源。

有时可以将阻塞和轮询混用：

while (!proceed) Thread.Sleep (10);

虽然不优雅，但是比一直循环要有效的多。当然在判断proceed的状态时，会有一致性的问题，正确的使用locking和signaling可以避免。

## 线程状态

可以通过 ThreadState 属性来查询线程的执行状态，返回值是ThreadState的枚举。这个值是按位存储的三‘层’。当然大部分状态是冗余无用重复的：
![state](/img/post/7-19-threading/state.png)

ThreadState 常用的四个状态值 Unstarted, Running, WaitSleepJoin, Stopped 。下面代码判断是否处于这四个状态：
public static ThreadState SimpleThreadState (ThreadState ts)
{
  return ts & (ThreadState.Unstarted |
               ThreadState.WaitSleepJoin |
               ThreadState.Stopped);
}

ThreadState属性用于诊断，不适用状态同步处理，因为判断状态后，再根据状态执行时，状态可以就已经变化了。

# 锁

排它锁可以保证一个时间内只有一个线程在运行某段代码。.Net 里有两个实现方式 lock 和 Mutex。lock 快捷易用，Mutex可以跨越程序在计算机进程间加锁。

### lock

class ThreadUnsafe
{
  static int _val1 = 1, _val2 = 1;
 
  static void Go()
  {
    if (_val2 != 0) Console.WriteLine (_val1 / _val2);
    _val2 = 0;
  }
}

这个类不是线程安全的：如果Go被两个线程同时调用，就可能会抛出除以 0 的异常，条件判断为真后，_val2被修改。

使用lock可以解决：

class ThreadSafe
{
  static readonly object _locker = new object();
  static int _val1, _val2;
 
  static void Go()
  {
    lock (_locker)
    {
      if (_val2 != 0) Console.WriteLine (_val1 / _val2);
      _val2 = 0;
    }
  }
}

同一时间，只有一个线程可以锁定同步对象_locker,其他竞争线程被阻塞，直到锁被解除。
如果多个线程竞争这个锁，那会被加入到一个队列，按照先进先出逐一赋予锁的权利。
排它锁也可以认为是强行将锁保护的操作排序，线程之间不会有重叠。

等待锁的进程被阻塞，ThreadState 属性值为 WaitSleepJoin。 后面章节里会有如何强制结束其他线程的锁。
 
<div class="sidebar">
<p class="sidebartitle">A Comparison of Locking Constructs</p>

<table border="1" cellspacing="0" cellpadding="0">
	<tbody><tr>
		<th valign="top">Construct</th>
		<th valign="top">Purpose</th>
		<th valign="top">Cross-process?</th>
		<th valign="top">Overhead*</th>
	</tr>
	<tr>
		<td valign="top">
			<a href="#_Locking">lock</a> (<code>Monitor.Enter</code> / <code>Monitor.Exit</code>)</td>
		<td valign="middle" rowspan="2">Ensures just one thread can access a resource, or section of code at a time</td>
		<td valign="top">-</td>
		<td valign="top">20ns</td>
	</tr>
	<tr>
		<td valign="top">
			<a href="#_Mutex">Mutex</a>
		</td>
		<td valign="top">Yes</td>
		<td valign="top">1000ns</td>
	</tr>
	<tr>
		<td valign="top">
			<a href="#_Semaphore">SemaphoreSlim</a> (introduced in Framework 4.0)</td>
		<td valign="middle" rowspan="2">Ensures not more than a specified number of concurrent threads can access a resource, or section of code</td>
		<td valign="top">-</td>
		<td valign="top">200ns</td>
	</tr>
	<tr>
		<td valign="top">
			<a href="#_Semaphore">Semaphore</a>
		</td>
		<td valign="top">Yes</td>
		<td valign="top">1000ns</td>
	</tr>
	<tr>
		<td valign="top">
			<a href="part4.aspx#_Reader_Writer_Locks">ReaderWriterLockSlim</a> (introduced in Framework 3.5)</td>
		<td valign="middle" rowspan="2">Allows multiple readers to coexist with a single writer</td>
		<td valign="top">-</td>
		<td valign="top">40ns</td>
	</tr>
	<tr>
		<td valign="top">
			<a href="part4.aspx#_Reader_Writer_Locks">ReaderWriterLock</a> (effectively deprecated)</td>
		<td valign="top">-</td>
		<td valign="top">100ns</td>
	</tr>
</tbody></table>

<p>*Time taken to lock and unlock the construct once on the
same thread (assuming no blocking), as measured on an Intel Core i7 860. </p>

</div>

## Monitor.Enter and Monitor.Exit

C#的lock 状态实际是一个语法糖，实际上是使用try/finally来调用Monitor.Enter 和 Monitor.Exit 。下面代码是简化版的 Go 方法的内部实现：

Monitor.Enter (_locker);
try
{
  if (_val2 != 0) Console.WriteLine (_val1 / _val2);
  _val2 = 0;
}
finally { Monitor.Exit (_locker); }

如果在Monitor.Exit前没有对同一对象调用Monitor.Enter就会抛出异常。

## Monitor.Enter 的 lockTaken 重载

上面的代码是C# 1.0-3.0时，编译器在处理lock语句的转换。
但是上面代码有个很隐晦的缺陷，假设在Monitor.Enter 的实现中，或者在Monitor.Enter和try之间有异常。（线程被意外终止或内存溢出）。如果这时候锁已经拿到，那就不会释放，因为没有机会进入try/finally。锁就被泄露了。

为了避免这种情况，CLR 4.0 里增加了一个Monitor.Enter的重载：

public static void Enter (object obj, ref bool lockTaken);

lockTaken 设置为false，当Enter方法有异常并且没有拿到锁。

下面是C#4.0 中的lock的编译
bool lockTaken = false;
try
{
  Monitor.Enter (_locker, ref lockTaken);
  // Do your stuff...
}
finally { if (lockTaken) Monitor.Exit (_locker); }

## TryEnter

Monitor 提供了一个TryEnter 方法，允许设置一个超时时间。如果方法获取锁，返回为true，如果因为超时没有获得锁，那返回false。TryEnter可以不输入参数，可以测试锁，如果不能马上获得锁就立刻超时。

## 选择合适的同步对象

只要是线程可以访问到的对象，都可以用来做同步锁。需要注意的是，必须是引用类型的对象。同步的对象一般是私有的，是一个实例或静态字段。同步的对象可以是要保护的对象本身
class ThreadSafe
{
  List <string> _list = new List <string>();
 
  void Test()
  {
    lock (_list)
    {
      _list.Add ("Item 1");
      ...

可以定义特定的对象（如前面的_locker）作为控制精确范围和粒度的锁。也可以使用this或者类型Type：
lock (this) { ... }
or:

lock (typeof (Widget)) { ... }    // For protecting access to statics

缺点是没有封装好锁的逻辑，很容易造成死锁和过度锁。用类型来锁，将锁延伸到了整个应用程序的边边角角。

还有，可以锁定 lambda表达式或者匿名方法定义的局部变量。

> 注意：锁并不是限制对同步对象本身的访问。如一个线程使用了lock(x)，其他线程仍然可以使用x.ToString()。要想阻塞必须是都调用lock(x)

## lock的时机

基本原则就是需要在操作“共享写字段”的周围加锁。即使是最简单的赋值也要考虑到同步。下面的例子里，Increment nor the Assign













