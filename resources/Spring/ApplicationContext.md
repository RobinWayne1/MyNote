# ApplicationContext

## 一、类结构

`ApplicationContext`是对`BeanFactory`的扩展,先来看看这两者的类结构之中的联系

<img src="E:\Typora\resources\Spring\BeanFactory结构.png" style="zoom:50%;" />

1. `BeanFactory`接口的功能主要是对单个bean的获取操作,如接口方法`getBean(String name)`

2. 继承自`BeanFactory`的`ListableBeanFactory`则在其父接口提供的对单个bean获取操作的基础上增加 对多个bean的获取方法,如

   ```java
//根据类类型来获取容器中的bean
<T> Map<String, T> getBeansOfType(Class<T> type);
//扫描在容器中的bean,返回那些 被参数指定的注解类型 修饰的bean,当要得到某些特定bean时可以通过在bean上自定义注解,然后通过这个方法得到它们
Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType)
   ```
   
3. 继承自`BeanFactory`的`HierarchicalBeanFactory`,`hierarchical`意思是按等级划分的,该接口的作用就是提供父子容器的层次关系,其具体方法为

   ```java
//得到父容器
BeanFactory getParentBeanFactory();
   ```

   其会与`ApplicationContext`中的`getParent()`接口方法与`ConfigurableApplicationContext`的`setParent()`接口方法配合使用

4. 继承自`BeanFactory`的`AutowireCapabaleBeanFactory`,具有众多极重要方法的一个接口,提供了自动装配的功能.极其重要的接口方法如下

   ```java
   //doCreateBean()的入口,在此处将会处理循环依赖、反射创建bean(在其中会先选择构造器并进行构造器注入,具体看bean加载.md)、调用populateBean()进行注解属性注入
   Object createBean(Class<?> beanClass, int autowireMode, boolean dependencyCheck);
   //初始化bean,在此处会调用invokeAwareMethod()以给bean访问容器的机会,还会应用后处理器,如在后处理器中将bean包装成代理
      Object initializeBean(Object existingBean, String beanName)
      //应用后处理器
      Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
   ```
   
   **虽然`ApplicationContext`没有继承此类,但是它会以组合的方式使用该接口**
   
   **`AutowireCapableBeanFactory getAutowireCapableBeanFactory()`，注意这个方法能够与`ApplicationContextAware`组合使用.**
   
   ![](E:\Typora\resources\Spring\ApplicationContext结构.png)
   
   1. `AbstractXmlApplicationContext`提供了对xml配置文件的解析,其中的`loadBeanDefinition()`会将xml的信息转储到BeanDefinition对象中.我觉得解析其实没什么好了解的,相关知识的整理在第二\三章中
   
## 二、？？父子容器的概念

1. 为什么要有父子容器？
   ==父子容器的作用主要是划分框架边界。==
   在J2EE三层架构中，在service层我们一般使用spring框架， 而在web层则有多种选择，如spring mvc、struts等。因此，通常对于web层我们会使用单独的配置文件。例如在上面的案例中，一开始我们使用spring-servlet.xml来配置web层，使用applicationContext.xml来配置service、dao层。如果现在我们想把web层从spring mvc替换成struts，那么只需要将spring-servlet.xml替换成Struts的配置文件struts.xml即可，而applicationContext.xml不需要改变。

2. 为什么不能在Spring的applicationContext.xml中配置全局扫描？
   如果都在spring容器中，这时的SpringMVC容器中没有对象，所以加载处理器，适配器的时候就会找不到映射对象，映射关系，因此在页面上就会出现404的错误。
   因为在解析@ReqestMapping解析过程中，initHandlerMethods()函数只是对Spring MVC 容器中的bean进行处理的，并没有去查找父容器的bean。因此不会对父容器中含有@RequestMapping注解的函数进行处理，更不会生成相应的handler。所以当请求过来时找不到处理的handler，导致404。

3. 如果不用Spring容器，直接把所有层放入SpringMVC容器的配置spring-servlet.xml中可不可以？
   如果把所有层放入SpringMVC的。但是事务和AOP的相关功能不能使用。

4. 同时扫描？
   会在两个父子IOC容器中生成大量的相同bean，这就会造成内存资源的浪费。

## 三、`refresh()`

   若是讲ApplicationContext，毫无疑问就要讲`AbstractApplicationContext`实现自`ConfigurableApplicationContext`的`refresh()`方法


```java
//抽象模板方法
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
        
        prepareRefresh();

		//调用loadBeanDefinition()方法,将xml配置的信息转储成BeanDefinition对象,将这些对象存放进BeanDefinitionMap(name→definition),解析请看第二/三章
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		 // 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
		prepareBeanFactory(beanFactory);

		try {
			//在AbstractApplicationContext中是一个空方法,此方法是给子类一个修改beanFactory的机会.此时的beanDefinitionMap已经全部解析完成,子类context可以对这部分配置信息进行修改(具体的用法当然就要看ConfigurableListableBeanFactory提供了什么接口了).当然这是容器内部使用的方法,用户无法使用
            //具体模板方法
			postProcessBeanFactory(beanFactory);

			//⭐这个方法则是用于调用实现了BeanFactoryProcessor的bean的postProcessBeanFactory()方法,postProcessBeanFactory()和上一个供容器使用的方法相同,只不过这里专门供给给用户使用.实现了BeanFactoryProcessor的bean将会在这里实例化(即在此方法内调用getBean()实例化),再调用其postProcessBeanFactory()方法   
            //（还有一个BeanDefinitionRegistryPostProcessor要讲，这是BeanFactoryPostProcessor的子接口）
			invokeBeanFactoryPostProcessors(beanFactory);

			//注册后处理器,如AnnotaionAwareAspectJAutoProxyCretator就会在这里被注册进beanPostProcessors成员变量中,想要看这个类的前世今生或BeanDefinitionRegistry请看我的AOP框架
            //⭐具体的逻辑主要是扫描BeanDefinitionMap,找到其中的BeanPostProcessor实现类,并调用getBean()实例化此类,然后将其加入进beanPostProcessors成员变量中
			registerBeanPostProcessors(beanFactory);
			
			//国际化
            initMessageSource();
				
            //初始化事件广播器
			initApplicationEventMulticaster();
			 
           //具体模板方法
           // 具体的子类ApplicationContext可以在这里初始化一些特殊的 Bean（在初始化 singleton beans 之前）
			onRefresh();
			
            //初始化事件监听器
			registerListeners();
			
            //初始化非懒加载的bean
			finishBeanFactoryInitialization(beanFactory);
			//发布Spring内置事件ContextRefreshedEvent，想看事件源码请移步https://blog.csdn.net/shenchaohao12321/article/details/85303453
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```

### 1、`prepareBeanFactory(beanFactory)`

来看看解析完之后该方法到底添加了什么后处理器

```java
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		//⭐重点注意这个后处理器,这就是ApplicationContext对BeanFactory扩展的体现,在AbstractAutowireCapableBeanFactory中实现的initializeBean()里面就显式的调用了bean实现的BeanNameAware、BeanClassLoaderAware和BeanFactoryAware接口的set方法，当时的我天真的以为就只有这几个Aware接口可以让bean访问ioc容器了，谁知道ApplicationContext用后处理器给Aware接口还做了扩展
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		//这个后处理器实现的是postProcessAfterInitialization()，用于将实现了ApplicationListener的bean加入到广播器中，详情请看广播机制
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}

```

### 2、`finishBeanFactoryInitialization(beanFactory)`

上面的方法要不然是广播机制的实现、要不就是简单的逻辑，不再赘述，直接进入正题。

```java
finishBeanFactoryInitialization(beanFactory)→
   // DefaultListableFactory.java
    @Override
	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
                    //⭐注意:这里的getBean()方法的参数前面有'&',说明此时初始化的是工厂本身,而不是工厂创建的目标bean,所以这也就解释了为什么在使用ApplicationContext且工厂创建的目标bean是ApplicationListener的实现类时,如果此目标bean没有被其他bean引用(即没有被其他bean依赖注入,populateBean())时,即没有调用过getBean()创建过真正的目标bean,这个监听器不会生效,因为他根本没有被ApplicationListenerDectector这个后处理器处理过,这个目标bean甚至还没有被实例化,所以自然在广播器中找不到他的存在(如果还看不懂就去看bean加载的getObjectForBeanInstance()方法,只有在调用完getObject()获得目标bean后,此方法才会使得后处理器作用在目标bean上)
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							@Override
							public Boolean run() {
								return ((SmartFactoryBean<?>) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {
                    //普通的bean将会来到这里就行初始化,请看bean的加载.md
					getBean(beanName);
				}
			}
		}

		//在实例化完所有bean后,若有bean实现了SmartInitializingSingleton接口,则调用其afterSingletonsInstantiated()方法进行回调,如EventListenerMethodProcessor用来检测含有@EventListener的方法
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged(new PrivilegedAction<Object>() {
						@Override
						public Object run() {
							smartSingleton.afterSingletonsInstantiated();
							return null;
						}
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}

```

### 3、`finishRefresh()`

```java
protected void finishRefresh() {
   //初始化生命周期处理器
   initLifecycleProcessor();

   //调用生命周期处理器的onRefresh()方法
   getLifecycleProcessor().onRefresh();

   //发布Spring内置事件
   publishEvent(new ContextRefreshedEvent(this));

   // Participate in LiveBeansView MBean, if active.
   LiveBeansView.registerApplicationContext(this);
}
```

#### ①、`LifeCycle`相关接口

这里主要是讲一下Spring的`LifeCycle`及其一系列实现:

```java
public interface Lifecycle {

  
   void start();

  
   void stop();

 
   boolean isRunning();

}
```

虽然这个接口的方法看起来很高大上,但如果bean实现了这个接口并没有什么卵用,因为容器内并没有对这个接口的方法进行回调。如果要调用`beanLifeCycle`就必须要显式调用。

但是,当Spring将`BeanFactory`扩展到`ApplicationContext`时,它也将`LifeCycle`接口扩展了。在`ApplicationContext`中的`finishRefresh()`将会回调`LifeCycle`的子接口`LifeCycleProcessor`的`onRefersh()`方法,`onRefersh()`实现中可以进一步调用`start()`回调。方法容器内有一个该接口的默认实现：`DefaultLifeCycleProcessor`,待会在调用逻辑时讲

```java
public interface LifecycleProcessor extends Lifecycle {

   /**
    * Notification of context refresh, e.g. for auto-starting components.
    */
   void onRefresh();

   /**
    * Notification of context close phase, e.g. for auto-stopping components.
    */
   void onClose();

}
```

#### ②、`finishRefresh()`调用逻辑解析

回到`finishRefresh()`,进入`initLifecycleProcessor()`方法

```java
//初始化生命周期处理器
protected void initLifecycleProcessor() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
      this.lifecycleProcessor =
            beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
      if (logger.isDebugEnabled()) {
         logger.debug("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
      }
   }
   else {
      DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
      defaultProcessor.setBeanFactory(beanFactory);
      this.lifecycleProcessor = defaultProcessor;
      beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
      if (logger.isDebugEnabled()) {
         logger.debug("Unable to locate LifecycleProcessor with name '" +
               LIFECYCLE_PROCESSOR_BEAN_NAME +
               "': using default [" + this.lifecycleProcessor + "]");
      }
   }
}
```

**非常简单的逻辑:先判断是否有用户bean实现了`LifeCycleProcessor`接口,若有则使用用户的生命周期处理器;若没有则使用内置的`DefaultLifeCycleProcessor`。**

#### ③、`onRefresh()`

接下来我们进入`DefaultLifeCycleProcessor`的`onRefresh()`方法

```java
	@Override
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	}

    private void startBeans(boolean autoStartupOnly) {
    	//获得实现了LifeCycle接口的bean
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<Integer, LifecycleGroup>();
		for (Map.Entry<String, ? extends Lifecycle> entry : lifecycleBeans.entrySet()) {
			Lifecycle bean = entry.getValue();
            //⭐这里就是回调的关键,DefaultLifeCycleProcessor只会对实现SmartLifecycle的bean,且其实现方法isAutoStartup()必须返回true时,才会将该bean加入phases的map中,从而调用其start()方法
            //如果是普通的LifeCycle实现bean,但是在代码中显式调用了applicationContext.start(),applicationContext将会委托LifeCycleProcessor处理start请求,且参数autoStartupOnly=true.即若果是显式调用start(),只要有bean实现了LifeCycle接口则都会被调用,不限于SmartLifecycle实现bean
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);
				LifecycleGroup group = phases.get(phase);
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(entry.getKey(), bean);
			}
		}
    //phases不为空
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<Integer>(phases.keySet());
            //⭐使用SmartLifeCycle实现bean的getPhase()方法进行排序
			Collections.sort(keys);
            //⭐⭐⭐⭐最终就在这里调用所有可用的LifeCyclebean的start()方法
			for (Integer key : keys) {
                //这里有部分底层代码忽略掉了
				phases.get(key).start();
			}
		}
	}
```

上面谈到了`DefaultLifeCycleProcessor`的`onRefresh()`逻辑,bean必须是`SmartLifeCycle`的实现类才会被回调**(也就是说`SmartLifeCycle`其实是和`DefaultLifeCycleProcessor`绑定使用的,若自己实现`LifeCycleProcessor`则可以自己处理这个回调逻辑)**,康康这个接口

```java
//注意这个Phased接口,如果多个用户bean实现了SmartLifeCycle,且有控制这些bean调用的顺序需求,则可以通过该接口的getPhase()方法控制其顺序
public interface SmartLifecycle extends Lifecycle, Phased {

  //返回值决定了是否在容器刷新完成后回调该LifeCycleBean的start()方法
   boolean isAutoStartup();

  
   void stop(Runnable callback);

}
public interface Phased {

	/**
	 * Return the phase value of this object.
	 */
	int getPhase();

}
```

#### ④、`onClose()`

与`onRefersh()`相类似,`DefaultLifeCycleProcessor`在`onRefersh()`实现中调用`LifeCycleBean.start()`;而`DefaultLifeCycleProcessor`也会在`onClose()`实现中调用`LifeCycleBean.stop()`,只不过暂时没有找到调用`onClose()`的入口,所以这里不讲解

#### ⑤小结

总而言之,Spring将在所有Bean都加载完毕后调用实现了`SmartLifeCycle`类的bean的`strat()`,这个接口可以用来在容器刚启动完成后创建一些异步任务什么的。