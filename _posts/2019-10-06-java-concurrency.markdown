---
layout:     post
title:      "Talk about java concurrency (1)"
subtitle:   " \"Little share in group\""
date:       2019-10-06 22:36:00
author:     "vikeria"
header-img: "img/in-post/java-concurrency/bg.jpg"
tags:
    - Java
    - 并发
---

> “Come on!”

## 前言
国庆前，组内讨论了一下大家希望了解的java相关知识。java并发以13/22居榜首，由我这边认领做一下简单的分享。正文即开始正式相关内容。

## 正文

### Java并发问题是如何产生的
先看两段代码

多线程操作共享变量导致的问题
```java
public class MyTest {
	private static long count = 0L;
	public static void main(String[] args) {
		for (int i = 0; i < 1000; i++) {
			new Thread(new Runnable() {
				@Override
				public void run() {
					count++;
				}
			}).start();
		}
		System.out.println(count); //输出的不一定是1000
		System.exit(0);
	}
}
```
代码乱序执行优化问题（仅举例，实际编译视编译器实际处理）
```java
public class MyTest {

	private static boolean flag = false;

	private static Integer i = null;

	public static void main(String[] args) {
		new Thread(new Runnable() {
			@Override
			public void run() {
			    // 下面两行没有依赖关系，可能发生重排序，即flag设置为true，i还未赋值
				i = new Integer(10);
				flag = true;
			}
		}).start();

		new Thread(new Runnable() {
			@Override
			public void run() {
				if(flag) {
					System.out.print(i); // 可能i还未赋值，为null
				}
			}
		}).start();
	}
}
```
并发问题 -> 多线程操作共享变量导致的问题，代码乱序执行优化问题

```Java
public class MyTest{
	public  int a = 0;
	public void inc(){
		a++ ;
	}
}
```
将上述代码反解析一下
```Java
 public void inc();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #12                 // Field a:I
         5: iconst_1
         6: iadd
         7: putfield      #12                 // Field a:I
        10: return
      LineNumberTable:
        line 6: 0
        line 7: 10
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Ltest/MyTest;
```
可以看到inc方法中，完成一个自加有很多jvm指令，在这一过程中，cpu可能会切换线程。线程T1挂起后其他线程T2修改了a的值，线程T1恢复之后再写入就会有问题。
```Java
0：将对象本身入栈，这里隐含第一个参数为this;
1：复制栈顶元素，也就是对象的引用；
2：获取栈顶对象属性a的值，并将其入栈；
5：将int型1推送至栈顶；
6：将栈顶两个int类型数据相加，并将结果入栈；
7：将栈顶的值放入对象属性a中；
10：return
```

这里就会提到原子性及可见性。
+ 原子性
    - 原子性是指一个操作是不可中断的，即多线程环境下，操作不能被其他线程干扰
+ 可见性
    - 可见性是指当一个线程修改了某一个共享变量的值，其他线程是否能够立即知道该变更

**显然前诉的a++并不是一个原子性操作，且某一线程完成自加后，对其他线程来说a的值并不立即可见。**

线程间并发问题的解决方案 -> 线程间同步和通信（大多数是为了解决同步问题），内存屏障（可见性）

操作系统中线程间同步和通信的方式：
+ 互斥锁 -> Synchronized(1.0)/Lock(1.5)
+ 条件变量 -> Condition(1.5) *可以以原子的方式阻塞进程，直到某个特定条件为真为止。对条件测试是在互斥锁的保护下进行的。条件变量始终与互斥锁一起使用。Condition condition = lock.newCondition();*
+ 信号量 -> Semaphore(1.5)

可见性、内存屏障 -> volatile

从以上可以看出，实际jvm在jdk中提供的线程间做同步和通信的工具类，在操作系统的设计中都是有迹可循的。

那么为什么一个简单的a++需要那么多的jvm指令呢？

### Java内存模型
##### java运行时数据区
![java运行时数据区](/img/in-post/java-concurrency/java-runtime-memory-model.png)

![java内存模型](/img/in-post/java-concurrency/java-memory-model.png)

Java运行时数据区把Java虚拟机内部划分为线程栈和堆。方法区、堆是线程共享的，当多个线程访问到共享变量（实例字段、静态字段和构成数组元素的对象），就可能会发生并发问题。

##### java内存模型（抽象概念，并非真实存在）
![java内存模型](/img/in-post/java-concurrency/java-memory-model-detail.png)

##### java内存交互的基本操作
![java内存交互](/img/in-post/java-concurrency/java-memory-operate.png)

虚拟机实现时必须保证下面介绍的每种操作都是原子的，不可再分的(对于 double 和 long 型的变量来说，load、store、read、write 操作在某些平台上允许有例外）。

如果要把一个变量从主内存中复制到工作内存，就需要按顺寻地执行read和load操作，如果把变量从工作内存中同步回主内存中，就要按顺序地执行store和write操作。Java内存模型只要求上述操作必须按顺序执行，而没有保证必须是连续执行。也就是read和load之间，store和write之间是可以插入其他指令的，如对主内存中的变量a、b进行访问时，可能的顺序是read a，read b，load b， load a。

为什么内存模型设计成这样？

### 物理机模型
![物理机模型](/img/in-post/java-concurrency/physicalmachine-memory-model-detail.png)
+ 硬件的效率问题
*cpu和内存之间的运算速度有几个数量级的差距*
+ 缓存作为内存和处理器之间的缓冲
*将运算需要使用到的数据复制到缓存中，让运算能快速运行，当运算结束后再从缓存同步回内存之中*

这样的结构同样存在并发问题，同样要解决原子性、可见性的问题。

物理机并发问题的解决有两种方式：总线锁、缓存一致性。总线锁为了达到独占共享内存的目的，锁住了cpu和内存交互的总线，消费太高，那么就剩下了缓存一致性。

常见的缓存一致性协议是 [MESI协议](https://en.wikipedia.org/wiki/MESI_protocol)，intel和amd分别在此基础上继续演进。也可以参见[这篇博文](https://www.cnblogs.com/z00377750/p/9180644.html)。

除了以上的问题，硬件上的优化及编译期的优化，还存在着其他问题，如指令重排序、流水线优化（对此有兴趣的同学可以看一看计算机组成原理）等都可能使处理器执行顺序和程序次序不一致。想了解更多的关于内存屏障的内容，可以查看 [Memory Barriers: a Hardware View for Software Hackers](http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.06.07c.pdf)、[Why Memory Barrier？](https://sstompkins.wordpress.com/2011/04/12/why-memory-barrier%EF%BC%9F/) 了解更多关于硬件上引入store buffer/validate queue提升cpu效率，并使用memory barries解决引入之后的问题。

后面我们讲到volatile关键字的时候还会提到这个协议，此处不再赘述。

### 缓存一致性问题（衍生）
+ 物理机模型
+ JMM内存模型
+ redis/memcache 缓存和数据库一致性等等（cache aside）

### java内存和物理机模型的关系
![java内存和物理机模型的关系](/img/in-post/java-concurrency/jmm-physical-relation.png)

此处仅做对比，并非实际的物理映射关系。

### 目前为止的JMM总结
+ Java线程间的通信由Java内存模型(JMM)控制，JMM决定一个线程对共享变量的写入何时以及如何变成对另一个线程可见
+ JMM是一个抽象的概念，并非真实存在，它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器的优化
+ JMM定义了线程和主内存之间的抽象关系
    - 线程之间的共享变量存储在主内存中（从硬件角度来说就是内存条）
    - 每个线程都有一个私有的本地内存，本地内存中存储了该线程用来读/写共享变量的副本（从硬件角度来说就是CPU的缓存，比如寄存器、L1、L2、L3缓存等）
    - 同时JVM通过JMM来屏蔽各个硬件平台和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访问效果

### JMM内存交互协议（运行规则）
##### 内存交互基本操作的三个特性
+ 原子性
    - 原子性是指一个操作是不可中断的，即多线程环境下，操作不能被其他线程干扰
+ 可见性
    - 可见性是指当一个线程修改了某一个共享变量的值，其他线程是否能够立即知道该变更
    - java中普通的共享变量不保证可见性，因为其的修改被写入内存的时机是不确定的，多线程并发下很可能出现"脏读"
    - 缓存优化或者硬件优化或指令重排以及编辑器的优化都可能导致一个线程修改不会立即被其他线程察觉
    - Java提供volatile保证可见性：写操作立即刷新到主内存，读操作直接从主内存读取
    - Java同时还可以通过加锁的同步性间接保证可见性：synchronized和Lock能保证同一时刻只有一个线程获取锁并执行同步代码，并在释放锁之后将变量的修改刷新到主内存中
+ 有序性
    - 对于一个线程的执行代码而言，我们总是习惯性认为代码的执行总是从上到下，有序执行
    - 但为了提供性能，编译器和处理器通常会对指令序列进行重新排序
    - 指令重排可以保证串行语义一致，但没有义务保证多线程间的语义也一致，即可能产生"脏读"

##### 8 种操作同步的规则
Java内存模型还规定了执行上述8种基本操作时必须满足如下规则:

+ 不允许read和load、store和write操作之一单独出现（即不允许一个变量从主存读取了但是工作内存不接受，或者从工作内存发起会写了但是主存不接受的情况），以上两个操作必须按顺序执行，但没有保证必须连续执行，也就是说，read与load之间、store与write之间是可插入其他指令的。
+ 不允许一个线程丢弃它的最近的assign操作，即变量在工作内存中改变了之后必须把该变化同步回主内存。
+ 不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主内存中。
+ 一个新的变量只能从主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量，换句话说就是对一个变量实施use和store操作之前，必须先执行过了assign和load操作。
+ 一个变量在同一个时刻只允许一条线程对其执行lock操作，但lock操作可以被同一个条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。
+ 如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作初始化变量的值。
+ 如果一个变量实现没有被lock操作锁定，则不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁定的变量。
+对一个变量执行unlock操作之前，必须先把此变量同步回主内存（执行store和write操作）。

我们重点关注的是针对lock的几条规则，很明显的是针对某一线程做lock操作后，在unlock之前的操作就满足了原子性、可见性以及有序性，所以是线程安全的。

Java的虚拟机并没有把lock和unlock操作直接开放给用户，而是提供了更高层次的字节码指令monitorenter和moniterexit来隐式的使用这两个操作。反映到Java代码中就是synchornized关键字。

##### Happends-Before规则
指令重排是有原则的，并非所有的指令都可以随便改变执行位置，如以下8种情况：
+ 程序顺序原则： 一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作
+ volatile规则： volatile变量的写先发生于读，这保证了volatile变量的可见性
+ 锁规则： 解锁必然发生在随后的加锁之前
+ 传递规则： A先于B，B先于C，那么A必然先于C
+ 线程启动规则： 线程的start()方法先于它的每一个动作
+ 线程中断规则： 线程的中断(Thread.interrupt())先于被中断线程的代码
+ 线程终止规则： 线程的所有操作先于线程的终止检测(Thread.join()方法结束，Thread.isAlive()检测)
+ 对象终结规则： 对象的构造函数执行(初始化)结束先于finalize()方法

##### Volatile
前面我们提到synchronized是可以满足原子性、可见性以及有序性的。那这边的volatile是只满足了可见性、有序性。

（一）可见性

先看一段代码
```Java
public class VolatileTest {
	// 添加volatile关键字
	private static volatile boolean initFlag = false;
	// 不添加volatile关键字
	// private static boolean initFlag=false;

	public static void main(String[] args) throws InterruptedException {
		// 线程1
		new Thread(new Runnable() {
			@Override
			public void run() {
				System.out.println("线程1开始执行...");
				while (!initFlag) {

				}
				System.out.println("线程1执行结束...");
			}
		}).start();

		Thread.sleep(2000);
		// 线程2
		new Thread(new Runnable() {
			@Override
			public void run() {
				System.out.println("线程2开始执行...");
				initFlag = true;
				System.out.println("线程2修改initFlag为true,线程结束...");
			}
		}).start();
	}
}
```
不添加volatile关键字的场景下，执行结果如下：
```Java
线程1开始执行…
线程2开始执行…
线程2修改initFlag为true,线程结束…
```
内存模型及流程示意图：

![内存模型及流程示意](/img/in-post/java-concurrency/no-volatile-operate.png)

添加volatile关键字的场景下，执行结果如下：
```Java
线程1开始执行…
线程2开始执行…
线程2修改initFlag为true,线程结束…
线程1执行结束…
```
内存模型及流程示意图：

![内存模型及流程示意](/img/in-post/java-concurrency/volatile-operate.png)

volatile缓存可见性实现原理:

底层实现主要是通过汇编lock前缀, 它会锁定这块内存区域的缓存(缓存行锁定) 并回写到主内存

IA-32(英特尔处理器)结构软件开发者手册对lock指令的解释:
- 会将当前处理器缓存行的数据立即写回到系统内存
- 写回内存的操作会引起其他CPU中缓存了该内存地址的数据无效(MESI)

MESI缓存一致性协议:

多个CPU从主内存中获取同一个数据到各自的高速缓存, 当其中某个CPU修改了缓存中的数据, 该数据会马上同步回主内存, 其它CPU通过总线嗅探机制可以感知到数据的变化, 从而将自己缓存中的数据失效

（二）有序性

常见处理器对写-读操作都是允许重排序的，并且常见的处理器都不允许对存在数据依赖的操作进行重排序（这里所说的数据依赖性仅针对单个处理器中执行的指令序列和单个线程中执行的操作，不同处理器之间和不同线程之间的数据依赖性不被编译器和处理器考虑）。

```java
public class MyTest {

	private static boolean flag = false;

	private static Integer i = null;

	public static void main(String[] args) {
		new Thread(new Runnable() {
			@Override
			public void run() {
			    // 下面两行没有依赖关系，可能发生重排序，即flag设置为true，i还未赋值
				i = new Integer(10);
				flag = true;
			}
		}).start();

		new Thread(new Runnable() {
			@Override
			public void run() {
				if(flag) {
					System.out.print(i); // 可能i还未赋值，为null
				}
			}
		}).start();
	}
}
```
前文一开始就举例的代码中，就有可能i还未被初始化并赋值。所以避免这些问题，JMM提供了内存屏障指令，由开发者来禁止指令做重排序。

内存屏障指令一共有4类
- LoadLoad Barriers：确保Load1数据的装载先于Load2以及所有后续装载指令
- StoreStore Barriers：确保Store1的数据对其他处理器可见（会使缓存行无效，并刷新到内存中）先于Store2及所有后续存储指令的装载
- LoadStore Barriers：确保Load1数据装载先于Store2及所有后续存储指令刷新到内存
- StoreLoad Barriers：确保Store1数据对其他处理器可见（刷新到内存，并且其他处理器的缓存行无效）先于Load2及所有后续装载指令的装载。该指令会使得该屏障之前的所有内存访问指令完成之后，才能执行该屏障之后的内存访问指令。

有兴趣的同学可以读 [Why Memory Barrier？](https://sstompkins.wordpress.com/2011/04/12/why-memory-barrier%EF%BC%9F/)了解更多。

（三）不含原子性

并非对变量使用了volatile修饰符，就说明对这个变量的所有操作都是线程安全的。
对volatile变量使用线程安全的场景：
+ 运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值
+ 变量不需要与其他的状态变量共同参与不变约束

## 解决并发问题(线程安全问题)的方法
+ synchronized
+ volatile（轻量级的同步，只提供可见性、有序性）

如何基于可见性、有序性来做线程安全呢？

##### CAS（Compare And Swap）
Linux内核实现了原子性操作cmpxchg指令。（该部分我没有找到太多的资料，有兴趣的可以自行查找后与我们分享）

结合java提供的volatile变量，那么就满足了线程安全的三个特性：原子性、可见性、有序性（当然此处的volatile变量只能修饰一个变量）。

我们jdk中自1.5后提供的j.u.c包种的很多线程安全类，比如说AtomicInteger等都是基于cas来实现的。

##### CAS缺点
+ ABA问题。解决方法：version
+ 可能循环次数太多，浪费cpu时间

## 写在最后
以上只是写了一些关于java并发编程的基础知识，后续将会按照java多线程发展史开讲jdk中对于java的工具的实现。

##### java多线程发展史
+ JDK 1.0
    - synchronized操作符
    - Object 类成员方法：wait()、notify()、notifyall()
    - Thread 类成员方法：sleep()、interrupt()、join()等
+ JDK 1.2
    - 引入了：ThreadLocal类
+ JDK 1.4
    - NIO
+ JDK 5.0 (JSR 133:Happens-before原则、JSR 166：j.u.c包) Doug Lea
    - Executors 框架
    - Callable和Futrue
    - Exchanger
    - 基于AQS的 Semaphore、ReentrantLock、ReentrantReadWriteLock 
    - Condition
    - 原子类Atomic*，线程安全的集合类ConcurrentHashMap等
+ JDK 6.0
    - Synchronized锁优化
    - CyclicBarrier
+ JDK 7.0
    - fork-join框架
+ JDK 8.0
    - LongAdder：相对AtomicLong性能更好
    - CompletableFuture：Future的增强版，多应用于函数式编程，支持流式调用 
    - StampedLock：ReadWriteLock的一个改进