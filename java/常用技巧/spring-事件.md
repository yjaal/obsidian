
```toc

```


## 使用场景

在一些业务场景中，当容器初始化完成之后，需要处理一些操作，比如一些数据的加载、初始化缓存、特定任务的注册等等。这个时候我们就可以使用 `Spring` 提供的 `ApplicationListener` 来进行操作。这和 `InitializingBean` 是不同的，它是在 Bean 实例化完成之后做一些初始化的操作。

## 使用

Spring 对事件机制也提供了支持，一个事件被发布后，被对应的监听器监听到，执行对应方法。并且 `Spring` 内已经提供了许多事件，`ApplicationEvent` 可以说是 `Spring` 事件的顶级父类。`ApplicationListener` 是监听器的顶级接口，事件被触发后，`onApplicationEvent` 方法就会执行

如果我们要写监听器，就要写这个监听器接口的实现类，`ApplicationEvent` 泛型就是我们要监听的类，所以我们要监听或者是发布，都是 `ApplicationEvent` 及其下面的子事件，通过查看 `ApplicationEvent` 类，我们发现有以下子事件：

1.  `ContextClosedEvent`：关闭容器发布这个事件
2.  `ContextRefreshedEvent`：容器刷新完成发布这个事件（所有 `bean` 都进行了实例化，完成了创建）
3.  `ContextStoppedEvent`：容器停止时发布这个事件
4.  `ContextStartedEvent`：容器开始执行时发布这个事件


```java
// 启动测试类 
@Test 
public void TestMain (){ 
	AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext (AppConfig. class); 
	// 自己发布一个事件 
	applicationContext.publishEvent(new ApplicationEvent ("自己发布的事件") { }); 
	applicationContext. close (); 
} 
// ApplicationListener 实现类 
@Component 
public class MyApplicationListener implements ApplicationListener<ApplicationEvent> { 
	// 当容器中发布此事件后，该方法会触发 
	public void onApplicationEvent(ApplicationEvent applicationEvent) { 
		System. out. println ("收到的事件：" + applicationEvent); 
	} 
} 
// 配置类 
@Configuration 
@ComponentScan ("listener") 
public class AppConfig { }
```

  其实就是实现 `ApplicationListener` 接口就可以监听相关事件，我们可以实现自己的事件，实现 `ApplicationEvent` 即可。输出结果如下，以下三点说一下：

-   容器启动时，会执行容器刷新完成事件，也就是 `ContextRefreshedEvent`
-   容器关闭时，会执行容器关闭事件，也就是 `ContextClosedEvent`
-   在启动类中，通过 `publishEvent` 来发布事件，执行这个方法的时候，`ApplicationListener` 就能监听到这个事件，就会回调 `onApplicationEvent` 执行

具体应用的时候可以直接注入发布器

```java
@Autowired
private ApplicationEventPublisher p;
```

同时，出了实现 `ApplicationListener` 接口外，还可以使用注解方式

```java
@Component 
public class MyApplicationListener { 
	// 当容器中发布此事件后，该方法会触发
	@EventListener 
	public void onApplicationEvent(ApplicationEvent applicationEvent) { 
		System. out. println ("收到的事件：" + applicationEvent); 
	} 
} 
```

## 实现分析

在上面的案例中，收到了三个事件，分别是：`ContextRefreshedEvent`、`ContextClosedEvent` 以及自己定义的 `MainTest$1[source=自己发布的事件]`，这几个事件在底层是如何收到的呢？

### 事件发布

最先收到的是 `ContextRefreshdEvent` 事件，注意：想要发布时间只需要实现 `ApplicationEventPublisher` 接口即可，当然本身 `spring` 中已经有很多类就实现了此接口，如下实现

```java
// AbstractApplicationContext.java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {  
   Assert.notNull(event, "Event must not be null");  
   if (logger.isTraceEnabled()) {  
      logger.trace("Publishing event in " + getDisplayName() + ": " + event);  
   }  
   // Decorate event as an ApplicationEvent if necessary  
   ApplicationEvent applicationEvent;  
   if (event instanceof ApplicationEvent) {  
      applicationEvent = (ApplicationEvent) event;  
   }   else {  
      applicationEvent = new PayloadApplicationEvent<>(this, event);  
      if (eventType == null) {  
         eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();  
      }   }  
   // Multicast right now if possible - or lazily once the multicaster is initialized 
   // 这里其实是一个在下面多播器初始化之前发布的事件
   if (this.earlyApplicationEvents != null) {  
      this.earlyApplicationEvents.add(applicationEvent);  
   }   else {  
      // 获取多播器进行事件发布
getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);  
   }  
   // Publish event via parent context as well...  
   if (this.parent != null) {  
      if (this.parent instanceof AbstractApplicationContext) {  
         ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);  
      }      else {  
         this.parent.publishEvent(event);  
      }   }}
```

主要逻辑就是获取多播器进行事件发布

```java
// SimpleApplicationEventMulticaster.java
@Override  
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {  
   ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));  
   for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {  
      Executor executor = getTaskExecutor();  
      if (executor != null) {  
         executor.execute(() -> invokeListener(listener, event));  
      }  
      else {  
         invokeListener(listener, event);  
      }  
   }  
}
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {  
   try {  
      listener.onApplicationEvent(event);  
   }   catch (ClassCastException ex) {  
      // ...
   }  
}
```

这里其实就是便利所有的监听器，然后如果本身自带 `Executor`，那就异步执行，否则就同步执行。具体执行就是调用监听器的 `onApplicationEvent` 方法。这里注意，监听器本身也是 `bean`，在容器启动 ` AbstractApplicationContext #refresh #registerListener ` 中会进行注册为监听器。














