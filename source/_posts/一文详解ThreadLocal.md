---
title: 一文详解ThreadLocal内存泄漏问题原因
date: 2021-10-25 20:20:19
tags:
  - JDK源码
  - ThreadLocal
categories: JDK源码
---

### 一文详解JDK中的ThreadLocal

#### 1 ThreadLocal概述

​		**ThreadLocal** 提供了**一种变量与线程绑定的机制**，通常把这种机制称为**线程本地变量**，在线程调用栈（方法调用链）的入口或中间可以让一些重要的变量与线程绑定，在后继的调用栈（方法调用链）可使用该变量。这个特性使得**ThreadLocal适用于方法调用链上参数透传**，如APM、日志、权限框架中透传上游重要的参数到下游。

​		**ThreadLocal不支持不同线程间变量的透传**，即如果在线程A中设置了一个变量ThreadLocalA 其中存储的值为AA，在线程B中想拿到ThreadLocalA 中存储的AA是拿不到的（结果将是null）。这也导致了使用线程池的线程，不能通过**ThreadLocal**将**线程本地变量**传递到线程池中供其使用（线程池中执行任务的线程通常不是提交任务对应的线程），同样类似线程间的异步操作**ThreadLocal**也不支持。线程间**线程本地变量**的透传可以通过阿里开源的**TrasmittableThreadLocal**来实现。当子线程要获取父线程中的**线程本地变量**，通过ThreadLocal同样无法获取，但可能通过其子类**InheritableThreadLocal**实现。

​		在高并发场景下，JDK自带的**ThreadLocal**性能不如**Netty**框架中实现的**FastThreadLocal**，**FastThreadLocal** 获取其中保证的变量值时，使用内部的index变量便可定位对应变量值，而不用像ThreadLocal那样通过**开放地址法**去定位对应变量值。

​		**ThreadLocal** 的线程本地变量机制实际是通过 **Thread**类中的**ThreadLocalMap**成员变量实现的。每一个**Thread**实例都维护了一个**ThreadLocalMap**实例。**ThreadLocalMap**通过**开放地址法**实现，其**Key**为**ThreadLocal**，**Value**为**ThreadLocal**中保存的变量。其**开放地址法**底层对应的数组为**Entry数组**，**Entry**类持有了**ThreadLocal**的**WeakReference**，持有了**ThreadLocal**中保存变量的强引用，这导致了不恰当的使用ThreadLocal容易引发内存泄漏问题。

#### 2 源码分析

**注意：下面的ThreadLocal的源码分析基本JDK8**

##### 2.1 引用类型

```java
public class ThreadLocal<T> {
 	...
}
```

​		**ThreadLocal**类本身比较简单其支持泛型，一共没有几行代码，其set与get方法上复杂的处理最终都到了内部类类**ThreadLocalMap**上，并且**ThreadLocalMap**使用**WeakReference**所涉及的东西比较多。**Reference**的源码分析参考之前的文章[Java Reference核心原理分析](https://mp.weixin.qq.com/s/8f29ZfGvZVPe0bO-FahokQ)。

​			这里先简单介绍一下java语言中的引用类型。java语言中引用类型分为**强引用、软引用（SoftReference）、弱引用（WeakReference）、虚引用（PhantomReference）**。

- **强引用**

  通常代码中看到的变量引用关系如下面的 threadLocalData，variable对对象的引用都是强引用

  ```java 
  ThreadLocal<Integer> threadLocalData = ThreadLocal.withInitial(() -> -1);
  String variable = "123";
  ```

- **软引用 （SoftReference）**

  垃圾回收器会根据内存需求酌情回收软引用指向的对象。普通的GC并不会回收软引用，只有在即将**OOM**的时候(也就是最后一次**Full GC**)如果被引用的对象只有**SoftReference**指向的引用，才会回收。如下**SoftValueReference**便持有其值V的软引用。

  ```java 
  static class SoftValueReference<K, V> extends
     SoftReference<V> implements ValueReference<K, V> {
     final ReferenceEntry<K, V> entry;
     SoftValueReference(ReferenceQueue<V> queue, V referent,  		       					             ReferenceEntry<K,V> entry) {
        super(referent, queue);
        this.entry = entry;
     }
  }
  ```

- **弱引用（WeakReference）**

  当发生GC时，如果当前对象只有**WeakReference** 类型的引用，则会被GC给回收掉。如下**ThreadLocalMap** map中的**Entry**便持有**ThreadLocal**的软引用。

  ```java 
  static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
      super(k);
      value = v;
    }
  }
  ```

- **虚引用（PhantomReference）**

  他是一种特殊的引用类型，不能通过虚引用获取到其关联的对象，但当GC时如果其引用的对象被回收，这个事件程序可以感知，这样我们可以做相应的处理。其最常用的场景是GC回收**DirectByteBuffer**对象时，利用**Cleaner**调用Unsafe类回收其对应的堆外内存。具体源码分析可参考[Java Reference核心原理分析](https://mp.weixin.qq.com/s/8f29ZfGvZVPe0bO-FahokQ)。

  ```java 
  public class Cleaner extends PhantomReference<Object> { 
  		...
  }
  ```

​		前面简单地介绍了对象的引用类型，GC决定一个对象是否能被回收与当对象具有的引用类型有很大的关系。一般会从GC Root开始向下搜索，如果对象与GC Root之间存在直接或间接的强引用有关系，则当前对象强可到达，不能被回收。如对象与GC Root之间只存在直接或间接的软引用有关系，则当前对象软可到达，GC时会视当前内存情况确定是否回收该对象。如对象与GC Root之间只存在直接或间接的弱引用有关系，则当前对象弱可到达，GC时不管内存如何该对象将都被回收，但在GC前可以再次强引用该对象达到让该对象不被回收。如对象与GC Root之间只存在直接或间接的虚引用有关系，则当前对象虚可到达，GC时该对象将被回收。

![](https://gitee.com/0909/blog/raw/master/img/20211026153107.png)

​		上面ObjectA、ObjectB、ObjectC、ObjectD、ObjectE、ObjectF、ObjectG 7个对象。

- 与GC Root存在直接或间接强引用关系的对象有 ObjectA，GC时ObjectA一定不会被回收。
- 与GC Root存在直接或间接软引用关系的对象有 ObjectB、ObjcetE，GC时ObjectB与ObjectE可能会被回收。
- 与GC Root存在直接或间接弱引用关系的对象有 ObjectC、ObjectF，GC时ObjectC与ObjectF一定会被回收。
- 与GC Root存在直接或间接虚引用关系的对象有 ObjectD、ObjectG，GC时ObjectD与ObjectG一定会被回收。

##### 2.2 ThreadLocal#get、#set、#remove方法

​	**ThreadLocal#get**整体逻辑相对简单，具体分析见下面代码的注解。当未给**ThreadLocal**设置值时，**get**方法将调用**setInitialValue**方法返回**initialValue**方法指定的**ThreadLocal**的初始值，默认**ThreadLocal**的**initialValue**为null。**ThreadLocal#get**方法实际是用其自身作为Key通过开放寻址法在其所属线程的**ThreadLocalMap**上查找对应的value（与线程绑定的变量）。

```java 
public T get() {
  //获取当前线程
  Thread t = Thread.currentThread();
  //从线程中获取成员变量ThreadLocalMap，此时ThreadLocalMap可能没被初始化
  ThreadLocalMap map = getMap(t);
  //当前线程成员变量ThreadLocalMap已初始化
  if (map != null) {
    //用this(当前ThreadLocal)为Key从ThreadLocalMap的Entry数组中找到对应的value
    ThreadLocalMap.Entry e = map.getEntry(this);
    //entry存在直接从其中拿出对应的值，然后返回
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  //当前线程成员变量ThreadLocalMap未初始化或者在其Entry数组中未找到对应的value, 设置value值
  return setInitialValue();
}
//设置ThreadLocal初始化值
private T setInitialValue() {
 	//获取ThreadLocal的初始化值
  T value = initialValue();
  //获取当前线程
  Thread t = Thread.currentThread();
  //从线程中获取成员变量ThreadLocalMap，此时ThreadLocalMap可能没被初始化
  ThreadLocalMap map = getMap(t);
  if (map != null)
    //在ThreadLocalMap上设置当前ThreadLocal对应值
    map.set(this, value);
  else
  	//为当前线程设置成员变量ThreadLocalMap值
    createMap(t, value);
  return value;
}
//为线程成员变量ThreadLocalMap设置值，同时将当前的TheadLocal对的值绑定到ThreadLocalMap上
void createMap(Thread t, T firstValue) {
  t.threadLocals = new ThreadLocalMap(this, firstValue);
}
//默认ThreadLocal的initialValue为null，创建ThreadLocal对象时可以覆盖该方法指定初始值
protected T initialValue() {
  return null;
}
```

**ThreadLocal#set**整体逻辑相对简单，具体分析见下面代码的注解。**ThreadLocal#set**方法实际是用其自身作为Key通过开放寻址法在其所属线程的**ThreadLocalMap**上将value与线程绑定。

```java
public void set(T value) {
  	//获取当前线程
    Thread t = Thread.currentThread();
    //从线程中获取成员变量ThreadLocalMap，此时ThreadLocalMap可能没被初始化
    ThreadLocalMap map = getMap(t);
    if (map != null)
      	//在ThreadLocalMap上设置当前ThreadLocal对应值
        map.set(this, value);
    else
      	//为当前线程设置成员变量ThreadLocalMap值
        createMap(t, value);
}
```

**ThreadLocal#remove**整体逻辑相对简单，**ThreadLocal#remove**方法实际是用其自身作为Key通过开放寻址法，将当前**ThreadLocal**与其所属线程解绑。

```java
public void remove() {
  ThreadLocalMap m = getMap(Thread.currentThread());
  if (m != null)
    //将当前ThreadLocal与其对应的线程解绑
    m.remove(this);
}
```

##### 2.3 ThreadLocalMap类

​		从上面的**ThreadLocal#get、#set、#remove**方法分析可以看到，最终这些操作都是在**ThreadLocalMap**上完成。文中最开始已介绍过**ThreadLocalMap**实际是通过**开放地址法**实现的，其内部的**Entry**数据组table用于存储**ThreadLocal**与保存在**ThreadLocal**的值，最终实现**ThreadLocal**内保存的值与线程绑定。

```java
static class ThreadLocalMap {
  // map的初始容量
  private static final int INITIAL_CAPACITY = 16;
  // Entry数组
  private Entry[] table;
  // Entry元素个数
  private int size = 0;
  // 阈值，用于扩容降低开放寻址时的冲突
  private int threshold;
}
//ThreadLocalMap中的Entry持有ThreadLocal的弱引用。
static class Entry extends WeakReference<ThreadLocal<?>> {
  // ThreadLocal上关联的值
  Object value;
  Entry(ThreadLocal<?> k, Object v) {
    super(k);
    value = v;
  }
}
```

**ThreadLocalMap**构造函数

```java
//创建ThreadLocalMap并在其上绑定第一个线程局部变量
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
  	//取模获取引用ThreadLocal的Entry的下标
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
  	//设置扩容的阈值
    setThreshold(INITIAL_CAPACITY);
}
```

**ThreadLocalMap#getEntry**方法以**ThreadLocal**作为**Key**从**ThreadLocalMap**采用**开放寻址法**从**Entry**数组中寻找引用**ThreadLocal**对应的**Entry**。**ThreadLocal#get**方法会调用该方法，获取保存在Entry内的Value，即**实际与当前线程绑定的变量值**。

```java 
private Entry getEntry(ThreadLocal<?> key) {
    //取模试探性获取引用ThreadLocal的Entry的下标
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
      	//发生冲突，采用开放寻址法从Entry数组中找引用ThreadLocal的Entry
        return getEntryAfterMiss(key, i, e);
}
//发生冲突，采用开放寻址法从Entry数组中找引用ThreadLocal的Entry
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
   Entry[] tab = table;
   int len = tab.length;
   while (e != null) {
     ThreadLocal<?> k = e.get();
     if (k == key) // 找到引用ThreadLocal的Entry返回
       return e;
     if (k == null) //找到之前引用ThreadLocal的Entry，但ThreadLocal的引用已被remove方法清理掉或被GC清理掉
       //通过重新哈希，清理已被remove或被GC回收的ThreadLocal上关联的value
       expungeStaleEntry(i);
     else // 继续向前寻找引用ThreadLocal的Entry
       i = nextIndex(i, len);
     e = tab[i];
   }
   return null;
 }
//开放寻址，获取元素下标
private static int nextIndex(int i, int len) {
  return ((i + 1 < len) ? i + 1 : 0);
}
```

**ThreadLocalMap#set**方法以**ThreadLocal**作为**Key**采用开放寻址法将value与其所属线程绑定。

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
  	//取模试探性获取引用ThreadLocal的Entry的下标
    int i = key.threadLocalHashCode & (len-1);
    //尝试用开放寻址的方法在Entry数组中找到之前引用ThreadLocal的Entry
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
      	//找到之前引用ThreadLocal的Entry，重置value值并返回
        if (k == key) {
            e.value = value;
            return;
        }
      	//找到之前引用ThreadLocal的Entry，但ThreadLocal的引用已被remove方法清理掉或被GC清理掉
        if (k == null) {
          	//重新将引用ThreadLocal的Entry放入到Entry数组中并清理已被remove或被GC回收的ThreadLocal上关联的value
            replaceStaleEntry(key, value, i);
            return;
        }
    }
  	//未找到之前引用ThreadLocal的Entry，创建Entry并放入Entry数组
    tab[i] = new Entry(key, value);
    int sz = ++size;
  	//清理槽位失败且Entry数组长度超过阈值，重新rehash对Entry数组扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

```

**ThreadLocalMap#remove**方法以**ThreadLocal**作为**Key**采用开放寻址法将value与其所属线程绑定。

```java
private void remove(ThreadLocal<?> key) {
  Entry[] tab = table;
  int len = tab.length;
  //取模试探性获取引用ThreadLocal的Entry的下标
  int i = key.threadLocalHashCode & (len-1);
  //用开放寻址法找到引用ThreadLocal的Entry，并将其清除
  for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
    if (e.get() == key) {
      //清除Entry上的弱引用ThreadLocal
      e.clear();
      //通过重新哈希，清理已被remove或被GC回收的ThreadLocal上关联的value
      expungeStaleEntry(i);
      return;
    }
  }
}
```

**ThreadLocalMap#getEntry、#set、#remove**方法内部最终都会尝试调用**expungeStaleEntry**方法。

**expungeStaleEntry**通过重新哈希，清理已被remove或被**GC**回收的**ThreadLocal**上关联的value， 该方法可以保证由于只与Entry存在弱引用关系的**ThreadLocal**被**GC**回收后，Entry上的Value（与**ThreadLocal**上关联的value）能被及时清理，而**不会因为Entry上的Value一直存在强引用最终导致的内存泄漏**。实际**ThreadLocal#set、#get、#remove**方法最终都会调用**expungeStaleEntry**方法。

```java
// 通过重新哈希，清理已被remove或被GC回收的ThreadLocal上关联的value
private int expungeStaleEntry(int staleSlot) {
  Entry[] tab = table;
  int len = tab.length;
  // 清理当前位置上已被remove或被GC回收的ThreadLocal上关联的value
  tab[staleSlot].value = null;
  // 清理当前位置上的Entry
  tab[staleSlot] = null;
  size--;
  // 向后重新哈希，直到对应位置上没有Entry。
  Entry e;
  int i;
  for (i = nextIndex(staleSlot, len); (e = tab[i]) != null;i = nextIndex(i, len)) {
 		ThreadLocal<?> k = e.get();
    //清理当前位置上已被remove或被GC回收的ThreadLocal上关联的value
  	if (k == null) {
      e.value = null;
      tab[i] = null;
      size--;
  	} else {
    //用开放寻址法，重新调整当前Entry在数组中的位置
      int h = k.threadLocalHashCode & (len - 1);
      //ThreadLocal初始位置h与i不一致，尝试将其放回始位置或开放寻址法后的位置
      if (h != i) {
        tab[i] = null;
        while (tab[h] != null)
       		 h = nextIndex(h, len);
        tab[h] = e;
      }
    }
  }
  return i;
}
```

#### 3 使用场景

​		本文开篇已介绍过，**ThreadLocal**适用方法调用链上参数的透传，但要注意是同线程间，但不适合异步方法调用的场景。对于异步方法调用，想做参数的透传可以采用阿里开源的**TransmittableThreadLocal**。权限、日志、事务等框架都可以利用**ThreadLocal**透传重要参数。

​		在使用**Spring Security**时，当用户认证通过后，业务逻辑处理中经常会去获取用户认证时的用户信息，通过会将这个功能封装在工具类中，如下的**SecurityUtils#getAuthUser**方法用于获取用户的认证信息，如果用户认证过返回用户信息，否则返回null。业务逻辑中直接通过**SecurityUtils#getAuthUser**方法便能方便的获取用户的认证信息。

```java
public class SecurityUtils {
  public static User getAuthUser() {
    try {
      // 通过Srping Security 上下文获取用户的认证信息
      Authentication auth = SecurityContextHolder.getContext().getAuthentication();
      return  Objects.nonNull(auth) ? (User) auth.getPrincipal() : null;
    } catch (Exception e) {
      log.error("Get user auth info fail", e);
      throw new CustomException("获取用户信息异常", HttpStatus.UNAUTHORIZED.value());
    }
	}
}
```

**SecurityContextHolder**采用策略模式实现，实默认策略便是通过ThreadLocal存储Spring Security的上下文信息，这个上下文信息中包括认证信息。**ThreadLocalSecurityContextHolderStrategy**源码如下。

```java
final class ThreadLocalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {
  //采用ThreadLocal存储线程上下文信息
	private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<>();
	public void clearContext() {
		contextHolder.remove();
	}
	public SecurityContext getContext() {
    //从ThreadLocal获取线程上下文信息
		SecurityContext ctx = contextHolder.get();
		if (ctx == null) {
			ctx = createEmptyContext();
			contextHolder.set(ctx);
		}
		return ctx;
	}
	public void setContext(SecurityContext context) {
		Assert.notNull(context, "Only non-null SecurityContext instances are permitted");
		contextHolder.set(context);
	}
  ...
}
```

#### 4 内存泄漏问题分析

##### 4.1 内存漏泄示例		

下面通过代码模拟ThreadLocal内存漏泄，注意运行指定的VM参数 -Xms大于50MB。

```java 
public class ThreadLocalOOM {
  public static void main(String[] args) throws InterruptedException {
    ThreadLocal tl = new CustomThreadLocal();
    tl.set(new Value50MB());
    //清理 CustomThreadLocal 对象的强引用
    tl = null;
    System.out.println("Call System.gc method to trigger Full GC");
    System.gc();
    //GC线程优先级较低，休眠3秒确保Full GC已完成
    Thread.sleep(3000);
  }
  public static class CustomThreadLocal extends ThreadLocal {
    private byte[] a = new byte[1024 * 1024 * 1];
    @Override
    public void finalize() {
      // Full GC 如果对象被回收，该方法会被调用
      System.out.println("CustomThreadLocal 1 MB finalized.");
    }
  }
  public static class Value50MB {
    private byte[] a = new byte[1024 * 1024 * 50];
    @Override
    public void finalize() {
      // Full GC 如果对象被回收，该方法会被调用
      System.out.println("Value50MB 50 MB finalized.");
    }
  }
}
```

控制台输出：

> **Call System.gc method to trigger Full GC**
> **My threadLocal 1 MB finalized.**

​		从上面的输出可以知道，发生**Full GC**后**CustomThreadLocal** 对象对应的**1MB**内存被回收，但其上面关联的值**Value50MB**对应的**50MB**内存并没有被**GC**回收，出现了**内存漏泄**。如果应用中存在大量的这类**ThreadLocal**关联的值没被**GC**回收到，内存不断漏泄，最终将导致应用程序整体**OOM**，程序崩溃。

**4.2 原因分析**

​		在第2节部分源码分析中，已知道实际**ThreadLocal**与其保存的值都是被放在**ThreadLocalMap**内部**Entry**对应的实例上。而**Entry**持有**ThreadLocal**的弱引用，当**ThreadLocal**只被**Entry**引用时，**ThreadLocal**对象将在**GC**时被无条件的回收掉。

​		上面**ThreadLocal**内存漏泄的示例中，强弱引用关系如下。

**强引用：**

- **ThreadLocalRef  =>ThreadLocal **
- **ThreadRef  => Thread  => ThreadLocalMap => Entry数组 => Entry => Value50MB**

**弱引用：**

- **Entry => ThreadLocal **



![](https://gitee.com/0909/blog/raw/master/img/20211027153948.png)

```
 tl = null;
```

执行 t1 = null后，强弱引用关系如下。

**强引用：**

- **ThreadRef  => Thread  => ThreadLocalMap => Entry数组 => Entry => Value50MB**

**弱引用：**

- **Entry => ThreadLocal **

![](https://gitee.com/0909/blog/raw/master/img/20211027154315.png)

**GC**时发现 **TheadLocal**上只存在 **Entry**对其的弱引用，于是无条件将**ThreadLocal**对应的内存回收，示例中是**CustomThreadLocal**对应的**1M**内存。

**4.3  如何避内存漏泄**

​		从第2节中的源码分析中已知道，**ThreadLocal#remove**方法实际会调用**ThreadLocalMap#expungeStaleEntry **方法，达到将已被**GC**回收的**ThreadLocal**上关联的Value的强引断开，避免内存泄漏。所以在每次使用完**ThreadLocal**后，只要调用其对应的**remove**方法，就可以避内存漏泄。

##### 4.4 为什么Entry不强引用ThreadLocal

**ThreadLocalMap**源码上的注解

> To help deal with very large and long-lived usages, the hash table entries use WeakReferences for keys. However, since reference queues are not used, stale entries are guaranteed to be removed only when the table starts running out of space.

可以看到主要是为了避免大的**ThreadLocal**与长时间存活的使用场景。如果不采用**Entry**弱引用**ThreadLocal**，**ThreadLocal**将一直与**Thread**共存，这更加容易引起内存漏泄。

#### 总结

​		本文首先对ThreadLocal做出了整体的概述，简要地说明其使用场景、不足、业界的改进方案。然后对**ThreaLocal**的源码进行了详细地分析，接着介绍了其具体的使用场景、日常使用中可能会遇到的问题与问题的解决方案。

#### 参考

[TransimittableThreadLocal](https://github.com/alibaba/transmittable-thread-local)

[Java Reference核心原理分析](https://mp.weixin.qq.com/s/8f29ZfGvZVPe0bO-FahokQ)

#### 结尾

​		原创不易，点赞、在看、转发是对我莫大的鼓励，关注公众号洞悉源码是对我最大的支持。同时相信我会分享更多干货，我同你一起成长，我同你一起进步。
