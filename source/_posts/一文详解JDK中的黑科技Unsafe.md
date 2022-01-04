---
title: 一文详解JDK中的黑科技Unsafe类
date: 2021-11-3 20:20:19
tags:
  - Unsafe
  - StampedLock
  - DirectByteBuffer
categories: JDK源码
---

### 一文详解JDK中的黑科技Unsafe类

#### 1 Unsafe类概述

​		如果说**Java**语言之中有没有什么黑科技，那么**Unsafe**当之为愧。**java**是一种安全性很高的语言，正常情况程序中不用直接操作内存（申请内存与释放内存），**JVM**会按照垃圾回收机制对无用的内存自动进行回收。**Unsafe**类打破了这一切，其提供了一些用于执行低级别、不安全操作的**native方法**，如直接访问系统内存资源、自主管理内存资源等。这些方法一方提升Java运行效率、增强Java语言操作底层资源能力，另一也给java语言的健壮性、安全性带来了隐患，这应该为什么他被命名为**Unsafe**的原因吧。

​		**JDK9**引入模块化机制后，**Unsafe**类的功能同时存在于**java.base模块的jdk.internal.misc.Unsafe** 与**jdk.unsupported模块的sun.misc.Unsafe**中。两个**Unsafe**类之间有细微的差，**jdk.unsupported模块的sun.misc.Unsafe**主要是为了向后兼容老版本的JDK，其与**JDK9**之前版本的**sun.misc.Unsafe**基本是一样的。而**java.base模块的jdk.internal.misc.Unsafe**，其被分配到**jdk.internal**包中表明该**Unsafe**类主要用于**JDK**本身核心类的实现。**JDK9**开始**JDK**原来依赖的**Unsafe**类已全转由**sun.misc.Unsafe**转为**jdk.internal.misc.Unsafe**。同时在**JDK9**中之前版本**Unsafe**的一些功能已被**VarHandle**类替代（**VarHandle**的功能是在[JEP193](http://openjdk.java.net/jeps/193)中提出的），随着时间的推移**java.base模块的jdk.internal.misc.Unsafe** 类与**jdk.unsupported模块的sun.misc.Unsafe**类终将退出历史的舞台(目前最新的**JDK17**中还存在)，取而代之的将是**java.lang.invoke.VarHandle**类。

​		在**JDK11**中即可以使用**java.base模块的jdk.internal.misc.Unsafe** 类也可以使用**jdk.unsupported模块的sun.misc.Unsafe**类，但最好别去使用**Unsafe**这个黑科技。**JDK11**中使用**java.base模块的jdk.internal.misc.Unsafe** 类时，编译与运行时要加入对应的参数，否则将会出现一些异常。

```java
import jdk.internal.misc.Unsafe;
public class App {
  private Integer i;
  public static void main(String[] args) throws Exception{
    Unsafe unsafe = Unsafe.getUnsafe();
    App app = new App();
    System.out.println(app.i);
    long offset = unsafe.objectFieldOffset(App.class.getDeclaredField("i"));
    unsafe.putObject(app, offset, 613);
    System.out.println(app.i);
  }
}
```

以在Idea下开发为例如果未设置如下编译参数（假设项目未指定**module-info.java**文件，为未命名模块），

> **--add-exports=java.base/jdk.internal.misc=ALL-UNNAMED**

Idea将提示

> **Package 'jdk.internal.misc' is declared in module 'java.base', which does not export it to the unnamed module**

同样运行程序时，如果未设置如下**VM**参数

> **--add-opens java.base/jdk.internal.misc=ALL-UNNAMED --illegal-access=warn**

控制台将抛出如下异常

> **Exception in thread "main" java.lang.IllegalAccessError: class org.example.App (in module jdk11.base) cannot access class jdk.internal.misc.Unsafe (in module java.base) because module java.base does not export jdk.internal.misc to module UNNAMED ...**

当然可以像**JDK9**之前那样去使用**sun.misc.Unsafe**类

```java
import java.lang.reflect.Field;
import sun.misc.Unsafe;
public class App {
  private Integer i;
  public static void main(String[] args) throws Exception{
    Field f = Unsafe.class.getDeclaredField("theUnsafe");
    f.setAccessible(true);
    Unsafe unsafe = (Unsafe) f.get(null);
    App app = new App();
    System.out.println(app.i);
    long offset = unsafe.objectFieldOffset(App.class.getDeclaredField("i"));
    unsafe.putObject(app, offset, 613);
    System.out.println(app.i);
  }
}
```

#### 2 Unsafe类能做什么？

​		前面说过**Unsafe**类能做一些低级别的操作，如直接访问系统内存资源、自主管理内存资源等。那**Unsafe**类还能做什么？

![](https://gitee.com/0909/blog/raw/master/img/20211103172118.png)

大致总结一下可以分为上面八大类**内存屏障、内存操作、线程调度、系统操作、Class操作、对象操作、CAS操作、数组操作**。

**注意：JDK8之后的版本中Unsafe类与之前版本中的Unsafe类有差异，下面我们基于JDK11来看一下jdk.internal.misc.Unsafe中各功能对应的方法。**

#### 3 内存屏障

##### 3.1 内存屏障简介

​		为了提升计算机处理效率，编译器会对编译后的指令进行重排，同时CPU会乱序执行指令。	**内存屏障（Memory Barriers）**技术可以在适当的位置禁止这种重排与乱序执行。**内存屏障**实际是一些特殊的指令，该指令使得在其之前的所有Load操作与Store操作都先于其后的Load操作与Store操作执行。

​		**内存屏障（Memory Barriers）**可分为**读屏障（Load Barriers）**和**写屏障（Store Barriers）**，这两者进行结合共有四种屏障**Load-Load Barriers、Load-Store Barriers、Store-Store Barriers、Store-Load Barriers**。下面看一下不同内存屏障的插入会达到什么要的效果。

```
load1 LoadLoad load2
```

> 保证 load1 数据的装载优先于 load2 以及所有后续装载指令的装载。对于 Load Barrier 来说，在指令前插入 Load Barrier，可以让高速缓存中的数据失效，强制重新从主内存加载数据。

```
load1 LoadStore store2
```

> 保证 load1 数据装载优先于 store2 以及后续的存储指令刷新到内存。

```
store1 StoreStore store2
```

> 保证 store1 数据对其他处理器可见，优先于 store2 以及所有后续存储指令的存储。对于 Store Barrier 来说，在指令后插入 Store Barrier，能让写入缓存中的最新数据更新写入主内存，让其他线程可见。

```
store1 StoreLoad load2
```

> 在load2 及后续所有读取操作执行前，保证 store1 的写入对所有处理器可见。这条内存屏障指令是一个全能型的屏障，它同时具有其他 3 条屏障的效果，而且它的开销也是四种屏障中最大的一个。

##### 3.2 内存屏障相关方法

**JDK11**中**jdk.internal.misc.Unsafe**类提供了如下方法插入对应的内存屏障。

```java
//读内存屏障，确保屏障前的load操作不会与其后的load操作与store操作重排序。
//相当于LoadLoad 与 LoadStore 屏障。其效果是强制读取操作从主内存中获取最新值。
@HotSpotIntrinsicCandidate
public native void loadFence();
//写内存屏障，确保屏障前的store操作、load操作不会与其后的store操作重排序。
//相当于StoreStore 与 LoadStore 屏障
@HotSpotIntrinsicCandidate
public native void storeFence();
//全内存屏障，确保屏障前的store操作、load操作不会与其后的store操作、load操作重排序。
//相当于StoreLoad 屏障
@HotSpotIntrinsicCandidate
public native void fullFence();
public final void loadLoadFence() {
  loadFence();
}
public final void storeStoreFence() {
  storeFence();
}
```

上面方法上**@HotSpotIntrinsicCandidate**注解表示HotSpot虚似机实现时，为提升方法运行性能可能根据具体平台对该方法进行内置的优化。作为Java应用程序开发者可以忽略这个注解。

##### 3.3 应用场景

​		**JDK8**中引入了一种新类型的锁**StampedLock**, 该锁是对**ReentrantReadWriteLock**进行一步进行了优化，其之前读写锁的基础上，引入了乐观读锁的概念，这种乐观读锁类似于无锁的操作，完全不会阻塞写线程获取写锁，从而缓解读多写少时写线程“饥饿”现象，在多读几乎没写的场景下效率低高。**但其不支持重入与条件等待**。由于**StampedLock**提供的乐观读锁不阻塞写线程获取写锁，当线程共享变量从主内存load到线程工作内存时，会存在数据不一致问题，所以当使用**StampedLock**的乐观读锁时需要遵从一定的模式来确保数据的一致性。在**StampedLock**的源码中，**Doug Lea** 为我们提供了使用**StampedLock**乐观锁的示例代码，这里稍微简化一如下。

```java
class Point {
  private double x, y;
  private final StampedLock sl = new StampedLock();
  // an exclusively locked method
  void move(double deltaX, double deltaY) {
    long stamp = sl.writeLock();
    try {
      x += deltaX;
      y += deltaY;
    } finally {
      sl.unlockWrite(stamp);
    }
  }
  // a read-only method, upgrade from optimistic read to read lock
  double distanceFromOrigin() {
    long stamp = sl.tryOptimisticRead(); // (1) 获取乐观读锁
    double currentX = x; double currentY = y;// (2) 复制主内存数据到工作内存
    if (!sl.validate(stamp)) { // (3) 校验乐观读后数据是否发生变化
      try {
        stamp = sl.readLock();//(4) 升级乐观读锁为悲观读锁
        currentX = x; //(5) 复制主内存数据到工作内存
        currentY = y;
      } finally {
        sl.unlockRead(stamp);//(6) 释放悲观读锁
      }
    }
    return Math.sqrt(currentX * currentX + currentY * currentY);//(7) 线程工作内存计算
  }
}
```

**StampedLock**中写锁还是和**ReentrantReadWriteLock**写锁一致，但读锁不一样。**distanceFromOrigin**计算点x, y到原点的距离时，会先尝试调用**StampedLock#tryOptimisticRead**方法，获取乐观读锁，再将主内存的数据复制到工作内存。但此时其他线程可能调用**move**方法修改了x, y的值，因此要通过**StampedLock#validate**方法校验主内存的数据是否发生变化，如果有发生变化则将乐观读锁升级为悲观读，重新将主内存数据复制到工作内存，否则直接返回计算结果。

<img src="https://gitee.com/0909/blog/raw/master/img/20211112153303.png" style="zoom:80%;" />

下面是**JDK11中StampedLock#validate**方法的源码，在执行校验逻辑之前会通过Unsafe的**loadFence**方法加入一个**load内存屏障**，目的是避免上面的**Point示例**中步骤(2)和**StampedLock#validate**中校验逻辑发生重排序导致校验不准确的问题。

```java 
//StampedLock#validate方法
public boolean validate(long stamp) {
   VarHandle.acquireFence();
   return (stamp & SBITS) == (state & SBITS);
}
//VarHandle#acquireFence方法
public static void acquireFence() {
  UNSAFE.loadFence();
}
```

#### 4 内存操作

​		**Java**语言中不像c、c++那样要工程师在代码中维护内存的申请与释放，**JVM**中的垃圾回收机制会自动回收内存，但**Unsafe**提供了应用代码层面管理内存的机制，这部分内存是非JVM管理的内存，通常叫作**堆外内存**。

##### 4.1 内存管理相关方法

**JDK11**中**jdk.internal.misc.Unsafe**类提供了如下内存管理的方法

```java
//分配指定字节的内存
public long allocateMemory(long bytes);
//指定地址扩充内存
public native long reallocateMemory(long address, long bytes);
//释放内存
public native void freeMemory(long address);
//在给定的内存块中设置值
public native void setMemory(Object o, long offset, long bytes, byte value);
//内存拷贝
public void copyMemory(Object srcBase, long srcOffset, Object destBase,
long destOffset, long bytes)
//根据偏移地址获取对应对象变量值，变量本身的访问修饰符限制将忽略，类似操作还有getInt、getLong、getChar等
public native Object getObject(Object o, long offset);
//根据偏移地址设置对象变量值，变量本身的访问修饰符限制将忽略，类似操作还有putInt、putLong、putChar等
public native void putObject(Object o, long offset, Object x);
//获取给定地址的byte类型的值（参数address值要为allocateMemory分配，否则结果正确性无法保证）
public native byte getByte(long address);
//为给定地址设置byte类型的值（参数address值要为allocateMemory分配，否则结果正确性无法保证）
public native void putByte(long address, byte x);
```

##### 4.2 为什么需要堆外内存

- **改善垃圾回收停顿时长**

  由于堆外内存是直接受操作系统管理而不是JVM，所以当我们使用堆外内存时，即可保持较小的堆内内存规模。从而在GC时减少回收停顿对于应用的影响。

- **提升程序I/O操作的性能**

  通常在I/O通信过程中，会存在堆内内存到堆外内存的数据拷贝操作，对于需要频繁进行内存间数据拷贝且生命周期较短的暂存数据，都建议存储到堆外内存。

##### 4.3 堆外内存运用之DirectByteBuffer

​		JDK中**ByteBuffer#allocateDirect**方法能直接分配堆外内存，其实际返回的是**DirectByteBuffer**对象，而**DirectByteBuffer**类内部又通过**Unsafe**的**allocateMemory**、**setMemory**与**freeMemory**等方法管理内存。**NIO**框架**Netty**中大量使用堆外内存，提升I/O操作的性能。

```
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```

​		下面是**DirectByteBuffer**构造函数，创建**DirectByteBuffer**的时候，通过**Unsafe#allocateMemory**分配内存、**Unsafe#setMemory**进行内存初始化，而后通过**Cleaner**跟踪**DirectByteBuffer**对象的垃圾回收，以实现当**DirectByteBuffer**被垃圾回收时，通过**Deallocator**调用**Unsafe#freeMemory**回收之前分配的堆外内存。

```java
DirectByteBuffer(int cap) {// package-private
  super(-1, 0, cap, cap);
  boolean pa = VM.isDirectMemoryPageAligned();
  int ps = Bits.pageSize();
  long size = Math.max(1L, (long)cap + (pa ? ps : 0));
  Bits.reserveMemory(size, cap);
  long base = 0;
  try {
    base = UNSAFE.allocateMemory(size);
  } catch (OutOfMemoryError x) {
    Bits.unreserveMemory(size, cap);
    throw x;
  }
  UNSAFE.setMemory(base, size, (byte) 0);
  if (pa && (base % ps != 0)) {
    // Round up to page boundary
    address = base + ps - (base & (ps - 1));
  } else {
    address = base;
  }
  cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
  att = null;
}
```

**Cleaner**跟踪**DirectByteBuffer**对象的垃圾回收，最终通过**Deallocator**调用**Unsafe#freeMemory**回收**DirectByteBuffer**上分配的堆外内存的整体流程如下图。当**DirectByteBuffer**对象不可达时，垃圾收集器会将**DirectByteBuffer Reference**加入的**pending链表**中，而守护线程**ReferenceHandler**，会不断地从**pending链表**中获取节点处理，如果节点的**Reference**类型为**PhantomReference**引用的子类**Cleaner**，守护线程在处理时会调用该**Refence**绑定的**Cleaner#clean方法**，即**DirectByteBuffer**绑定的**Cleaner**，这个**Cleaner**会通过**Deallocator#run**方法调用**Unsafe#freeMemory**，最终实现**DirectByteBuffer**上分配的堆外内存被回收。

<img src="https://gitee.com/0909/blog/raw/master/img/20211112151451.png" style="zoom:80%;" />

具体的源码分析可以参考我之前的文章[Java Reference核心原理分析](https://mp.weixin.qq.com/s/8f29ZfGvZVPe0bO-FahokQ)。

#### 5 线程调度

​		**JDK**内部有一个很重要的类**LockSupport** ，Java锁和同步器框架的核心类**AbstractQueuedSynchronizer**中线程的挂起与恢复全依赖于**LockSupport#park方法与LockSupport#unpark方法**，而**LockSupport**类的**park**方法、**unpark**方法分别依赖于**Unsafe**类的**park**方法、**unpark**方法。

**JDK11**中**LockSupport**类与**Unsafe**类中线程调用相关方法如下。

```java
//LockSupport类park与unpark方法
public static void park(Object blocker) {
  Thread t = Thread.currentThread();
  setBlocker(t, blocker);
  U.park(false, 0L);
  setBlocker(t, null);
}
public static void unpark(Thread thread) {
  if (thread != null)
    U.unpark(thread);
}
//Unsafe类park与unpark方法
public native void unpark(Object thread);
public native void park(boolean isAbsolute, long time);
```

**注意：JDK8的Unsafe中还有monitorEnter、monitorExit、tryMonitorEnter方法，但在JDK11的jdk.internal.misc.Unsafe类中已将这些方法移除。**

#### 6 CAS操作

​		**CAS操作**是**Compare And Swap**的简写，其核心思想是先比较再交互，有点类似是乐观锁，通过version比较再进行更新，如果version值发生变化则更新将失败。其将预期原值与当前原值比较，如果相同则表示未发生改变，则用新值去更新原值，否则代表原值已改变**CAS操作**将失败。**Unsafe**底层实现**CAS操作**依赖于CPU的原子指令（**cmpxchg指令**)。

**JDK11**中**jdk.internal.misc.Unsafe**类提供了如下**CAS操作**方法，**JDK8**以后**Swap** 这个单词被换成了**Exchange**，之前版本都是***compareAndSwap******这类的方法。

```java
// 同时针对Long、Byte、Boolean等提供的类似的compareAndExchange***方法
public final native int compareAndExchangeInt(Object o, long offset, int expected, int x);
// 同时针对Long、Byte、Boolean等提供的类似的compareAndSet***方法
public final native boolean compareAndSetInt(Object o, long offset, int expected, int x);
public final native Object compareAndExchangeObject(Object o, long offset, 
   Object expected, Object x);
public final native boolean compareAndSetObject(Object o, long offset,Object expected, 
   Object x);
```

​		**CAS操作**在**AtomicInteger**类中大量使用，如下源码中**VALUE**代表**AtomicInteger**中存储的实际value值相对的内存地址偏移量，有了这偏移量后，便可调用**Unsafe**类的**CAS**操作。

```java
public class AtomicInteger extends Number implements java.io.Serializable {
  private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
  private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");
  private volatile int value;
}
```

#### 7 对象操作

​		对象操作主要是先通过**Field**或是**Class**与变量名获取对象成员属性相对于对象本身的内存地址的偏移量，然后再利用这个内存偏移量做一些其他的操作比如设置变量值、通过**CAS操作**设置值、采用**volatile**的存储语义写变量的值、采用**volatile**的加载语义读取变量的值。还一个**allocateInstance**方法，可以该方法可发绕过构造方法创建对象，只要传入对应的class就行，并且其不需要初始化代码、JVM安全检查等。

**JDK11**中**jdk.internal.misc.Unsafe**类提供了如下对象操作方法：

```java 
//通过Field获取对象成员属性相对于对象本身的内存地址的偏移量
public native long objectFieldOffset(Field f);
//通过Class与变量名，获取对象成员属性相对于对象本身的内存地址的偏移量
public long objectFieldOffset(Class<?> c, String name);
//获得给定对象的指定地址偏移量的值，还有基于int、long、char等类似的操作
public native Object getObject(Object o, long offset);
//给定对象的指定地址偏移量设值，还有基于int、long、char等类似的操作
public native void putObject(Object o, long offset, Object x);
//从对象的指定偏移量处获取变量的引用，采用volatile的加载语义，还有基于int、long、char等类似的操作
public native Object getObjectVolatile(Object o, long offset);
//存储变量的引用到对象的指定的偏移量处，采用volatile的存储语义，还有基于int、long、char等类似的操作
public native void putObjectVolatile(Object o, long offset, Object x);
//有序、延迟版本的putObjectVolatile方法，不保证值的改变被其他线程立即看到。
// 只有在field被volatile修饰符修饰时有效，还有基于int、long、char等类似的操作
public native void putOrderedObject(Object o, long offset, Object x);
//绕过构造方法、初始化代码来创建对象
public native Object allocateInstance(Class<?> cls) throws InstantiationException;
```

#### 8 数组操作

​		数组相关的操作主要有**arrayBaseOffset**与**arrayIndexScale**这两个方法，**arrayBaseOffset**用于获取数组首个元素相对的内存地址偏移量，而**arrayIndexScale**用于获取数组中一个元素占用内存的大小，两者配合起来使用，便可定位数组中每个元素在内存地址中的位置。然后结合上面的对象操作方法，设置获取、更改数组元素的值。

**JDK11**中**jdk.internal.misc.Unsafe**类提供了如下数组操作方法：

```java
//获取数组首个元素相对的内存地址偏移量
public native int arrayBaseOffset(Class<?> arrayClass);
//取数组中一个元素占用内存的大小
public native int arrayIndexScale(Class<?> arrayClass);
```

​		**JDK8**中**AtomicIntegerArray、AtomicLongArray**便结合这个方法来操作其内存存储数据的数组。**JDK9**后已改用**VarHandler**内部的方法。

**JDK8**中的**AtomicIntegerArray**部分源码

```java 
public class AtomicIntegerArray implements java.io.Serializable {
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    //获取数组首个元素相对的内存地址偏移量
    private static final int base = unsafe.arrayBaseOffset(int[].class);
    private static final int shift;
    private final int[] array;
    static {
      	//取数组中一个元素占用内存的大小
        int scale = unsafe.arrayIndexScale(int[].class);
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }
  	//通过下标计算index为i的元素，相对的内存地址偏移量
  	private static long byteOffset(int i) {
        return ((long) i << shift) + base;
    }
  	private long checkedByteOffset(int i) {
        if (i < 0 || i >= array.length)
            throw new IndexOutOfBoundsException("index " + i);
        return byteOffset(i);
    }
  	//通过上面7小节中的对象操作，在指定内存地址中采用volatile的存储语义写入值
  	public final void set(int i, int newValue) {
        unsafe.putIntVolatile(array, checkedByteOffset(i), newValue);
    }
  	
}
```

**Unsafe**类中有**Class**与系统的操作就不一一看了，感兴趣的同学可以自己看一下**Unsafe**的源码。

#### 总结

​		本文介绍了**JDK**中的黑科技**Unsafe**类，按功能进行了归类划分，并介绍了这些方法的使用场景，部分功能还结合了源码进行了简单地分析。还是那句话**Unsafe**类提供了一些用于执行低级别、不安全操作的**native方法**，如直接访问系统内存资源、自主管理内存资源等，不会用千万别乱用（它是**unsafe**的），但作为黑科技不了解怎么算得算是Java工程师呢。

#### 结尾

​		原创不易，点赞、在看、转发是对我莫大的鼓励，关注公众号洞悉源码是对我最大的支持。同时相信我会分享更多干货，我同你一起成长，我同你一起进步。
