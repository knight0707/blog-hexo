---
title: 深入分析Dubbo SPI 机制实现原理之实践应用
date: 2022-01-11 19:19:19
tags:
  - Dubbo
  - SPI
categories: Dubbo
---

### 深入分析Dubbo SPI 机制实现原理之实践应用

​		Dubbo采用"微内核+插件"的方式实现了对扩展开发对修改封闭，有效地将框架的内核与扩展进行了解耦。Dubbo扩展jdk原生的SPI机制，实现依赖注入、AOP、按需创建实例三个核心功能。本文将从JDK原生的SPI机制开始，逐步深入分析Dubbo中的SPI机制。Dubbo中的SPI机制利用了策略模式+工厂模式+单例模式+配置文件，深入理解其原理有利于日常开发写出扩展性更高、耦合度更低的高质量代码。深入分析Dubbo SPI 机制实现原理将分为两篇《深入分析Dubbo SPI 机制实现原理之应用》与《深入分析Dubbo SPI 机制实现原理之源码解读》，《深入分析Dubbo SPI 机制实现原理之实践应用》即本篇侧重分析Dubbo的SPI机制整体如何实现与具体实践应用，《深入分析Dubbo SPI 机制实现原理之源码解读》侧重结合源码深入分析。

#### 1 JDK 中SPI机制

​		SPI (Service Provider Interface) 是jdk中自带的一种服务定义与提供机制。服务定义一般由权威的组织或机构完成，即服务相关的核心接口的定义；服务提供一般由第三方厂商实现，即服务相关的核心接口的具体实现；服务的使用者并不关心不同厂商具体实现的差异，而是按照标准的接口完成实现业务逻辑。SPI中最具代表性的便是JDBC规范，JDK中给出的数据操作的核心接口定义，而像mysql、SQL Server、Oraclet等数据库厂商则提供了具体实现的Connector jar实现。以JDBC为例，JDBC规范定义了Driver接口，而像mysql厂商则提供了具体的Connector jar，并在jar中包含`META-INF/services`目录，该目录下有一个名为java.sql.Driver的文件，即JDBC规范定义的Driver接口，其内容为mysql厂商的具体实现类：

> com.mysql.jdbc.Driver
> com.mysql.fabric.jdbc.FabricMySQLDriver

在JDBC4规范中，作为使用者只要classpath下面有对应的mysql connector，直接用下面代码便可以获取mysql数据库的连接。

```java 
Connection conn = 
  DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb", "test","test");
```

同样如果是SQL Server的话，只要修改对应获取Connection的url参数，更改classpath下面connector为SQL Server 就可以获取SQL Server数据库的连接。这样一来就将不同厂商在数据库Driver实现上差别给屏蔽了，达到了关注点分离的目的，业务开发人员只要按照JDBC的规范去写业务逻辑，当然像Mybatis、Hiberante等ORM框架又在JDBC规范上做更多的封装让操作数据库变得更加友好简单。

##### 1.1 JDK 中SPI如何使用

可以看到SPI整体可分为三步服务定义、服务提供、服务调用

- **服务定义** 主要是为服务或功能提供接口的定义与抽象。下面代码是Repository服务的定义。

  ```java
  package example;
  public interface Repository {
    void save(String data);
  }
  ```

- **服务提供** 根据自身特点实现服务定义阶段定义的接口，并在classpath下面创建`META-INF/services`目录，该目录下有一个文件名与已定义好的接口名一致的文件（如上看到的java.sql.Driver），文件内容为该接口的具体实现类的类名，如果有多个具体实际则换行，一行一个具体实现类的类名。下面代码是Repository服务的不同实现，以及按SPI规则将服务暴露出去。

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

##### 1.2 JDK 中SPI核心源分析

​		JDK的SPI的本质是从配置文件中获取接口的具体实现类，然后通过类的全限定名加载类并通过无参数的构造函数反射创建对象。下面结合上面的例子简单的分析一下SPI的核心源码。

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
    	//参数loader为ServiceLoader#load方法中通过contextClassLoader获取到的AppClassLoader
	    c = Class.forName(cn, false, loader);
	} catch (ClassNotFoundException x) {
	  //省略无关代码...
	}
  //省略无关代码...
	try {
    	//通过反射调用类的无参构造函数创建对象
	    S p = service.cast(c.newInstance());
	    providers.put(cn, p);
	    return p;
	} catch (Throwable x) {
    //省略无关代码...
	}
	throw new Error();
};
```

​		通过前面核心源码的分析，可以看到JDK原生的SPI机制是通过获取配置文件中接口实现类的全限定类名，然后通过反射调用无参数构造函数创建对象来实现服务定义与具体实现的隔离，但对于像Dubbo这像的RPC框架而言JDK原生的SPI机制，显得过于简单与粗暴。虽然SPI机制利用反射隐式地为我们创建对象，但却只能通过无参地构造方法创建对象。如果SPI能像Spring框架中那样具备依赖注入、AOP、按需加载实例是不是能更好地满足Dubbo框架将核心接口定义与具体实现分离的诉求（"微内核+插件" 的诉求）呢？实际Dubbo拓展的SPI机制便实现了依赖注入、AOP、按需加载实例的功能，我仿佛在Dubbo的SPI机制上看到了Spring的身影。

#### 2 Dubbo中SPI机制概述

​		有了上面对JDK原生SPI的一个整体认识后，再理解Dubbo中的SPI机制将相对容易很多。Dubbo的拓展的SPI机制核心要解决三件事，即**如何实现按需创建实例、如何实现依赖注入、如何实现AOP**，当然除了这三点外Dubbo的SPI机制还提供超强的自适应拓展机制。在没有分析Dubbo中的SPI机制前，大家可以思考一下如何去实现这三点。本节先整体看一下怎么使用Dubbo中SPI机制提供的各种能力，下一节通过源码深入分析具体实现。	

**注意：后面分析是基于dubbo-2.6.4** 

##### 2.1 Dubbo 中SPI基本用法     

 		Dubbo中的SPI机制同样是通过解析配置文件获取接口的具体实现类，然后通过反射完成对象创建与依赖注入，同时在创建对象的过程中会判断是否有Wrapper类的存在来实现AOP功能（本质上Dubbo中的SPI实现AOP是通过静态代理的）。与JDK的SPI机制一样同样可以将整个服务定义到最后调用服务，拆分为三大步。

> **1、定义服务**
>
> 定义服务的核心接口，并在服务接口上加上@SPI注解，如果服务要实现自适应拓展可在接口方法上加上@Adaptive注解或是在接口实现类上加@Adaptive注解（这实际属性提供服务部分）。
>
> **2、提供服务**
>
> 实现服务接口，并在META-INF/dubbo/internal/或META-INF/dubbo/或META-INF/services/目录下面创建名为服务接口名的配置文件。配置文件的内容为键值对，key为拓展实现类的键，值为拓展实现类的全限定类名。如果要实现自适应拓展功能可在类上加@Adaptive注解。下面是Dubbo中com.alibaba.dubbo.common.compiler.Compiler 接口实现类对应配置文件的具体内容。
>
> ```java
> adaptive=com.alibaba.dubbo.common.compiler.support.AdaptiveCompiler
> jdk=com.alibaba.dubbo.common.compiler.support.JdkCompiler
> javassist=com.alibaba.dubbo.common.compiler.support.JavassistCompiler
> ```
>
> **3、调用服务**
>
> 与JDK的SPI机制类似，服务实例的创建与调用都有一个核心类负责，Dubbo中该类为ExtensionLoader。通过ExtensionLoader#getExtensionLoader方法获取对应的ExtensionLoader实例，然后再通过Extension的名称调用ExtensionLoader#getExtension方法获取具体的接口实现类，最后调用接口定义的方法完成服务调用 。下面展示了如何通过ExtensionLoader获取Dubbo的Compiler实现类中JavassistCompiler实现，然后调用Compiler#compile方法将源代码编译为字节码并加载到JVM中生成对应的Class类实例。
>
> ```java
> ExtensionLoader<Compiler> loader = ExtensionLoader.getExtensionLoader(Compiler.class);
> //javassist为com.alibaba.dubbo.common.compiler.Compiler文件中JavassistCompiler实现的Key 
> Compiler compiler = loader.getExtension("javassist");
> compiler.compile("resource code ...", Thread.currentThread().getContextClassLoader());
> ```

实际上面调用服务的过程中是通过策略模式实现的，当首次传Extension的名称时ExtensionLoader才会去配置文件中找到相应的限制定类名，并将该类加载到JVM然后通过反射创建对象，创建完后会将创建好的实现缓存在全局的ConcurrentMap中，这个过程完成了实例的按需创建。

##### 2.2 Dubbo 中SPI自适应拓展用法

​		Spring中依赖注入与AOP是整个框架的核心精髓，同样Dubbo的SPI中的依赖注入与AOP也是实现"微内核+插件"的精髓。在讲如何使用Dubbo 中SPI自适应拓展时，先来讲一下自适应拓展，因为Dubbo在依赖注入时会用到自适应拓展。自适应拓展是什么？简单地理解就是在拓展接口方法被调用时，先调用一个代理类，然后再由代理类通过策略模式根据运行时传入的参数决定动态哪一个创建扩展实现并最终调用其方法。看起有点像普通的策略模式，通过参数不同然后定位到接口的某一具体实现，但实际上Dubbo的SPI的自适应拓展要更智能点。Dubbo首先会根据调用拓展接口时传入的参数为拓展接口生成具体有代理功能的代码，然后通过javassist 或 jdk 编译这段代码，并将对应的类加载到JVM，最后通过反射创建拓展接口的代理实例。而代理实例内部则采用了普通的策略模式，根据根据调用拓展接口时传入的参数去决定具体要调用哪一个拓展实现。

​		上面讲到Dubbo的SPI机制会根据调用拓展接口传入的参数决定使用哪一个具体拓展实现类。Dubbo为了形成统一的契约机制，定义了URL模型来完成整个框架中核心参数的传递与获取。URL模型与我们熟悉的URL（Uniform Resource Locators）很类似，这不细讲可以参数Dubbo 官网 [Dubbo 中的 URL 统一模型](https://dubbo.apache.org/zh/blog/2019/10/17/dubbo-%E4%B8%AD%E7%9A%84-url-%E7%BB%9F%E4%B8%80%E6%A8%A1%E5%9E%8B/)。在使用自适应拓展时，要接口声明的方法要包括类型为URL参数或者参数内有getURL方法用于获取URL，否则运行时将抛出异常。要从接口声明的方法的参数列表中获取URL，主要是为了通过URL去获取扩展类的键，也就是为了定位具体要调用哪个扩展实现。

​		自适应拓展使用上与Dubbo中SPI基本用法的区别主要是要在方法上加 @Adapitve注解，同时方法参数列表中要有URL类型参数或者能通过参数的getURL方法获取URL。@Adapitve注解后面分析源时会详细讲，@Adapitve可标注在方法或类上，标注方法表示Dubbo会自动为其生成自适应类（实际就是代理类），标注在类上表示自适应类要人工实现。Dubbbo内部仅有两个类被 Adaptive 注解，分别是 AdaptiveExtensionFactory 和AdaptiveCompiler。来通过一个例子对自适应拓展有更加直观的认识。

```java
package org.knight;
@SPI
public interface EngineMaker {
    @Adapitve
    Engine makeEngine(URL url);
}
```

Dubbo 会为EngineMaker接口生成具体代理功能的代码，下面是Dubbo为EngineMaker接口生成的代理类代码：

```java
package org.knight;
public class EngineMaker$Adaptive implements EngineMaker {
    public Wheel makeEngine(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
    	  // 1.从 URL 中获取 EngineMaker 名称
        String engineMakerName = url.getParameter("engine.maker");
        if (engineMakerName == null) {
            throw new IllegalArgumentException("engineMakerName == null");
        }
        // 2.通过 SPI 加载具体的 EngineMaker
        EngineMaker engineMaker = ExtensionLoader
    		.getExtensionLoader(EngineMaker.class).getExtension(engineMakerName);
        // 3.调用目标方法
        return engineMaker.makeEngine(url);
    }
}
```

EngineMaker$Adaptive 是一个代理类，EngineMaker$Adaptive将代理EngineMaker#makeEngine 方法，在makeEngine方法内部通过参数URL获取具体的EngineMaker的拓展实现的扩展名（Dubbo中SPI机制配置文件中的key），再通过SPI加载具体的EngineMaker实现类，最后调用实现类实例的makeEngine 方法。

##### 2.3 Dubbo 中SPI依赖注入与AOP用法

​		按照前面说的方式使用Dubbo中的SPI机制去调用具体的扩展实现时，Dubbo的SPI机制内部在加载创建实例后，会检测具体的拓展实现类内部所有的只有一个参数的public的setXXX方法，然后通过ExtensionFactory尝试获取对应的扩展实现并完成依赖注入。下面通过一个详细的例子来看一下，CarMaker的拓展类ElectricCarMaker内部依赖了EngineMaker，并暴露了setEngineMaker方法以便Dubbo注入依赖的EngineMaker。

```java 
//Engine接口
package org.knight;
public interface Engine {}
//FuelEngine实现
package org.knight;
public class FuelEngine implements Engine {
  public FuelEngine() {System.out.println("FuelEngine implementation!");}
}
//ElectricEngine实现
package org.knight;
public class ElectricEngine implements Engine {
  public ElectricEngine() { System.out.println("ElectricEngine implementation!");}
}
//Car接口
package org.knight;
public interface Car {
  void run();
}
//ElectricCar实现
package org.knight;
public class ElectricCar implements Car {
  private final Engine engine;
  public ElectricCar(Engine engine) {
    this.engine = engine;
    System.out.println("ElectricCar implementation!");
  }
  public void run() {
    System.out.println("ElectricCar run with " +  engine.getClass());
  }
}
//CarMaker接口
package org.knight;
import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.SPI;
@SPI
public interface CarMaker {
  Car makeCar(URL url);
}
//ElectricCarMaker实现
package org.knight;
import org.apache.dubbo.common.URL;
public class ElectricCarMaker implements CarMaker {
  public EngineMaker engineMaker;
  @Override
  public ElectricCar makeCar(URL url) {
    return new ElectricCar(engineMaker.makeEngine(url));
  }
  public void setEngineMaker(EngineMaker engineMaker) {
    System.out.println("Inject property engineMaker by Dubbo SPI!");
    this.engineMaker = engineMaker;
  }
}
//EngineMaker接口
import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.Adaptive;
import org.apache.dubbo.common.extension.SPI;
@SPI
public interface EngineMaker {
  //指URL模型要传入的参数名称为engineMakerType或engineType
  @Adaptive({"engineMakerType", "engineType"})
  Engine makeEngine(URL url);
}
//ElectricEngineMaker实现
package org.knight;
import org.apache.dubbo.common.URL;
public class ElectricEngineMaker implements EngineMaker {
  @Override
  public ElectricEngine makeEngine(URL url) {return new ElectricEngine();}
}
//FuelEngineMaker实现
package org.knight;
import org.apache.dubbo.common.URL;
public class FuelEngineMaker implements EngineMaker {
  @Override
  public FuelEngine makeEngine(URL url) {return new FuelEngine();}
}
```

META-INF/services下面有两个配置文件org.knight.CarMaker内容为

> electricCarMaker=org.knight.ElectricCarMaker

org.knight.EngineMaker内容为

> electricEngineMaker=org.knight.ElectricEngineMaker
> fuelEngineMaker=org.knight.FuelEngineMaker

执行操作的主函数代码如下

```java
//主函数
package org.knight;
import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.ExtensionLoader;
public class Main {
  public static void main(String args[]) {
    //1、获取对应ExtensionLoader实现
    ExtensionLoader<CarMaker> extensionLoader = ExtensionLoader.getExtensionLoader(CarMaker.class);
    //2、通过扩展类的名（配置文件中的key）, 加载并创建对应CarMaker扩展实现（ElectricEngineMaker）。
    CarMaker carMaker = extensionLoader.getExtension("electricCarMaker");
    //3、通过URL传递engineMakerType参数，engineMakerType参数的值为electricEngineMaker
    URL url = new URL("", "", 8080).addParameter("engineMakerType", "electricEngineMaker");
    //4、调用具体扩展实现类的方法（实际调用ElectricEngineMaker#makeCar方法）
    Car car = carMaker.makeCar(url);
    //5、执行Car#run方法
    car.run();
  }
}
```

上面的代码执行完后得出以下结果

> Inject property engineMaker by Dubbo SPI!
> ElectricEngine implementation!
> ElectricCar implementation!
> ElectricCar run with class org.knight.ElectricEngine

可以看到Dubbo为CarMaker的拓展类ElectricCarMaker注入了依赖的EngineMaker对象，并且实现注入的EngineMaker类型为ElectricEngineMaker而不是FuelEngineMaker。上面比较关键代码片段是

```java
@SPI
public interface EngineMaker {
  //指URL模型要传入的参数名称为engineMakerType或engineType
  @Adaptive({"engineMakerType", "engineType"})
  Engine makeEngine(URL url);
}
```

与

```
//3、通过URL传递engineMakerType参数，engineMakerType参数的值为electricEngineMaker
URL url = new URL("", "", 8080).addParameter("engineMakerType", "electricEngineMaker");
```

EngineMaker#makeEngine方法上的@Adaptive注解的值engineMakerType与engineType规定要通过参数engineMakerType或engineType去URL模型中获取EngineMaker自适应扩展的扩展名。如果URL模型中参数engineMakerType对应的值非空，则Dubbo优先从URL中获取该参数值作为扩展名；否则Dubbo再考虑从URL模型中获取参数engineType，URL模型中参数engineType对应的值非空，则Dubbo从URL中获取参数engineType值作为扩展名；最后Dubbo会采用默认的扩展名，也就是@SPI注解的值作为扩展名作为兜底的扩展名，当然提前是默认的扩展名要存在（上面的情况没有指定默认的扩展名，@SPI("electricEngineMaker") 代表将electricEngineMaker作为默认的扩展名）。当所有情况当不符合时，那么Dubbo就无法获取扩展名，也无法完成后面的依赖注入这时会直接抛出异常。Dubbo获取到EngineMaker对应的扩展名后，则会从META-INF/services/org.knight.EngineMaker配置文件中用扩展名去找到具体的全限定类名，并加载类到JVM同时通过反射创建对象，而后再将ElectricEngineMaker实例注入到ElectricCarMaker内。

​		那如果EngineMaker#makeEngine方法上的@Adaptive注解的值为空时Dubbo会如何处理呢？此时Dubbo会将EngineMaker按其内部规则转化为engine.maker作为URL中扩展名的参数，然后通过该参数即engine.maker去从RUL模型中获取值作为扩展名。再结合@Adaptive注解 value()上的注释体会一下上面的分析。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    String[] value() default {};
}
```

String[] value() default {};上面的注释如下

```java
Decide which target extension to be injected. The name of the target extension is decided by the parameter passed in the URL, and the parameter names are given by this method.
If the specified parameters are not found from URL, then the default extension will be used for dependency injection (specified in its interface's SPI).
For example, given String[] {"key1", "key2"}:
find parameter 'key1' in URL, use its value as the extension's name
try 'key2' for extension's name if 'key1' is not found (or its value is empty) in URL
use default extension if 'key2' doesn't exist either
otherwise, throw IllegalStateException
If the parameter names are empty, then a default parameter name is generated from interface's class name with the rule: divide classname from capital char into several parts, and separate the parts with dot '.', for example, for org.apache.dubbo.xxx.YyyInvokerWrapper, the generated name is String[] {"yyy.invoker.wrapper"}.
Returns:parameter names in URL
```

可以参照下面的流程图，仔细理解一下如何从URL模型中获取扩展名，然后用扩展名去加载类。实际整个流程的核心目标是确定URL模型中获取扩展名的参数名，即通过什么参数从URL模型中获取扩展名。

![](https://gitee.com/0909/blog/raw/master/img/20220113100031.png)

Dubbo中SPI机制的依赖注入如何使用已分析完了，下面看看该如何去使用Dubbo中SPI机制的AOP。Spring中的AOP本质上是代理，Dubbo中SPI机制的AOP本质上也是代理，只不过这个代理是静态的，即要先创建代理类然后才能在实际类执行前后作相应的增强。这里先创建EngineMaker的代理类也就是AOP执行类，代码如下。

```java
package org.knight;
import org.apache.dubbo.common.URL;
public class EngineMakerWrapper implements EngineMaker {
  private final EngineMaker engineMaker;
  //公共的构建函数参数声明时一定要有被增强的接口作为唯一的参数，这是Dubbo实现AOP的入口
  public EngineMakerWrapper(EngineMaker engineMaker) {
    this.engineMaker = engineMaker;
  }
  @Override
  public Engine makeEngine(URL url) {
    System.out.println("Before Advice by EngineMakerWrapper!");
    Engine engine = engineMaker.makeEngine(url);
    System.out.println("After Advice by EngineMakerWrapper!");
    return engine;
  }
}
```

AOP类EngineMakerWrapper代理了EngineMaker，并在执行对应makeEngine方法前后加上了简单的增强。现在只要在原来的META-INF/services/org.knight.EngineMaker配置文件中新增一行如下配置便可使AOP功能生效。

> engineMakerWrapper=org.knight.EngineMakerWrapper

然后再次执行之前的main函数得到如下的输出。

> Inject property engineMaker by Dubbo SPI!
> Before Advice by EngineMakerWrapper!
> ElectricEngine implementation!
> After Advice by EngineMakerWrapper!
> ElectricCar implementation!
> ElectricCar run with class org.knight.ElectricEngine

编写Dubbo中SPI机制的AOP对应的代理类时，一定要有公共的构造函数，且构建函数参数声明时一定要有被增强的接口作为唯一的参数，否则Dubbo无法实现AOP，同时在配置文件中要将该代理类加入。虽然在配置文件中通过键值对配置了AOP对应的代理类，但却不能像正常扩展类那样通过扩展名获取到对应的实现。

```java 
ExtensionLoader<EngineMaker> loader = ExtensionLoader.getExtensionLoader(EngineMaker.class);
/**
 * META-INF/services/org.knight.EngineMaker配置文件有对应的
 * engineMakerWrapper=org.knight.EngineMakerWrapper，但获取不到engineMaker，其值为null
 **/
EngineMaker engineMaker = loader.getExtension("engineMakerWrapper");
```

上面的代码执行后将抛出异常提示*No such extension org.knight.EngineMaker by name engineMakerWrapper*

#### 总结

​		本文从JDK原生的SPI机制开始由浅及深逐步地介绍了Dubbo中SPI机制，并通过示例展示了如何运用Dubbo中SPI机制的依赖注入、AOP、自适应拓展，本中重点分析了如何从URL模型与自适应拓展注解@Adaptive来确定要最终要获取的扩展类。下篇中将结合源码深入分析了依赖注入、AOP、自适应拓展是如何实现的。希望看完本文你会有所收获，限于本人能力有限文中不正确还望指正。最后欢迎关注个人公众号洞悉源码。

#### 参考

 [Dubbo 中的 URL 统一模型](https://dubbo.apache.org/zh/blog/2019/10/17/dubbo-%E4%B8%AD%E7%9A%84-url-%E7%BB%9F%E4%B8%80%E6%A8%A1%E5%9E%8B/)

[Dubbo SPI](https://dubbo.apache.org/zh/docsv2.7/dev/source/dubbo-spi/)

[SPI 自适应拓展](https://dubbo.apache.org/zh/docsv2.7/dev/source/adaptive-extension/)

#### 结尾

​		原创不易，点赞、在看、转发是对我莫大的鼓励，关注公众号洞悉源码是对我最大的支持。同时相信我会分享更多干货，我同你一起成长，我同你一起进步。
