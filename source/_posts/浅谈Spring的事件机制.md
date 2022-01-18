---
title: 浅谈Spring的事件机制
date: 2021-10-25 20:20:19
tags:
  - Spring 
  - ApplicationContext
  - 观察者模式
categories: Spring
---

### 浅谈Spring的事件机制

#### 1 java中的事件机制

​		java 在JDK中提供了定义事件机制的两个重要类，分别是EventObject与EventListener。事件机制与设计模型中的观察者模式类似。事件机制通常有三个参与者，分别是Event、EventPublisher、EventListener，EventPublisher负责发布Event，EventPublisher具体实现内部持有了多个EventListener，当Event发布后EventPublisher会通知所有的观察者。但察者模式通常只有观察者（Observer）与被观察者（Subject）两者参与，被观察者具体实现内部持有了多个观察者，同时被观察者提供注册观察者与取消观察者的入口，当被观察者状态发生改变时，其会通知所有的观察者。可以看到事件机制中的Event相当于观察者模式中Subject的状态，而EventPublisher相当于观察者（Subject），EventListener相当于观察者（Observer）。本质上事件机制与察者模式是一样的。

​		那事件机制有什么好处呢？来看一个简单的例子，以用户注册为例，当用户注册完后，会给用户发送对应的短信通知，同时也会发送邮件通知。后来产品说要搞积分体系，新注册的用户还得增加积分。那这个时候用户注册的那段代码，就得改一下加上给用户增加积分的逻辑，这样就违反了设计原则中的开闭原则（对扩展开放，对修改关闭）。有了事件机制后，我们可以怎么做呢？来看看下面的例子。

```java
package org.knight;
import java.util.EventObject;
//用户注册Event
public class RegisterEvent extends EventObject {
  public RegisterEvent(String payload) {
    super(payload);
  }
}
//用户注册Event监听器
package org.knight;
import java.util.EventListener;
public interface UserRegisterListener extends EventListener {
  //自定义监听事件的方法
  void onRegisterEvent(RegisterEvent event);
}
//用户注册Event发布者
package org.knight;
import java.util.HashSet;
import java.util.Set;
public class UserRegisterPublisher {
  //持有多个Event监听器
  private final Set<UserRegisterListener> listeners = new HashSet<>();
  //增加Event监听器
  public boolean addListener(UserRegisterListener listener) {
    return listeners.add(listener);
  }
  //删除Event监听器
  public boolean removeListener(UserRegisterListener listener) {
    return listeners.remove(listener);
  }
  //发布事件
  public void publishEvent(RegisterEvent event) {
    for (UserRegisterListener listener : listeners) {
      listener.onRegisterEvent(event);
    }
  }
}
//发送短信
package org.knight;
public class ToSendMessageListener implements UserRegisterListener {
  @Override
  public void onRegisterEvent(RegisterEvent event) {
    System.out.println("User " + event.getSource() + " register, send message!");
  }
}
//发送邮件
package org.knight;
public class ToSendMailListener implements UserRegisterListener {
  @Override
  public void onRegisterEvent(RegisterEvent event) {
    System.out.println("User " + event.getSource() + " register, send email!");
  }
}
//用户注册服务
package org.knight;
public class UserRegisterService {
  private final UserRegisterPublisher publisher = new UserRegisterPublisher();
  public boolean userRegister(String username) {
    System.out.println("用户"+username+"注册成功");
    publisher.publishEvent(new RegisterEvent(username));
    return true;
  }
  //注册Event监听器
  public boolean regiserListener(UserRegisterListener listener) {
    return publisher.add(listener);
  }
  //删除Event监听器
  public boolean removeListener(UserRegisterListener listener) {
    return publisher.add(listener);
  }
}
//主函数
package org.knight;
public class MainUserCenter {
  public static void main(String args[]) {
    UserRegisterService registerService = new UserRegisterService();
    //注册相应的listener
    registerService.regiserListener(new ToSendMessageListener());
    registerService.regiserListener(new ToSendMailListener());
    registerService.userRegister("叶易");
  }
}
```

上面定义了用户注册事件RegisterEvent，RegisterEvent扩展了JDK中的Event类；UserRegisterListener负责监听RegisterEvent，其扩展了JDK中的EventListener接口，同时定义了听事件的onRegisterEvent方法，其有两个具体实现分别负责用户注册后发送信息与发送邮件，分别是ToSendMessageListener与ToSendMailListener；UserRegisterPublisher为事件发布类，内部持有多个UserRegisterListener，同时定了addListener与removeListener用以维护内部持有的UserRegisterListener，当然其还定义了事件发布方法publishEvent。上面代码的MainUserCenter#main执行后结果如下。

> 用户叶易注册成功
> User 叶易 register, send email!
> User 叶易 register, send message!

上面提到要新增增加用户积分的功能，有了事件监听机制后就可以实现对扩展开放对修改关闭。只要新增一个用户注册后增加积分的监听器，然后在MainUserCenter中将其注册一下便可以。当然用户注册这个场景，真实业务上不会不会这么做，而是通过MQ发送用户注册的消息，然后需要的业务服务再监听对应主题下的这个消息执行业务逻辑。

#### 2 Spring中的事件机制

##### 2.1 Spring中的事件机制使用

​	Spring定义了ApplicationEvent、ApplicationEventListener、ApplicationEventPublisher来支撑其内部的事件机制。同时还提供了EventListener注解，方便将方法对应类直接转换为监听器。

```java 
//事件定义，扩展了JDK中的EventObject，增加事件发生时间
public abstract class ApplicationEvent extends EventObject {
	/** System time when the event happened. */
	private final long timestamp;
	public ApplicationEvent(Object source) {
		super(source);
		this.timestamp = System.currentTimeMillis();
	}
	public final long getTimestamp() {
		return this.timestamp;
	}
}
//事件监听器定义，扩展了JDK中的EventListener，增加自定义的onApplicationEvent方法
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
	void onApplicationEvent(E event);
}
//事件发布者定义，Spring中典型的事件发布者是ApplicationContext
@FunctionalInterface
public interface ApplicationEventPublisher {
  //该方法会发布事件后会通知所有注册事件监听器
	default void publishEvent(ApplicationEvent event) {
		publishEvent((Object) event);
	}
	//该方法会发布事件后会通知所有注册事件监听器，事件监听器内部具体执行时可是同步或是异步，原则上越快越好
	void publishEvent(Object event);
}
```

Spring中和事件相关的另外一个重要的接口是ApplicationEventMulticaster，其内部持有多个ApplicationListener，并定义注册ApplicationListener与删除ApplicationListener的方法。Spring中的ApplicationEventPublisher发布事件将由其代理完成。来看一下将上面的介绍的用户注册的例子改为Spring中的事件后具体的代码。

```java
package org.knight.spring;
import org.springframework.context.ApplicationEvent;
//用户注册事件，扩展了Spring的ApplicationEvent
public class RegisterEvent extends ApplicationEvent {
  public RegisterEvent(String payload) {
    super(payload);
  }
}
//发送短信
package org.knight.spring;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;
//加上Component注解，让Spring能将该ApplicationListener注册
@Component
public class ToSendMessageListener implements ApplicationListener<RegisterEvent> {
  @Override
  public void onApplicationEvent(RegisterEvent event) {
    System.out.println("User " + event.getSource() + " register, send message!");
  }
}
//发送邮件
package org.knight.spring;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;
//加上Component注解，让Spring能将该ApplicationListener注册
@Component
public class ToSendMailListener implements ApplicationListener<RegisterEvent> {
  @Override
  public void onApplicationEvent(RegisterEvent event) {
    System.out.println("User " + event.getSource() + " register, send email!");
  }
}
//用度注册服务
package org.knight.spring;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.context.ApplicationEventPublisherAware;
import org.springframework.stereotype.Service;
@Service
public class UserRegisterService implements ApplicationContextAware, ApplicationEventPublisherAware {
  private ApplicationContext applicationContext;
  private ApplicationEventPublisher applicationEventPublisher;
  public boolean userRegister(String username) {
    System.out.println("用户"+username+"注册成功");
    applicationEventPublisher.publishEvent(new RegisterEvent(username));
    //也可以用applicationContext发送事件
    //applicationContext.publishEvent(new RegisterEvent(username));
    return true;
  }
  @Override
  public void setApplicationContext(ApplicationContext applicationContext) {
    this.applicationContext = applicationContext;
  }
  @Override
  public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher){
    this.applicationEventPublisher = applicationEventPublisher;
  }
}
//主函数
package org.knight.spring;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
public class MainUserCenter {
  public static void main(String[] args) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    //扫描对应包
    context.scan("org.knight.spring");
    context.refresh();
    //获取用户注册服务
    UserRegisterService userRegisterService = context.getBean(UserRegisterService.class);
    userRegisterService.userRegister("叶易");
  }
}
```

可以看到能很方便地利用Spring的事件扩展机制，只要定义对应事件RegisterEvent扩展ApplicationEvent，然后再定义对应的ApplicationListener就行，自定义的ApplicationListener一定要加上@Component、@Service之类的注解以便Spring能将ApplicationListener注册。上面的UserRegisterService实现了ApplicationContextAware与ApplicationEventPublisherAware，以便Spring分别能将ApplicationContext与ApplicationEventPublisher注入到UserRegisterService中，在发布事件时可以采用ApplicationContext或ApplicationEventPublisher。ApplicationContext接口扩展了ApplicationEventPublisher接口，同样具体事件发布的能力，这主要是为支撑容器启动、容器刷新、容器暂停时发布对应的事件。提前提到过可以通过EventListener注解，定义监听器类，同时通知监听器去执行具体的业务逻辑时可以采用同步或异步的方式。

##### 2.2 ApplicationContext如何异步执行事件

​		来简要地看一下Srping事件机制中重要的实现ApplicationContext。Spring源码中ApplicationContext扩展了ApplicationEventPublisher接口，Spring给ApplicationContext定义其必须具备事件发布的能力。

> The ability to publish events to registered listeners. Inherited from the ApplicationEventPublisher interface.

在ApplicationContext的子类AbstractApplicationContext中可以看到具体事件发布能力实现的代码。其中的成员变量 applicationEventMulticaster，代理了ApplicationEventPublisher 完成了事件发布。

```java 
//Helper class used in event publishing
ApplicationEventMulticaster applicationEventMulticaster
```

AbstractApplicationContext中发布事件的方法最终调用如下方法。

```java 

protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
		// 省略部分代码
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		} else {
			applicationEvent = new PayloadApplicationEvent<>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
			}
		}
		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		} else {
      //applicationEventMulticaster代理完成事件发布
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}
		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			} else {
				this.parent.publishEvent(event);
			}
		}
	}
	ApplicationEventMulticaster getApplicationEventMulticaster() throws IllegalStateException {
		if (this.applicationEventMulticaster == null) {
			throw new IllegalStateException("ApplicationEventMulticaster not initialized - " +
					"call 'refresh' before multicasting events via the context: " + this);
		}
		return this.applicationEventMulticaster;
	}
```

从上面源可以看到ApplicationEventMulticaster#multicastEvent方法代理了ApplicationEventPublisher 完成了事件发布。

```java
public interface ApplicationEventMulticaster {
	void addApplicationListener(ApplicationListener<?> listener);
	void addApplicationListenerBean(String listenerBeanName);
	void removeApplicationListener(ApplicationListener<?> listener);
	void removeApplicationListenerBean(String listenerBeanName);
	void removeAllListeners();
	void multicastEvent(ApplicationEvent event);
	void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);
}
```

ApplicationEventMulticaster接口定义增加与删除ApplicationListener的相关方法，同时提供了事件发布的multicastEvent方法。其有抽象类AbstractApplicationEventMulticaster与具体实现SimpleApplicationEventMulticaster。AbstractApplicationEventMulticaster只提供了维护与检索ApplicationListener的相关方法，而发布事件的multicastEvent方法的具体实现则在具体实现类SimpleApplicationEventMulticaster上完成，该类实现代码也相对简明。

```
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
	//真正执行listener中任务的Executor，实现异步通知listener处理的关键
	private Executor taskExecutor;
	public void setTaskExecutor(@Nullable Executor taskExecutor) {
		this.taskExecutor = taskExecutor;
	}
	protected Executor getTaskExecutor() {
		return this.taskExecutor;
	}
	//省略部分源码...
}
```

SimpleApplicationEventMulticaster中的Executor是实现异步通知listener处理的关键。当taskExecutor不为空时，会采用Executor对应的线程池异步通知listener执行任务，否则同步遍历所有listener并通知listener执行任务。

```
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
  ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
  //获取具体的执行者
  Executor executor = getTaskExecutor();
  for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
  if (executor != null) {
  	executor.execute(() -> invokeListener(listener, event));
  } else {
  	invokeListener(listener, event);
  }
 }
}
```

#### 总结

​		本文从结合具体例子与代码从JDK中的事件机制谈起，逐步介绍了如何使用Spring中的事件机制，同时简要地分析了ApplicationContext中事件异步执行的相关源码。本文源码分析并不深入，感兴趣的读者可以结合上面的介绍深入分析Spring中的事件机制。

#### 结尾

​		原创不易，点赞、在看、转发是对我莫大的鼓励，关注公众号洞悉源码是对我最大的支持。同时相信我会分享更多干货，我同你一起成长，我同你一起进步。
