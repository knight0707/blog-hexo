---
title: 一文详解JDK9之前的ClassLoader
date: 2021-11-16 20:20:19
tags:
  - ClassLoader
  - SPI
  - JDK源码
  - Fat-Jar
categories: JDK源码
---

### 一文详解JDK9之前的ClassLoader

​		提到类加载器（ClassLoader）也许能让你想起过往开发中遇到的诸如类冲突、ClassNotFoundException、NoClassDefFoundError之类的令人头疼的问题。Google了一下，关于ClassLoader的文章很多，但我还是觉得没有把一些核心的知识点结串再一起，大多数文章停留在介绍双亲委派机制上，而且很多文章都是基于JDK8或之前的版本去分析ClassLoader的，JDK9模块化的引入已将历史的双亲委派机制破坏。今天先总结一下JDK8涉及的知识点与源码，后面再专门针对JDK9中的ClassLoader写一篇分析。下面是本文的总体内容。

![](https://gitee.com/0909/blog/raw/master/img/20211124114341.png)

**注意：后面的涉及到JDK源码分析的部分，若没有明确指出都是JDK8的源码**

#### 1 ClassLoader概述

##### 1.1 ClassLoader做什么的？

​		ClassLoader顾名思义他是用来加载类的，ClassLoader的设计源为了满足Java Applet的需求。当Java源代码经过编译后会生成对应的字节码文件（ *.class文件，当然也可按Java语言规范去生成的 *.class文件），这种字节码本质是遵守JVM规范的字节数组。 ClassLoader负责将字节码文件转换成JVM内存中的Class对象（这些字节码文件可以来源本地磁盘文件 *.class、也可以来源Jar包里的  *.class、甚至可以来源于远程服务器的字节流）。通常利用相应的反编译工具便可将  *.class文件转换成源代码文件，通过加密技术能对字节码文件进行密处理，但加密后的字节码文件已不再是遵守JVM规范的字节码数组，其不能被系统类加载器直接加载到JVM中，需要我们自己实现对应的ClassLoader。

```java
class Class<T> implements java.io.Serializable, ...{
  private final ClassLoader classLoader;
  private Class(ClassLoader loader) {
    classLoader = loader;
  }
 	...                               
} 
```

​		Class类中有对应的ClassLoader成员变量，用于表明当前Class对象是由哪个ClassLoader加载而来的。**注意：JVM中判断一个类是不是同一个类不是直接通过类的全限定名是否相等来的。两个类要相同其类的全限定名要相同并且要由相同的类加载器加载，否则在作赋值、equals、isAssignableFrom、instanceof等操作时都会出现问题。**

##### 1.2  延迟加载

​		一个类什么时候被加载到JVM呢？JVM运行时并不是将工程依赖的所有类，一次性全加载到内存实例化成Class对象，而是按需加载，也称延迟加载。程序在运行过程中会逐步加载其需要用到的类，一个类一旦被ClassLoader加载过后就会缓存在内存中，程序运行中要再次使用该类的话会直接从缓存中取出。

​		当程序调用某个类的静态方法时，该类肯定要被加载的，但该类的实例成员变量对应的类并不需要加载，但静态成员变量对应的类可能会被加载，因为静态方法可能会访问该静态成员变量。而实例成员变量对应的类需要等到实例化对象的时候才会加载。

```java
public class ClassLazyLoadTest {
  public static void main(String[] args) {
    MainClass.callStaticMethod();
  }
  static class MainClass {
    FieldClass fieldClass;
    static void callStaticMethod() {
    }
  }
  static class FieldClass {
  }
}
```

​		上面的代码启动时加上VM参数 **-verbose:class** 能看到有哪些类被加载到JVM中。运行结果如下：

> ...
> [Loaded sun.net.spi.DefaultProxySelector$1 from /Library/Java/JavaVirtualMachines/jdk1.8.0_241.jdk/Contents/Home/jre/lib/rt.jar]
> [Loaded **org.fzdata.classloader.ClassLazyLoadTest** from file:/Users/mac/Desktop/0909/zc/backend-service/target/classes/]
> [Loaded sun.net.NetProperties$1 from /Library/Java/JavaVirtualMachines/jdk1.8.0_241.jdk/Contents/Home/jre/lib/rt.jar]
> ...
> [Loaded java.lang.Void from /Library/Java/JavaVirtualMachines/jdk1.8.0_241.jdk/Contents/Home/jre/lib/rt.jar]
> [Loaded **org.fzdata.classloader.ClassLazyLoadTest$MainClass** from file:/Users/mac/Desktop/0909/zc/backend-service/target/classes/]
> ...

​		可以看到ClassLazyLoadTest类与MainClass类被加载到JVM中，而FieldClass类虽然是MainClass类的实例成员变量，但由于main方法中调用MainClass.callStaticMethod()时只需要类MainClass就行，而FieldClass类只有当实例化MainClass对应的对象时才需要加载。

##### 1.3  类的生命周期

 		从JVM将类加载到内存到调用类的方法使用类，要经过一系列阶段包括：加载 Loading、连接 Linking（校验 Verification、准备 Preparation、解析 Resolution）、初始化 Initialization 、使用 Using 、卸载 Unloading。其中 校验、准备、解析统一归属连接。

![](https://gitee.com/0909/blog/raw/master/img/20211119123740.png)

​		加载、验证、准备、初始化与卸载这五个阶段的顺序是确定，类的加载过程按上图的顺序按部就班地开始，而解析阶段则不一定JVM规范并没有明确解析的顺序，其由可能发生在初始化阶段之后。**注意：上图表示的是各阶段的开始时间之间的顺序关系，而不是说验证阶段要等待加载阶段完成以后才能进行，实际这些阶段通常都是互相交叉地混合式进行的，在一个阶段执行的过程中会调用或激活下一个阶段。**

- **加载阶段**

  该阶段主要将不同来源的类的字节码文件转化为Class对象，以便访问类的各类信息。前面提过类加载是延迟加载，只有真正要使用类时才会触发类的加载。类的加载过程包括加载、验证、准备、解析、初始化的五个阶段，从Java开发的角度来看，只有加载阶段能被干预，其他阶段都是由JVM主导。

- **验证阶段**

  验证阶段主要是校验字节码文件是否符合JVM的规范，主要分为文件格式校验、元数据验证、字节码验证等。

- **准备阶段**

  准备阶段是正式为类变量（被static修饰的变量）分配内存并设置零值的阶段，其内存都分配在方法区，不同类型的变量都会有对应的零值比如引用类型是null、int 是0 、boolean是false等。**注意：这个阶段只会为类变量分配内存，而实例变量内存的分配要随着对象实例化时一起分配在堆内存中; 同时如果类变量是一个静态常量（同时被static与final修饰的变量），这阶段就会为其赋值成对应的常量值。**

  ```java
  public static int a = 9;
  public static final int b = 613;
  public static String c = "yeyi";
  ```

  上面的例子在这个阶段 a 的值为0，b值为613，c值为 null。

- **解析阶段**

  解析阶段实际是将符号引用解析成为直接引用，符号引用只是一个区分的标识并不指向对象真正存储的内存地址，而直接引用可以直接指向目标的指针、相对偏移量或能定位到目标的句柄。比如Person类中引用了一个 Address 类，一开始 Person 类并不知道 Address 类在内存中的地址，所以就先搞个符号引用替代一下，假装知道，等类加载解析的时候再将这个符号引用替换为Address 类真正的内存地址。

- **初始化阶段**

  初始化是类加载的最后一步，初始化阶段会为类变量赋予初始值（不包括静态常量，同时被static与final修饰的变量）。其实际是执行类的初始化方法clinit<>，该方法会被加锁执行，且只执行一次。clinit<>方法会按程序定义的顺序收集类中的static代码块将其放入clinit<>方法中。clinit<>方法与对象的构造函数init<>不同，其不能显示调用父类的clinit<>方法，JVM会保证调用子类的clinit<>方法前，父类的clinit<>方法已被调用。JVM 规范 Java SE 8 版本第五节中明确规定只有6种情况会触发类的初始化，同时在 Java语言规范 **Initialization of Classes and Interfaces**一节中有相对于Java工程师更直观的说明。下面是JVM规范 Java SE 8 版本明确的会触发类的初始化的6种情况。

  > **1、JVM启动时用户要指定一个主函数（main方法），JVM会先初始化该方法所属的类。**
  >
  > **2、执行new、getstatic、putstatic、invokestatic节码指令时引用了对应类且类未初始化，则先初始化该类。这4条指令常见的Java代码场景是：使用new关键字实例化对象、读取或设置一个类的静态变量（被final修饰的静态变量除外，其值在常用池中）、调用一个类的静态方法。**
  >
  > **3、通过java.lang.reflect包或Class类中方法对类进行反射操作时，如果类未初始化，则会先初始化该类。**
  >
  > **4、当初次调用MethodHandle 实例时，初始化该MethodHandle指向的方法所在类。**
  >
  > **5、当一个类的其子类在初始化时，如果该类未初始化则会触发其初始化。**
  >
  > **6、当一个接口定义了default方法，其直接或间接实现类在初始时，若该接口未初始化则会触发其初始化。**

  **上面的这6种情况也被叫作主动使用，其他的都是被动使用，即使代码中引用了对应的class，JVM也不会对类进行初始化。**

  可以用下面方式验证接口与类是否初始化。

  ```java
  public interface A {
    //验证接口是否初始初始化
    int a = Test.out("A", 1);
  }
  public class Test {
    public static int out(String i, int v) {
      System.out.println("Interface" + i + "init!");
      return v;
    }
  }
  public class B {
    static {
      //验证类是否初始初始化
      System.out.println("Class B init!");
    }
  }
  ```

  **注意：验证类是否加载与类是否初始化不要混淆，一个类可能已加载但没有初始化，类是否加载通过运行时加-verbose:class能看到，类是已初始化可以通过上面的代码看输出。**

#### 2 双亲委派机制

​		ClassLoader加载资源与类时遵循代理模式，每一个ClassLoader有对应的父类加载器。当ClassLoader要加载一个类时，其会先尝试让他的父类加载器去加载该类，只有在父类加载器无法加载该类时其自己才会尝试去加载该类，而父类加载器又会按照这个规则优先让他的父类加载器去加载类，直到最上层的引导类加载器，这种JVM加载类的默认规则通常被叫作双亲委派模型。

##### 2.1 类加载器各司其职

​		JVM运行时会存在很多种ClassLoader，不同的ClassLoader负责从不同的地方加载字节码文件。JVM内置了三个重要的类加载器分别是**引导类加载器（Bootstrap ClassLoader）、扩展类加载器（Extension ClassLoader）与应用类加载器（App ClassLoader，也叫系统类加载器System ClassLoader**。

​		引导类加载器主要负责加载 JDK中的核心类，这些类位于 <JAVA_HOME>/lib/rt.jar 文件中，日常开发常用的JDK类都在java.xxx.* 里面，比如 java.util.*、java.io.*、java.nio.*、java.lang.* 等等。引导类加载器比较特殊，它是由c语言实现的，他也被叫作**根加载器**。

​		扩展类加载器主要负责加载 JDK中的扩展类，这些类通常位于 <JAVA_HOME>/lib/ext/*.jar 文件中，但可以通过设置系统运行参数 **-Djava.ext.dirs=你要指定的扩展类目录** 进行修改，不过最好别这么做。

​		应用类加载器主要负责加载工程中自己开发的类以及依赖第三方类库，它加载在环境变量 CLASSPATH、-classpath 或 -cp 命令行选项中找到的应用程序类型类。可以通过ClassLoader 类提供的静态方法 getSystemClassLoader() 获取应用类加载器。当启动一个main函数时，应用类加载器开始加载main函数所属的类。

​		JDK中还内置了URLClassLoader用于从指定文件目录或是从网络中加载字节码文件，扩展类加载器与应用类加载器都是他的子类。扩展类加载器与应用类加载器都sun.misc.Luancher的内部类，分别对应ExtClassLoader类与AppClassLoader类。

​		上面提到的引导类加载器、扩展类加载器、应用类加载器要加载类的路径都可以通过参数配置去改变其默认行为。

| ClassLoader 类型          | 参数选项                                                     | 说明                                                         |
| :------------------------ | :----------------------------------------------------------- | ------------------------------------------------------------ |
| **Bootstrap ClassLoader** | -Xbootclasspath: <br>-Xbootclasspath/a: <br/>-Xbootclasspath/p: <br/> | 设置Bootstrap ClassLoader的搜索路径<br>把路径添加到己存在Bootstrap ClassLoader搜索路径的后面<br>把路径添加到已存在Bootstrap ClassLoader搜索路径的前面<br> |
| **ExtClassLoader**        | -Djava.ext.dirs                                              | 设置ExtClassLoader的搜索路径                                 |
| **AppClassLoader**        | -Djava.class.path=<br>-cp<br>-classpath                      | 设置AppClassLoader的搜索路径                                 |

##### 2.2 双亲委派模型

​		本小节最前面就介绍过JVM加载类的过程默认遵循双亲委派模型，当前的类加载器在加载类时会优先让其父类加载器去加载，直到最上层的引导类加载器，如果父类加载器加载不了再尝试自去加载类。上面已介绍过引导类加载器、扩展类加载器、应用类加载器都具体负责加载哪些类。引导类加载器是扩展类加载器的父类加载器，其本身是用c言语实现的，他没有父类加载器在JDK中也找不到对应的类，所以像System、Thread、Integer等java.xxx.*包下面的类，通过他们的Class#getClassLoader时都会返回null。扩展类加载器是应用类加载器的父类加载器，其对应的类为sun.misc.Luancher$ExtClassLoader，而应用类加载对应的类为sun.misc.Luancher$AppClassLoader。

![](https://gitee.com/0909/blog/raw/master/img/20211118091315.png)

​		当应用类加载在尝试加载一个类时，会先尝试让他的父类加载器扩展类加载器去加载这个类，而扩展类加载器在尝试加载这个类时又会尝试让他的父类加载器引导类加载器去加载这个类。如果上层的类加载器加载类成功，则类加载完成，否则对应的下层自己再尝试去加载类。**注意：父类加载器不能访问子类加载器加载的类，而子类加载却能访问父类加载器加载的类。上面说的父类加载器与子类加载器，不要与类继承关系上的父子类的概念混淆，AppClassLoader 的父类加载器是ExtClassLoader，但在的类继承上他们之间不存在父子关系，他们都是URLClassLoader的直接子类。**

##### 2.3 ClassLoader#loadClass源码

​		在ClassLoader#loadClass方法内基本能看到上面介绍的双亲派模型的整体工作流程。下是省略部分非核心代码后JDK8中ClassLoader#loadClass方法的源码。loadClass方法源码相对简单，这不再分析。

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
  synchronized (getClassLoadingLock(name)) {
    // 从缓存中查找是否已加载类
    Class<?> c = findLoadedClass(name);
    if (c == null) {
      try {
        // 存在父类加载器优先由父类加载器加载类
        if (parent != null) {
          // 递归调用父类加载器的loadClass方法加载类
          c = parent.loadClass(name, false);
        } else {
          // 由引导类加载器加载类
          c = findBootstrapClassOrNull(name);
        }
      } catch (ClassNotFoundException e) {
        // 如果抛出该异常说明父类加载无法加载类
      }
      if (c == null) {
        // 父类加载器无法加载，自已尝试通过findClass加载类
        c = findClass(name);
      }
    }
    if (resolve) {
      resolveClass(c);
    }
    return c;
  }
}
/**ClassLoader中findClass默认会抛出ClassNotFoundException
 要子类重写该方法实现自定义的类加载器**/
protected Class<?> findClass(String name) throws ClassNotFoundException {
  throw new ClassNotFoundException(name);
}
```

##### 2.4 为什么要引入双亲委派模型？

​		之前提到过JVM运行时要判断是否为同一个类（**可能用同一个类的实现 Class实例会更恰当**），必须类的全限定名相同且类由同一个类加载器加载。有了双亲派模型后就可以确保JVM中JDK的核心类在内存的唯一性。即使自己编程了一个名为java.lang.Thread类，按照双亲派模型最后Thread也是被引导类加载器加载，正常情况应用类加载器根本没机会去加载java.lang.Thread类。双亲派模型的存在保证的JDK核心类库的安全性，别人没法随意篡改核心类的实现。

##### 2.5 自定义类加载器

​		**通常可以通过继承ClassLoader重写findClass方法来实现自定义的类加载器，这样可以保证类加载流程遵循双亲委派模型，但些情况下也可以重写loadClass方法**。双亲委派模型是在JDK1.2时引入的，在这之前loadClass方法已存在，为了向后兼容引入了findClass方法。重写loadCloass会打破双亲委派模型，关于打破双亲委派机制将在后面介绍。前面了解到ClassLoader中findClass方法默认抛出ClassNotFoundException异常。重写findClass方法实现自定义ClassLoader，一般分两步：

- **在findClass内部调用私有的loadClassData方法，根据给定名字获取符合JVM规范的字节码数组**
- **将获取到的字节码通过CloassLoader#defineClass方法生成对应的Class实例**

```java
class CustomClassLoader extends ClassLoader {
  public Class findClass(String name) {
    //1、获取符合JVM规范的字节码数组
    byte[] b = loadClassData(name);
    //2、调用defineClass方法生成Class实例
    return defineClass(name, b, 0, b.length);
  }
  private byte[] loadClassData(String name) {
    //TODO load the class data from somewhere
  }
}
```

#### 3 打破双亲委派机制

​		前面说过正常情况下尝试加载类的路径是：**引导类加载器 =〉扩展类加载器 =〉应用类加载器 =〉用户自定义类加载器，即遵循双亲委派模型**。但如果引导类加载器，加载类时要用到非JDK中的类呢？这时只能选择打破类加载的双亲委派机制。

##### 3.1 打破双亲委派机制的方式

​		先看一下打破双亲委派机制的两种方式**重写ClassLoader#loadClass方法**与**通过Thread#getContextClassLoader获取的类加载器去加载类**。

- **重写ClassLoader#loadClass方法**

  前面在实现自定义类加载时就提到过重写ClassLoader#loadClass方法能会打破双亲委派机制，从上面ClassLoader#loadClass的源码分析可以看到双亲委派机制实际在这个方法中实现，一旦重写该方法就可以改变类加载的流程。

- **通过Thread#getContextClassLoader获取的类加载器去加载类**

  一方面Thread内部有一个类型为ClassLoader的contextClassLoader变量，该变量默认值为AppClassLoader，该变量可以通过Thread#setContextCalssLoader方法设置、通过Thread#getContextClassLoader方法获取。

  ```java 
  //Thread#setContextClassLoader方法，会根据安全管理器的策略查看是否有权限设置contextClassLoader
  public void setContextClassLoader(ClassLoader cl) {
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
      sm.checkPermission(new RuntimePermission("setContextClassLoader"));
    }
    contextClassLoader = cl;
  }
  //Thread#getContextClassLoader方法，会根据安全管理器的策略查看是否有权限获取contextClassLoader
  public ClassLoader getContextClassLoader() {
    if (contextClassLoader == null)
      return null;
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
      ClassLoader.checkClassLoaderPermission(contextClassLoader,
    					 Reflection.getCallerClass());
    }
    return contextClassLoader;
  }
  ```

  另一方面**Class#forName**可以传入指定的类加载器来加载类。

  ```java 
  public static Class<?> forName(String name, boolean initialize,
                                 ClassLoader loader)
    throws ClassNotFoundException {
    Class<?> caller = null;
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
      caller = Reflection.getCallerClass();
      //省略安全校验源码
    }
    return forName0(name, initialize, loader, caller);
  } 
  private static native Class<?> forName0(String name, boolean initialize,
                                          ClassLoader loader,
                                          Class<?> caller)
    throws ClassNotFoundException;
  ```

  **重点来了，有了与Thread绑定的contextClassLoader，再结合Class#forName能指定类加载器加载类，就可以通过切换与Thread绑定的ClassLoader实现想用什么类加载器就用什么类加载器，这最终将打破双亲委派机制**。

  **有一点要注意Thread中contextClassLoader默认为AppClassLoader**。Java中所有线程都是直接或间接由主线程（main方法所属线程）创建的（主线程本身除外），而主线程对应的contextClassLoader默认为AppClassLoader。通过下面Thread构建方法最终调用的init方法可以看到线程在创建时默认contextClassLoader为其父类的contextClassLoader，即所有线程默认的contextClassLoader都是AppClassLoader。

  ```java
  //Thread构造方法最终调用的Thread#init方法，contextClassLoader赋值相关源码
  private void init(ThreadGroup g, Runnable target, String name,
                    long stackSize, AccessControlContext acc,
                    boolean inheritThreadLocals) { 
    //...省略其他源码
  	if (security == null || isCCLOverridden(parent.getClass()))
      this.contextClassLoader = parent.getContextClassLoader();
    else
      this.contextClassLoader = parent.contextClassLoader;
    //...省略其他源码
  }
  ```

  结合上面说的Thread中contextClassLoader默认为AppClassLoader与Class#forName，先来提前看一下Java内部提供的服务发现SPI机制打破双亲委派机制的核心源码：

  ```java
  //ServiceLoader#load加载服务
  public static <S> ServiceLoader<S> load(Class<S> service) {
    //获取当前线程的ClassLoader，这个ClassLoader是AppClassLoader
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
  }
  //ServiceLoader$LazyIterator#nextService 获取下一个可用服务实现，省略无关代码
  private S nextService() {
  	Class<?> c = null;
    //外部实际接口实现类的全限定类名如exapmle.MysqlRepository
    String cn = nextName;
  	try {
      	//loader参数为上面的AppClassLoader
  	    c = Class.forName(cn, false, loader);
  	} catch (ClassNotFoundException x) {
  	  //省略无关代码...
  	}
    //省略无关代码...
  	try {
  	    S p = service.cast(c.newInstance());
  	    providers.put(cn, p);
  	    return p;
  	} catch (Throwable x) {
      //省略无关代码...
  	}
  	throw new Error();
  };
  ```

  **按照之前讲的双亲委派机制java.util.ServiceLoader类最终由引导类加载器完成，而ServiceLoader本身实现又要通过Class.forName方法去加载非JDK的类如com.mysql.jdbc.Driver，直接用引导类加载器一定加载不了的，只能通过指定Class.forName的CloassLoader参数了。**

##### 3.2 Service Provider Interface (SPI)	

​		Java语言提供了一种叫作Service Provider Interface (SPI)机制，该机制能为程序提供动态的扩展点，标准与规范的定义方负责对核心功能给出接口定义，而第三方的厂商则去实现对应的接口。以JDBC为例，JDBC规范定义了Driver接口，而像mysql厂商则提供了具体的connector Jar，并在Jar中包含`META-INF/services`目录，该目录下有一个名为java.sql.Driver的文件，即JDBC规范定义的Driver接口，其内容为mysql厂商的具体实现类：

> com.mysql.jdbc.Driver
> com.mysql.fabric.jdbc.FabricMySQLDriver

在JDBC4规范中，作为使用者只要classpath下面有对应的mysql connector，直接用下面代码便可以获取mysql数据库的连接。

```java 
Connection conn = 
  DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb", "test","test");
```

同样如果是SQL Server的话，只要修改对应获取Connection的url参数，更改classpath下面connector为SQL Server 就可以获取SQL Server数据库的连接。这样一来就将不同厂商在数据库Driver实现上差别给屏蔽了，达到了关注点分离的目的，业务开发人员只要按照JDBC的规范去写业务逻辑，当像Mybatis、Hiberante等ORM框架又在JDBC规范上做更多的封装让操作数据库变得更加友好简单。

可以看到SPI整体可分为三步服务定义、服务提供、服务调用

- **服务定义** 主要是为服务或功能提供接口的定义与抽象。下面代码是Repository服务的定义。

  ```java
  package example;
  public interface Repository {
    void save(String data);
  }
  ```

- **服务提供** 根据自身特点实现服务定义阶段定义的接口，并在classpath下面创建`META-INF/services`目录，该目录下有一个文件名与已定义好的接口名一致的文件(如上看到的java.sql.Driver)，文件内容为该接口的具体实现类的类名，如果有多个具体实际则换行，一行一个具体实现类的类名。下面代码是Repository服务的不同实现，以及按SPI规则将服务暴露出去。

  ```java
  package example;
  public class MysqlRepository implements Repository {
    @Override
    public void save(String data) {
      System.out.println(data + " saved by Mysql Repository!");
    }
  }
  ```

  ```java
  package example;
  public class MongoDBRepository implements Repository {
    @Override
    public void save(String data) {
      System.out.println(data + " saved by MongoDB repository!");
    }
  }
  ```

  META-INF/services/example.Repository 文件内容如下：

  > example.MongoDBRepository
  > example.MysqlRepository		

- **服务调用** 通过classpath下面引用上面服务提供方实现的具体Jar（如果服务提供与服务调用在同一工程可以直接进行使用），然后通ServiceLoader#load方法加载对应服务，并通过ServiceLoader#iterator方法遍历可用的服务。

  ```java
  //通过load方法加载对应服务
  ServiceLoader<Repository> serviceLoader = ServiceLoader.load(Repository.class);
  //遍历可用的服务
  Iterator<Repository> iterator = serviceLoader.iterator();
  while (iterator.hasNext()) {
    Repository repository = iterator.next();
    //调用实际服务
    repository.save("Data");
  }
  ```

介绍完SPI机制后，再次把SPI打破双亲委派模型核心代码看一下：

```java
//ServiceLoader#load加载服务
public static <S> ServiceLoader<S> load(Class<S> service) {
  //获取当前线程的ClassLoader，这个ClassLoader是AppClassLoader
  ClassLoader cl = Thread.currentThread().getContextClassLoader();
  return ServiceLoader.load(service, cl);
}
//ServiceLoader$LazyIterator#nextService 获取下一个可用服务实现，省略无关代码
private S nextService() {
	Class<?> c = null;
  /**以上面Repository服务的SPI为例，变量cn实际是保存在META-INF/services/example.Repository
  文件中的类名example.MongoDBRepository或example.MysqlRepository每次遍历服务时获取一个**/
  String cn = nextName;
	try {
    	//loader为ServiceLoader#load方法中通过contextClassLoader获取到的AppClassLoader
	    c = Class.forName(cn, false, loader);
	} catch (ClassNotFoundException x) {
	  //省略无关代码...
	}
  //省略无关代码...
	try {
	    S p = service.cast(c.newInstance());
	    providers.put(cn, p);
	    return p;
	} catch (Throwable x) {
    //省略无关代码...
	}
	throw new Error();
};
```

​		**那面试时怎么回答”JDBC4规范是怎么打破类加载的双亲委派机制？“才更加恰当直观。** 希望看完上面的源码分析对你有所启示。

##### 3.3  OSGi

​		OSGi(Open Service Gateway Initiative) 技术是 Java 动态化模块化系统的一系列规范，在JDK9之前其实际是业界Java模块化的标准。OSGi通过自定义类加载器实现了模块化热部署。OSGi中程序程序模块被称作Bundle，每个Bundle都有自已的类加载器，当要更换模块时就将Bundle与类加载器一同更换以实现热部署。OSGi相对复杂，本人项目中还未真真接触过，但按照我的推理其改变类加载的双亲委派机制无非也是基于**重写ClassLoader#loadClass方法**或**通过Thread#getContextClassLoader获取的类加载器去加载类**。我这里就抛砖引玉了，OSGi学习曲线很高并且项目中真正用到的比较少，如果真对类似技术感兴趣，我推荐大家看看蚂蚁金服的容器隔离sofa-ark。

#### 4 关于四打破双亲委派机制

​		从JDK1.2双亲委派机制提出到JDK9，双亲委派机制一次又一次的被打破。大体可以分为下面四次。

- **向后兼容Class#loadClass方法打破双亲委派机制**
-  **SPI机制打破双亲委派机制**
- **OSGi模块化热加载技术打破双亲委派机制**
-  **JDK9 引入模块化打破双亲委派机制**

这些都最终都可以归为第3小节介绍的两种方式**重写ClassLoader#loadClass方法**与**通过Thread#getContextClassLoader获取的类加载器去加载类**。

##### 4.1 向后兼容Class#loadClass方法打破双亲委派机制

​		在JDK1.2提出双亲委派机制之前，JDK之前的版本中ClassLoader#loadClass方法已存在，JDK1.2对ClassLoader#loadClass方法进行了重构将双亲委派机制的实际放入到了该方法。ClassLoader#loadClass实际是public的，为了后面兼容那些用老版本通过ClassLoader#loadClass实现自定义类加载的用户，JDK1.2引入了ClassLoader#findClass方法。后继在自定义类加载器就建议重写ClassLoader#findClass方法，来维护JDK1.2提出双亲委派机制。所以说ClassLoader#loadClass为打破双亲委派机制留入了入口。当然双亲委派机制不是在什么场景都是最合适的，所以打破他也是必然。

##### 4.2  SPI机制打破双亲委派机制

​		第3小节已从SPI机制核心类ServiceLoader源码分析了，ServiceLoader类是通过引导类加载器加载的，而其内部调用的方法要通过Class#forName方法加载`META-INF/services/自定义接口名`文件中非JDK核心库中的类，即要通过引导类加载器去加载非JDK核心库中的类，按照双亲委派机制这是实现不了的。**于是有了Thread#contextClassLoader，通过Thread#getContextClassLoader获取的类加载器去加载类，便可实现上面的功能**。

##### 4.3  OSGi模块化热加载技术打破双亲委派机制

​		OSGi模块化技术中实现了自定义的类加载器，每一个模块都有各自的类加载器，其类加载流程没有完全遵循双亲委派机制中自下而上的委托，而是存在很多平级之间的类加载器查找。

##### 4.4  JDK9 引入模块化打破双亲委派机制

​		 JDK9 引入模块化后，类打包与加载都是按模块来的，而不是按上面说的Bootstrap ClassLoader、Extension ClassLoader、ExtClassLoader各自加载什么目录下面的类，同时Bootstrap ClassLoader、Extension ClassLoader、ExtClassLoader的具体实现也发生了很大改变。关于JDK9中的ClassLoader的分析，后面再会专门写一篇文章。

#### 总结

​		本文先对ClassLoader做了整体的概述，然后介绍了JDK8中类加载的双亲委派模型，而后又介绍了打破双新委派模型的相关的内容。希望通过本文你对JDK中的ClassLoader有更加系统性的了解，而不只是停留在双亲委派模型上。后面我将再转门针对JDK9详细分析一下ClassLoader。

#### 参考

[The Java® Virtual Machine Specification Java SE 8 Edition](https://docs.oracle.com/javase/specs/jvms/se8/jvms8.pdf)

《深入理解Java虚拟机：JVM高级特性与最佳实践》

[sofa-ark类隔离技术分析调研](https://blog.mythsman.com/post/5d29b12c373f140fc98304a1/)

#### 结尾

​		原创不易，点赞、在看、转发是对我莫大的鼓励，关注公众号洞悉源码是对我最大的支持。同时相信我会分享更多干货，我同你一起成长，我同你一起进步。
