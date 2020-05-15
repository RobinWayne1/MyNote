# Spring的观察者模式——事件广播机制

## 一、简单使用

事件广播机制主要涉及三个接口——`ApplicationEventPublisher `、`ApplicationEvent`和`ApplicationListener<>`。宏观来说广播的大致流程就是:`ApplicationEvent`作为一个事件对象,通过`ApplicationEventPublisher `发布这个事件,而作为观察者的`ApplicationListener`将会专门负责接收相应的事件,以进行处理。

### 1、具体例子

ApplicationEvent的使用:

```java
public class UserRegisterEvent extends ApplicationEvent{
    public UserRegisterEvent(String name) { //name即source
        super(name);
    }
}
```

其中父类构造方法是`public ApplicationEvent(Object source)`,source将会被存放到其父类java内置的`EventObject`中,可通过`getSource()`获得

ApplicationEventPublisher的使用

```java
@Service
public class UserService implements ApplicationEventPublisherAware {
    public void register(String name) {
        System.out.println("用户：" + name + " 已注册！");
        applicationEventPublisher.publishEvent(new UserRegisterEvent(name));
    }
    private ApplicationEventPublisher applicationEventPublisher;
    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }
}
```

在用户代码中使用publisher时需要通过`Aware`接口获得`ApplicationEventPublisher`对象,调用该对象的`publishEvent()`方法将事件对象发布出去

ApplicationListener的使用:

```java
@Service
public class EmailService implements ApplicationListener<UserRegisterEvent> {
    @Override
    public void onApplicationEvent(UserRegisterEvent userRegisterEvent) {
        System.out.println("邮件服务接到通知，给 " + userRegisterEvent.getSource() + " 发送邮件...");
    }
}
```

监听器通过接口声明的泛型类型从而只获取相应的事件对象,在`onApplicationEvent()`方法中进行相应的逻辑处理.**==要注意,监听器和发布者都必须声明为bean添加进ioc容器中,发布者需要通过Aware接口或使用`@Autowired`获得ApplicationContextListener,而实现`ApplicationListener`的监听器将会在后处理器`ApplicationListenerDectecor`中将监听器bean添加到广播器中;而使用`@EvenListener`标注的方法的bean将会在实例化完bean之后(`getBean()`后),被`SmartInitializingSingleton`的实现类包装成`ApplicationListenerAdapter`然后将其加入广播器其中==**

## 二、源码分析

### 1、调用逻辑

我们以上面的例子开始看起:

```java
@Service
public class UserService implements ApplicationEventPublisherAware {
    
    public void register(String name) {
        System.out.println("用户：" + name + " 已注册！");
        applicationEventPublisher.publishEvent(new UserRegisterEvent(name));
    }
    
    private ApplicationEventPublisher applicationEventPublisher;
    
    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }
}
```

`ApplicationContext`继承了`ApplicationEventPublisher`接口,并由其实现类`AbstractApplicationContext`实现了`publishEvent()`方法,所以通过`Aware`接口得到的publisher其实就是`ClassPathXmlApplicationContext`。

接下来看`publishEvent()`内部逻辑:

```java
//AbstractApplicationContext.java
protected void publishEvent(Object event, ResolvableType eventType) {
   Assert.notNull(event, "Event must not be null");
   if (logger.isTraceEnabled()) {
      logger.trace("Publishing event in " + getDisplayName() + ": " + event);
   }

   //下面的判断语句是用来判断发布的事件是否是ApplicationEvent,若不是则用PayloadApplicationEvent(ApplicationEvent子类)将event包装一下
   ApplicationEvent applicationEvent;
   if (event instanceof ApplicationEvent) {
      applicationEvent = (ApplicationEvent) event;
   }
   else {
      applicationEvent = new PayloadApplicationEvent<Object>(this, event);
      if (eventType == null) {
         eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
      }
   }

   // Multicast right now if possible - or lazily once the multicaster is initialized
   if (this.earlyApplicationEvents != null) {
      this.earlyApplicationEvents.add(applicationEvent);
   }
   else {
       //多播逻辑入口.获取事件多播器,将事件发布出去
      getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
   }

   // Publish event via parent context as well...
   if (this.parent != null) {
      if (this.parent instanceof AbstractApplicationContext) {
         ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
      }
      else {
         this.parent.publishEvent(event);
      }
   }
}
```

事件广播器将事件进行广播:

```java
//SimpleApplicationEventMulticaster.java
public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
   ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    //根据事件的类型，获取广播器内的相应的监听器，并逐个调用
   for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
       //还可以是异步调用，用户可以通过Aware获取ApplicationContext，进而调用getApplicationEventMulticaster()获得广播器,从而setTaskExecutor()使得广播的发布过程变成异步(好像可以直接使用@Async进行异步调用?没有了解过)
      Executor executor = getTaskExecutor();
      if (executor != null) {
         executor.execute(new Runnable() {
            @Override
            public void run() {
               invokeListener(listener, event);
            }
         });
      }
      else {
          //调用逻辑
         invokeListener(listener, event);
      }
   }
}
```

```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
    //监听器调用报错则会使用ErrorHandler处理,set()的方式和线程池相同,不再赘述
   ErrorHandler errorHandler = getErrorHandler();
   if (errorHandler != null) {
      try {
         doInvokeListener(listener, event);
      }
      catch (Throwable err) {
         errorHandler.handleError(err);
      }
   }
   else {
      doInvokeListener(listener, event);
   }
}
```

最终调用逻辑:

```java
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
   try {
       //直接调用onApplicationEvent()方法发布事件
      listener.onApplicationEvent(event);
   }
   catch (ClassCastException ex) {
      String msg = ex.getMessage();
      if (msg == null || matchesClassCastMessage(msg, event.getClass().getName())) {
         // Possibly a lambda-defined listener which we could not resolve the generic event type for
         // -> let's suppress the exception and just log a debug message.
         Log logger = LogFactory.getLog(getClass());
         if (logger.isDebugEnabled()) {
            logger.debug("Non-matching event type for listener: " + listener, ex);
         }
      }
      else {
         throw ex;
      }
   }
}
```

监听器有两种实现方式:

* 一种是bean直接实现`ApplicationListener`,在最终调用时则直接调用bean的实现方法 

* 第二种是使用`@EventListener`注解标记方法,在添加==(这里的添加是在容器刷新的末阶段`finishBeanFactoryInitialization()`初始化完非延迟加载的bean(`getBean()`后)后,调用`SmartInitializingSingleton`接口的实现类`EventListenerMethodProcessor`的`afterSingletonsInstantiated`方法,此方法会创建一个监听适配器并将其加入进广播器中留个疑问,看到再说)==这种监听器时会将注解标注的方法作为bridgeMethod,用`ApplicationListener`的实现类`ApplicationListenerAdapter`将桥方法包装起来,进行发布调用的时候则会反射调用监听适配器里的桥方法完成发布

  使用注解方式的listener更加灵活,如下

  ```java
  //ApplicationListenerAdapter.java
  onApplicationEvent(event)→
  public void processEvent(ApplicationEvent event) {
  		Object[] args = resolveArguments(event);
  		if (shouldHandle(event, args)) {
              //利用反射调用方法，还扩展了返回值
  			Object result = doInvoke(args);
  			if (result != null) {
                  //处理返回值
  				handleResult(result);
  			}
  			else {
  				logger.trace("No result object given - no result to handle");
  			}
  		}
	}
  ```
  
  处理逻辑：
  
  ```java
  protected void handleResult(Object result) {
     if (result.getClass().isArray()) {
        Object[] events = ObjectUtils.toObjectArray(result);
        for (Object event : events) {
           publishEvent(event);
        }
     }
     else if (result instanceof Collection<?>) {
        Collection<?> events = (Collection<?>) result;
        for (Object event : events) {
           publishEvent(event);
        }
     }
     else {
        publishEvent(result);
     }
  }
  ```
  
  从对返回值的处理可以看出，桥方法的可以返回任何类型的数据，并且`handleResult()`会将这些数据以事件的形式重新发布出去,变相使得一个监听器同时也可以是一个发布者,且不需要用aware获得`ApplicationEventPublisher`,直接将事件返回就可以进行发布了

### 2、广播器与监听器的初始化

#### ①广播器

`SimpleApplicationEventMulticaster`将会在刷新容器时的`initApplicationEventMulticaster()`中被创建

```java
protected void initApplicationEventMulticaster() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
      this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
      if (logger.isDebugEnabled()) {
         logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
      }
   }
   else {
       //通常会来到这里创建对象
      this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
      beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
      if (logger.isDebugEnabled()) {
         logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
               APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
               "': using default [" + this.applicationEventMulticaster + "]");
      }
   }
}
```

#### ②监听器

监听器将会在`registerListeners()`中向之前创建的广播器添加进listener的beanName,其中获得listener的方法通过`ListableBeanFactory`的`getBeanNamesForType()`获取beanName

```java
protected void registerListeners() {
   // Register statically specified listeners first.
   for (ApplicationListener<?> listener : getApplicationListeners()) {
      getApplicationEventMulticaster().addApplicationListener(listener);
   }

 
   String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
   for (String listenerBeanName : listenerBeanNames) {
      getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
   }

   // Publish early application events now that we finally have a multicaster...
   Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
   this.earlyApplicationEvents = null;
   if (earlyEventsToProcess != null) {
      for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
         getApplicationEventMulticaster().multicastEvent(earlyEvent);
      }
   }
}
```

要注意,上面的`registerListener()`只是向广播器中的`ListenerRetriever`对象(存放listener实例和名字的对象)添加listenerName的beanName,向广播器添加监听器的逻辑(实现`ApplicationListener`的bean)实则是在后处理器`ApplicationListenerDectector`中

```java
public Object postProcessAfterInitialization(Object bean, String beanName) {
		if (this.applicationContext != null && bean instanceof ApplicationListener) {
			// potentially not detected as a listener by getBeanNamesForType retrieval
			Boolean flag = this.singletonNames.get(beanName);
			if (Boolean.TRUE.equals(flag)) {
				//将当前bean添加到广播器中
				this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
			}
			else if (Boolean.FALSE.equals(flag)) {
				if (logger.isWarnEnabled() && !this.applicationContext.containsBean(beanName)) {
					// inner bean with other scope - can't reliably process events
					logger.warn("Inner bean '" + beanName + "' implements ApplicationListener interface " +
							"but is not reachable for event multicasting by its containing ApplicationContext " +
							"because it does not have singleton scope. Only top-level listener beans are allowed " +
							"to be of non-singleton scope.");
				}
				this.singletonNames.remove(beanName);
			}
		}
		return bean;
	}
```

对于含有`@EventListener`标注的方法的bean，将会在所有非懒加载的bean实例化完之后，被实现了`SmartInitializingSingleton`的`EventListenerMethodProcessor`类回调,然后真正将桥方法包装成监听适配器加入进广播器.其中`SmartInitializingSingleton`的作用就相当于`finishRefresh()`中的`ContextRefreshedEvent()`事件的监听器.(测试之后确实如此，这部分源码我还没有仔细阅读，有时间补上)

