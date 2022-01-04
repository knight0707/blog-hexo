---
title: 深入浅出Java内存模型
date: 2021-11-3 20:20:19
tags:
  - volatile
  - synchronized
  - JMM
  - double-check
categories: java
---

###  深入浅出Java内存模型

#### Java 内存模型概述

​		**Java 内存模型**（**Java Memory Model，JMM**）其主要的目标是定义程序中各个变量的访问规则，即虚似机中将变量储存到内存与从内存中取出变量这样的底层细节。**JMM** 是一个抽象的概念，它描述了一系列的规则或者规范，用来解决多线程的共享变量访问问题。这里所说的变量，包括实例字段、静态字段，但**不包括局部变量和方法参数，因为后者是线程私有的，不存在数据竞争问题，即没有线程安全问题**。

​		从**JMM**本身角度理解，可以分为二方面，一方面其抽象了**主内存（Main Memory ）与工作内存（Work Memory）**的概念，另一方面其定义了**一组保障内存可见性与正确性的规范**。**JMM**抽象出了**主内存**与**工作内存**的概念，**主内存**中存储了线程之间的共享变量，**工作内存**是通过对缓存、寄存器、写缓冲区等的抽象，每个线程都有一个私有的**工作内存**，**工作内存**中存储了**主内存**中共享变量对应的副本。线程在执行时先从**主内存**加载数据**工作内存**，然后执行逻辑后再将数据刷新回**主内存**，使该数据对其他线程可见。如果没有一定的规范与约束，这种模型明显存在数据一致性的问题（由于Java编译器指令重排序优化和CPU乱序执行优化的存在，使得问题变得更加复杂）。所以**JMM**基于内存屏障（**Memory-Barrier**）提供了类似**as if serial**与**happens-before**的保障，最终保障了多线程之间操作的可见性与正确性。

​		而从**JMM**使用者的角度理解，**JMM**平衡了**JVM**工程师、编译器工程师在性能上的需求与**Java**程序员开发安全的简单性上的渴望。**JMM在保证正确性的同时会最大限度的放宽对指令重排和乱序执行的限制**。对于**Java**程序员，**JDK**提供了如**volatile**关键字、**synchronized**关键字、**final**关键字、**Java**并发类这样的顶层机制，为程序员提供简单的编程模型。

#### Java为什么需要内存模型

​		为了提升计算机的运行效率，编译器、内存系统通常会对指令进行重排，处理器会对指令进行乱序/并行执行，处理器与主内存之间存在高速缓存以减少CPU的等待时间。

![](https://gitee.com/0909/blog/raw/master/img/20211109113941.png)

​		处理器优化其实也是重排序的一种类型，重排序可分三大类

- **编译器优化的重排序** ， 编译器在不改变单线程程序语义放入前提下，可以重新安排语句的执行顺序。
- **指令级并行的重排序** ，现代处理器采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。
- **内存系统的重排序**，由于处理器使用缓存和读写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

​		类似上面的这些优化给并发编程带来了**一致性**、**可见性**、**有序性**等问题。这些问题从深层次来看与**缓存一致性**、**处理器优化**、**指令重排序**有关。**JMM**就是为了解决这些问题而提出的，其通过抽象内存模型，制定类似**as if serial**与**happens-before**的规则来屏蔽底层编译器、内存系统、处理器重排的细节，最终为**Java**工程师提供更加清晰、简单的编程模型。

​		编译器/JVM工程师，会根据**Java**语言规范中对**JMM**的规定，去处理不同平台、不同处理器架构上的差异，通过使用类似**内存屏障（Memory-Barrier）**之类技术，保证执行结果符合 **JMM** 的推断。而**Java**工程师，则不用关心这些细节，通过遵守 **happens-before 规则**、**volatile**语义、**synchronized**语义、**final** 语义以及合理地使用**Java**并发包下的相关类，便可写出可靠的多线程应用。虽然单线程下面也会出现上面说的编译器重排，处理器乱序执行等问题，但其都遵守**as-if-serial语义**，即编译器与处理器怎么优化后都能保证单线程下面程序执行结果的可预测性与正确性。

![](https://gitee.com/0909/blog/raw/master/img/20211108133444.png)

​		从上面可以很直观的看到，**JMM**的引入使得**Java**工程师、编译器/**JVM**工程师、处理器工程师之间的关注点进行有效的分层隔离，不同架构的CPU对最上层的应用程序的**Java**工程师而言已是透明的，**Java**工程师将更聚焦于业务逻辑的开发。

#### 主内存与工作内存

​		前面说到JMM抽象了**主内存**与**工作内存**，**主内存**中存储了线程间的共享变量，通过对缓存、寄存器、写缓冲区等的抽象每个线程都有一个私有的**工作内存**，**工作内存**中存储了主内存对应的副本。线程对变量的读写操作都发生在其自身的**工作内存**中，而不能直接操作**主内存**中的变量。线程不能直接读写其他线程私有**工作内存**中的数据，线程间的变量传必须通过**主内存**完成。**JMM**规定了线程应该何时从**主内存**中加载数据到**工作内存**，何时又该将**工作内存**中的副本数据刷新到**主内存**，使数据对其他线程可见。这种可见性机制在**JMM**中是通过**happens-before规则**定义的，而具体**JVM**实现上则是通类似**内存屏障（Memory-Barrier）**之类的技术。

![](https://gitee.com/0909/blog/raw/master/img/20211108172615.png)

​		如上图，如果线程A在**工作内存**中修改变量X要对线程B可见的话，其首先要将变量X刷新回**主内存**，当变量X被刷新回**主内存**时，线程B立刻能感知到（底层实现上是通过**总线嗅探**）其**工作内存**中的变量X对应**主内存**的值已被修改，于是线程B会将其**工作内存**变量X设置为失效，当线程B后继的操作如果要用户变量X时又会重新从**主内存**加载最新的值。

​		**JMM**抽象出的**主内存**与**工作内存**的概念，很容易让人想起多核**CPU**架构下的硬件内存模型。**JMM**中的主内存可以理解为计算机的主内存，而**JMM**中的工作内存可以理解为**CPU**中集成的高缓存缓存（**L1 Cache**、**L2 Cache**、**Register**等）。

![](https://gitee.com/0909/blog/raw/master/img/20211109140026.png)

​		硬件内存模型中，各核高速缓存、主内存之间存在**缓存一致性**问题，然后通过缓存一致性协议、总线加锁等方式去解决该问题。类似的**JMM**为解决**缓存一致性**问题，提出了**happens-before规则**，同时采用保**内存屏障技术**实现**happens-before规则**。

#### happens-before规则

​		**JMM**中最重要的概念是**happens-before规则**，其描述了两个操作的内存可见性。如果要保证操作X的结果对操作Y可见（如果操作X与操作Y是否在同一个线程中），那么操作 X要 **happens-before** 操作 Y。如果操作X与操作Y之间不存在 **happens-before** 关系，则JVM可以对操作X与操作Y任意重排。

​		当一个共享变量被多个线程读写时，如果在读操作与写操作之间没有依照**happens-before规则**，那就会发生**数据竞争**（发生**数据竞争**时程序的执行结果将不可预测）。**JMM**中有如下的**happens-before规则**：

- **程序顺序规则**：一个线程中的每个操作，**happens-before**于该线程中的任意后续操作。
- **volatile变量规则**：对一个**volatile**变量的写，**happens-before**于任意后续对这个**volatile**变量的读。
- **监视器锁定规则**：对对象的**unLock**操作**happens-before**于后面对同一个对象的**lock**操作。
- **线程中断规则**：对线程 **interrupt()** 的调用**happens-before**于线程代码中检测到中断事件的发生，可以通过 **Thread.interrupted()** 方法检测是否发生中断。
- **线程启动规则**：对线程 **start()** 的操作**happens-before**于线程内的任何操作。
- **对象终结规则**：一个对象的初始化完成**happens-before**于它的 **finalize()** 方法的开始。
- **传递规则**：如果操作 A **happens-before**于操作 B，而操作 B 又**happens-before**于操作 C，则可以得出操作 A **happens-before**于操作 C。

​		来看一下平时开发中经常用到的**volatile关键字**与**synchronized关键字**，在**happens-before**规则下的效果。**volatile关键字遵守volatile变量规则**，**其效果是一个线程对taskRunningFlag类型变量的写操作的结果，立刻对其他任意线程对同一个变量的读操作可见**。

```Java
public class App {
  private volatile boolean taskRunningFlag = true;
  public static void main(String[] args) {
    new App().runTask();
  }
  public void runTask() {
    Runnable task = () -> {
      long startTime = System.currentTimeMillis();
      while (taskRunningFlag) {
        System.out.println("Task is running !");
        try {
          Thread.sleep(100);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
      System.out.println("Task run " + (System.currentTimeMillis() - startTime) + "ms !");
    };
    new Thread(task).start();
    Runnable taskController = () -> {
      try {
        Thread.sleep(500);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      System.out.println("After 500 ms stop task !");
      taskRunningFlag = false;
    };
    new Thread(taskController).start();
  }
}
```

> Task is running !
> Task is running !
> Task is running !
> Task is running !
> Task is running !
> After 500 ms stop task !
> Task run 520ms !

可以看到任务线程一直在进行，直到500ms后任务控制线程将taskRunningFlag设置为false，由于任务控制线程与任务线程在并行运行，任务最终运行时间会略微超过500ms，这里任务运行了520ms，但volatile类型的taskRunningFlag的写操作是对其他线程立即可见的。**注意**：**volatile关键字可以保障单条指令操作的可见性，但不保障多个指令操作的可见性即不保障操作原子性，x++，++x，--x，x--这类操作编译后都对应多个指令所以使用volatile关键字也不能保证操作的可见性，要用AtomicInteger，synchronized关键字等替代。**

​		**synchronized关键字遵守监视器锁定规则，其效果是一个已获得锁的代码块或方法在释放锁之前，其内部的所有操作的结果，对于即将获得到同一把锁的其他线程可见。**

<img src="https://gitee.com/0909/blog/raw/master/img/20211110094418.png" style="zoom:80%;" />

​	

​		**视器锁定规则的可见性，实际将原子性涵盖进来了，因为其保证的是unlock Monitor 之前的所有操作对于lock Monitor 之后的操作都可见**。如上图线程X中对变量y与变量z的赋值操作，对于后继锁定同一个Monitor的线程Y都是可见的。**注意：线程X与线程Y一定要锁同一个对象的Monitor，才能保障这种可见性规则。别外，JMM在保证正确性的同时会最大限度的放宽对指令重排和乱序执行的限制了。这意味着线程X同步代码块内 y= 7与z = 3是可以进行重排的，只是整个同步代码块内部的操作不会与同步代码块外部的操作重排，理解这点非常重要。**

#### 内存屏障

​		**happens-before规则属于JMM保证可见性的规范，而非具体实现**。编译器工程师、JVM工程师在实现编译器与**JVM**时，会根据这个规范在适当的位置利用**内存屏障（Memory Barrier）**技术禁止指令重排保证可见性。**内存屏障**可分为**读屏障（Load Barriers）**和**写屏障（Store Barriers）**，**JMM** 中对这两者进行结合共有四种屏障**Load-Load Barriers、Load-Store Barriers、Store-Store Barriers、Store-Load Barriers**。下面看一下不同内存屏障的插入会达到什么要的效果。

```c
load1 LoadLoad load2
```

> 保证 load1 数据的装载优先于 load2 以及所有后续装载指令的装载。对于 Load Barrier 来说，在指令前插入 Load Barrier，可以让高速缓存中的数据失效，强制重新从主内存加载数据。
>

```c
load1 LoadStore store2
```

> 保证 load1 数据装载优先于 store2 以及后续的存储指令刷新到内存。

```c
store1 StoreStore store2
```

> 保证 store1 数据对其他处理器可见，优先于 store2 以及所有后续存储指令的存储。对于 Store Barrier 来说，在指令后插入 Store Barrier，能让写入缓存中的最新数据更新写入主内存，让其他线程可见。
>

```c
store1 StoreLoad load2
```

>
> 在load2 及后续所有读取操作执行前，保证 store1 的写入对所有处理器可见。这条内存屏障指令是一个全能型的屏障，它同时具有其他 3 条屏障的效果，而且它的开销也是四种屏障中最大的一个。
>

#### Double-Check Lock 问题

​		单例模式中经常会通过 **Double-Check Lock** 实现实例的惰性加载。单不正确的使用 **Double-Check Lock** 却会带来线程安全问题。

```java
public class SingletonInstance {
  private static SingletonInstance instance;
  public static SingletonInstance getInstance() {
    if (instance == null) {
      synchronized (SingletonInstance.class) {
        if (instance == null) {
          instance = new SingletonInstance();
        }
      }
    }
    return instance;
  }
}
```

​		上面的代码，看似能通过**synchronized关键**字保证了顺序性，达到获取正确的SingletonInstance实例，实际却有可能获取到未完全构建的SingletonInstance实例。原因在于 **instance = new SingletonInstance();**这行代码实际可以分为三条指令:

> memory = allocate(); // 1：分配对象的内存空间 
>
> ctorInstance(memory); // 2：初始化对象 
>
> instance = memory; // 3：设置instance指向刚分配的内存地址 

前面强调过**JMM在保证正确性的同时会最大限度的放宽对指令重排和乱序执行的限制了，这意味着同步代码块/方法内可以进行重排的，只是整个同步代码块/方法内部的操作不会与同步代码块/方法外部的操作重排**。所以在编译器、CPU重排优化后，最后的同步块内的指令执行顺序可能如下：

> memory = allocate(); // 1：分配对象的内存空间 
>
> instance = memory; // 3：设置instance指向刚分配的内存地址 
>
> ctorInstance(memory); // 2：初始化对象 

现在再来看看线程A与线程B同时调用**SingletonInstance.getInstance()**方法可能出现的结果。

![](https://gitee.com/0909/blog/raw/master/img/20211110151913.png)

如上图紫色为线程A的操作，橙色为线程B的操作。当instance初始为null时，线程A与线程B交替执行后会出现上面的先后执行顺序。最终导致线程B获取到了线程A未完成构建的对象。

![](https://gitee.com/0909/blog/raw/master/img/20211110151722.png)

如何处理该问题呢？可以用**volatile关键字**修饰instance变量，达到禁示分配对象的内存空间、初始化对象、设置instance指向内存地址这三步重排的。正确**Double-Check Lock** 实现的单例模式如下：

```java 
public class SingletonInstance {
  private static volatile SingletonInstance instance;
  public static SingletonInstance getInstance() {
    if (instance == null) {
      synchronized (SingletonInstance.class) {
        if (instance == null) {
          instance = new SingletonInstance();
        }
      }
    }
    return instance;
  }
}
```

#### 结尾

​		翻译不易，点赞、在看、转发是对我莫大的鼓励，关注公众号洞悉源码是对我最大的支持。同时相信我会分享更多干货，我同你一起成长，我同你一起进步。

#### 参考

[Synchronization and the Java Memory Model](http://gee.cs.oswego.edu/dl/cpj/jmm.html)

[JSR 133 (Java Memory Model) FAQ](https://www.cs.umd.edu/~pugh/Java/memoryModel/jsr-133-faq.html)

[The "Double-Checked Locking is Broken" Declaration](http://www.cs.umd.edu/~pugh/Java/memoryModel/DoubleCheckedLocking.html)

《Java Concurrency in Practice》

[深入理解Java内存模型](https://www.infoq.cn/minibook/Java_memory_model)

《深入理解Java虚似机》
