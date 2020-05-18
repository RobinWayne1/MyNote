# 第五章 bean的加载

## 一、`getBean()`

```java
//AbstractBeanFactory.java
getBean()→
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
 	/* 获取一个 “正统的” beanName，处理两种情况，一个是前面说的 FactoryBean(前面带 ‘&’)，
    *一个是别名问题，因为这个方法是 getBean，获取 Bean 用的，你要是传一个别名进来，是完全可以的,
    *所以这里将会将这些name转换为容器内使用的唯一指定的id(即<bean id>定义的id)
   */
		final String beanName = transformedBeanName(name);
		Object bean;

		
    	//下面专门介绍:从缓存中获取单例bean,只找单例,因为原型一定会重新创建bean
    	//⭐获取的其中一种可能是获得已经创建完成的bean,另一种是正在创建的bean,如果正在创建的bean允许循环依赖则会将bean的工厂ObjectFactory放入缓存中(doCreateBean()里的addSingletonFactory()方法).在B创建依赖A时通过ObjectFactory提供的实例化方法来中断A的属性填充(即doCreateBean()方法里的populateBean()方法,方法内获取属性值也是通过getBean()方法获得的,使B中持有的A仅仅是刚刚初始化并没有填充任何属性的A,这就解决了循环依赖)
    //二二二二二二二二
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
            //在getSingleton()返回不为空的情况下再执行getObejctForBeanInstance()就是要判断getSingleton()返回的是否是FactoryBean,如果是则调用FactoryBean的getObject()将真正的bean返回.
            //方法内通过FactoryBean创建完bean之后有个后处理器,Spring尽可能保证所有bean初始化侯都会调用后处理器处理
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		//来到这说明bean还没有被创建过(或该bean不在此容器中),则要准备createBean
		else {
			//这个原型bean正在被创建,但此时又要开始创建这个bean,往往是填充属性时循环依赖了,则抛出异常
            //(原型的循环依赖是无解的对吧?)
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}
//---------------------------------------------------------------------------------------
			//父子容器的具体用处,如果当前容器没有该bean的beanDefinition,则尝试父容器中的到这个bean
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}
//---------------------------------------------------------------------------------------
			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				  
         		// 先初始化<bean depends-on="beanid"> 中depends-on定义的bean,该属性的作用就是在创建当前bean之前必须先创建depends-on指定的bean{@url https://www.iteye.com/blog/yanln-2210723}
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}
//---------------------------------------------------------------------------------------

				//来到这里时,既检查了单例缓存,也检查了父容器是否有此bean,且已经将depends-on指定的bean创建完成了,现在可以正式创建该bean
                //单例bean的创建
				if (mbd.isSingleton()) {
                    //三三三三三三
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
//---------------------------------------------------------------------------------------
				//原型bean的创建
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
//---------------------------------------------------------------------------------------
 				// 如果不是 singleton 和 prototype 的话，需要委托给相应的实现类来处理
				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
                        //⭐注意这里还有一次判断FactoryBean,容器开始刷新时单例缓存中没有FactoryBean的缓存,这个factoryBean照样是通过createbean()方法创建,创建完毕后再进行FactoryBean判断返回真正的目标bean
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}
//---------------------------------------------------------------------------------------
		//检查由beanName获取到的bean是否能转换为requiredType参数指定的类型
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

### 1.FactoryBean

如果某个bean的创建不只是对成员变量的注入,还要有许多其他的操作来初始化此bean,那么此时单纯的使用配置文件是无法代替Java语言的初始化操作的。基于此，我们可以使用`FactoryBean`。`FactoryBean`相当于目标bean的工厂类,由于目标bean的创建工作繁杂,所以我们将这项工作委托给工厂`FactoryBean`去进行而不是配置文件.

`FactoryBean`需要实现这三个接口:

```java
public interface FactoryBean<T> {
    //获得目标bean
    T getObject() throws Exception;
    Class<T> getObjectType();
    boolean isSingleton();
}
```

具体例子:

```java
public class MyCarFactoryBean implements FactoryBean<Car>{
    private String make; 
    private int year ;

    public void setMake(String m){ this.make =m ; }

    public void setYear(int y){ this.year = y; }

    public Car getObject(){ 
      // 这里我们假设 Car 的实例化过程非常复杂，反正就不是几行代码可以写完的那种
      CarBuilder cb = CarBuilder.car();

      if(year!=0) cb.setYear(this.year);
      if(StringUtils.hasText(this.make)) cb.setMake( this.make ); 
      return cb.factory(); 
    }

    public Class<Car> getObjectType() { return Car.class ; } 

    public boolean isSingleton() { return false; }
}
```

当需要使用Car时,只需要调用getBean("myCarFactoryBean"),此时则会在`doGetBean()`方法中,执行**`getObejctForBeanInstance()`**以将真正需要的bean——`Car`返回

```xml
<bean class = "com.Robin.MyCarFactoryBean" id = "car">
  <property name = "make" value ="Honda"/>
  <property name = "year" value ="1984"/>
</bean>
<bean class = "com.Robin.Person" id = "josh">
  <property name = "car" ref = "car"/>
</bean>
```

注:在`getSingleton()`和`createBen()`后都有`getObejctForBeanInstance()`方法

`getObejctForBeanInstance()`的主体:

```java
getObejctForBeanInstance()→
    //在getObejctForBeanInstance()方法内会先判断要得到的是工厂还是目标bean,如果是目标bean则会进入此方法
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
   if (factory.isSingleton() && containsSingleton(beanName)) {
      synchronized (getSingletonMutex()) {
          //若是单例,已创建过的目标bean会存放在factoryBeanObjectCache中
         Object object = this.factoryBeanObjectCache.get(beanName);
         if (object == null) {
            object = doGetObjectFromFactoryBean(factory, beanName);
             //直接调用getObject()方法
        →{
            return factory.getObject();
        }
            Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
            if (alreadyThere != null) {
               object = alreadyThere;
            }
             //一般情况都是来到这里
            else {
               if (object != null && shouldPostProcess) {
                  if (isSingletonCurrentlyInCreation(beanName)) {
                     // Temporarily return non-post-processed object, not storing it yet..
                     return object;
                  }
                  beforeSingletonCreation(beanName);
                  try {
                      //⭐调用后置后处理器
                     object = postProcessObjectFromFactoryBean(object, beanName);
                  }
                  catch (Throwable ex) {
                     throw new BeanCreationException(beanName,
                           "Post-processing of FactoryBean's singleton object failed", ex);
                  }
                  finally {
                     afterSingletonCreation(beanName);
                  }
               }
               if (containsSingleton(beanName)) {
                   //将单例目标bean加入cache中
                  this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
               }
            }
         }
         return (object != NULL_OBJECT ? object : null);
      }
   }
   else {
      Object object = doGetObjectFromFactoryBean(factory, beanName);
      if (object != null && shouldPostProcess) {
         try {
            object = postProcessObjectFromFactoryBean(object, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
         }
      }
      return object;
   }
}
```

**比较简单的逻辑:检查要得到工厂还是目标bean→若是目标bean,则检查cache中是否有该目标bean→若没有,则调用工厂bean的`getObject()`获取目标bean→调用后置后处理器**

缺点:使用`FactoryBean`创建的bean不再属于容器管理的bean,即所有Spring提供的注入方式不能在这个bean内使用(看`getObejctForBeanInstance()`,显而易见没有`populateBean()`方法),但后处理器还可以作用在这个bean上,所以`ApplicationListener`对它来说还是能够作用的(要注意`refresh()`时生成的是factoryBean而不是目标bean,看ApplicationContext的笔记)

### 2.从缓存中获取单例bean(解决循环依赖) `getSingleton()`

首先介绍用于存储bean的不同map:

```java
//用于保存BeanName和创建bean实例(bean)的关系,一个bean完整创建完后则会被放到此map中.该一级缓存的特点是:bean已经完完全全初始化完成,并且已经是可用的状态了
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

//用于保存BeanName和ObjectFactory工厂之间的关系,用于循环依赖出现的第一次时得到未初始化完的bean实例,之后立马删除这个工厂.这个三级缓存的特点是:1.获取ObjectFactory之后调用objectFactor.getObject()会应用要获取的bean的后处理器,如这个bean中的有方法属于AOP增强的目标,则AnnotaionAwareAutoProxyCreator将会应用于改本bean,将该bean用JDK动态代理包装后才返回2.此时Spring还没有对该bean进行属性注入
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

//当第二次实现循环依赖又来到此bean时,该成员变量将会返回未初始化完的bean实例给getBean()的调用者.这个二级缓存的特点是:1.存入二级缓存的Bean此时都已应用了后处理器2.此时Spring还没有对该bean进行属性注入
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
```



```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   Object singletonObject = this.singletonObjects.get(beanName);
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
          //如果不是第一次处理循环依赖,则会在这里找到实例
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
             //在doCreateBean()方法中,会在createBeanInstance()反射创建实例之后,populateBean()之前将含有bean实例的ObjectFactory放进this.singletonFactories成员变量中,若在属性注入时出现循环依赖又回来到此bean,则会返回singletonFactory中未初始化完成的bean
            if (singletonFactory != null) {
                //应用后处理器并返回singletonFactory中未初始化完成的bean，
               singletonObject = singletonFactory.getObject();
                //此时将bean从三级缓存转移到二级缓存
               this.earlySingletonObjects.put(beanName, singletonObject);
                //强制移除三级缓存，二级缓存有的bean三级缓存不可以有也没必要有，看后面就明白这两个缓存的设计思路
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

主要就是从这三级缓存中获取实例。接下来我们就分析Spring为了解决循环依赖引出的一系列巧妙的设计。

#### ①、循环依赖的解决

##### Ⅰ、何为循环依赖

循环依赖分为以下三种：

1. A的构造方法中依赖了B的实例对象，同时B的构造方法中依赖了A的实例对象
2. A的构造方法中依赖了B的实例对象，同时B的某个field或者setter需要A的实例对象，以及反之
3. A的某个field或者setter依赖了B的实例对象，同时B的某个field或者setter依赖了A的实例对象，以及反之

第一种循环依赖是不可能解决的，这是程序的特性使然，两个`<init>`会不断的调用直到报`StackOverFlowError`错误。而Spring解决的是后面两种，因为依赖注入是Spring的特性，所以Spring就有处理这种错误的手段。

##### Ⅱ、循环依赖的解决思想

在`createBean()`中,在`createBeanInstance(beanName, mbd, args);`反射创建完Bean实例后,有这样的代码:

```java
//AbstractBeanFactory.java
//检测是否需要提前曝光,检测当前bean是否是单例bean且是否允许循环依赖
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isDebugEnabled()) {
			logger.debug("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
        //⭐⭐如果需要提前曝光，在初始化完成前将ObjectFactory加三级缓存singletonFactories,用于解决循环依赖
		addSingletonFactory(beanName, new ObjectFactory<Object>() {
			@Override
			public Object getObject() throws BeansException {
				return getEarlyBeanReference(beanName, mbd, bean);
			}
		}));
	}
```
逐行代码分析

```java
//这就是向三级缓存中加入ObjectFactory的方法	
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
                //想三级缓存中加入ObjectFactory
				this.singletonFactories.put(beanName, singletonFactory);
                //强制移除二级缓存的bean,因为bean的获取顺序是一级缓存→二级缓存→三级缓存,如果二级缓存中有bean,则三级缓存的bean就没有意义了
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
	}
```

⭐然后来看看`getEarlyBeanReference()`

```java
//AbstractBeanFactory.java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
   Object exposedObject = bean;
   if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
       //获取该Bean的后处理器
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
          //如果该后处理器有实现SmartInstantiationAwareBeanPostProcessor接口,调用其getEarlyBeanReference()方法
         if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
            SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
            exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
            if (exposedObject == null) {
               return null;
            }
         }
      }
   }
   return exposedObject;
}
//我们来看SmartInstantiationAwareBeanPostProcessor的其中一个实现
//AbstractAutoProxyCreator.java
	@Override
	public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
		Object cacheKey = getCacheKey(bean.getClass(), beanName);
		if (!this.earlyProxyReferences.contains(cacheKey)) {
			this.earlyProxyReferences.add(cacheKey);
		}
        //⭐包装此bean,返回一个该bean的动态代理对象
		return wrapIfNecessary(bean, beanName, cacheKey);
	}
```

现在就已经很清晰明了了。三级缓存存入的`ObjectFactory`的作用就是在要解决循环依赖调用`objectFactory.getObject()`时,先对该bean应用后处理器。为什么要应用后处理器？可以看到，`addSingletonFactory()`是在`createBeanInstance()`反射创建完实例,`populateBean()`之前甚至可以说`initializeBean()`之前调用的,这样就说明此时放入三级缓存中的那个bean并没有应用后处理器。例如两个需要被增强的bean ——bean1和bean2循环依赖，Spring先初始化bean1.如果不应用后处理器将bean1包装成代理直接就返回给bean2的`populateBean()`的话（在`populateBean()`就已经对依赖注入的成员真正赋值完毕了），此后在bean2真正运行的时候，由于注入的bean1不是Spring中那个代理bean1了，则对bean1的调用Spring就不会应用增强，致使产生错误。所以这就是二级缓存和三级缓存存在的必要性,就是为了区分有应用过后处理器和没有应用过后处理器的bean。

### 3.缓存中无实例的情况下获取单例——`createBean()`

```java
sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
					@Override
					public Object getObject() throws BeansException {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
						
							destroySingleton(beanName);
							throw ex;
						}
					}
				});
```
```java
getSingleton(String beanName,ObjectFactory singletonFactory)
{
    //钩子,记录加载状态
    beforeSingletonCreation(beanName);
    //创建bean,即是调用createBean(beanName, mbd, args)方法
    singletonObject=singletonFactory.getObject();
    //钩子,移除加载状态记录
    afterSingletonCreation(beanName);
    //将bean加入singletonObjects缓存
    addSingleton(beanNmae,singletonObject);
    
    return singletonObject;
}
```

#### ①、==`createBean()`==

```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
            //⭐根据bean使用对应策略创建新的实例
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		
        //检测是否需要提前曝光
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
            //⭐⭐如果需要提前曝光，在初始化完成前将ObjectFactory加三级缓存singletonFactories,用于解决循环依赖
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			}));
		}

	
		Object exposedObject = bean;
		try {
            //⭐对bean进行属性填充,其中可能存在依赖于其他bean的属性则会递归初始化依赖bean
			populateBean(beanName, mbd, instanceWrapper);
            //⭐执行后处理器、initmethod等等多个回调函数
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

~~~java
protected BeanWrapper  createBeanInstance()
{
    // 如果工厂方法(即<bean factory-method="">)不为空则使用工厂方法初始化
    return instantiateUsingFactoryMethod(beanName, mbd, args);
    //构造函数自动注入
    return autowireConstructor();
    //使用默认构造函数构造(无参数),利用反射创建实例
    return  instantiateBean();
}
~~~

##### Ⅰ、`createBeanInstance()`里的`autowiredConstructor()`:

带有参数的bean实例化步骤

1. 首先,先判断开发人员在使用getBean(String name,Object...args)方法时有没有给定参数explicitArgs,如果有则可以直接将其当作构造器的参数,没有的话尝试从配置文件解析.

2. 从配置文件解析时首先查看一下缓存中有没有已经解析好了的构造器参数赋值给argsToResolved,如果有则将构造器参数转换转换为最终值(因为缓存中的值可能是原始值和最终值,例如构造器需要int类型,而缓存中是"1"时,则要进行转换).

3. 而如果没有被缓存则要重新解析.首先提取配置文件的构造函数参数<constructor-args>,然后调用resolveConstructorArguments()将解析到的参数个数返回给minNrOfArgs,同时在方法内把参数名,值等信息赋值给resolvedValues,最重要的是方法还将参数所属的类加载了.

4. 接下来就是选定构造器.将所有构造器按参数个数降序排序,并进行迭代,迭代到有相同个数参数的构造器,则获取其参数名,并根据参数名,参数类型,参数值,构造器等通过createArgumentArray()方法创建参数持有者-->argsHolder,argsHolder包含有原始值和类型转换后参数值,选定的构造函数匹配权重等信息.其中,这个方法将会转换参数类型,如果转换不成功(即类型不匹配)则会抛出异常,countinue;继续迭代下一个构造器.

5. 最后就通过argsHolder的匹配权重将最接近匹配的选择为构造函数,并创建实例返回.

```java
autowireConstructor()
{	//
    this.beanFactory.initBeanWrapper(bw);
    {
        //注册一系列常用的属性编辑器,注册后在属性填充环节就可以直接让Spring使用这些编辑器进行属性的解析.
		registerCustomEditors(bw);
    }
    if(explicitArgs!=null)
    {
        argsToUse=explicitArgs;
    }
    else
    {
        //尝试从缓存中取
        argsToResolve=mbd.preparedConstructorArguments;
       	//缓存中有则,
        if(argsToResolve!=null)
            argsToUse=resolvePreparedArguments();
    }
    //没有被缓存
    if(constructorToUse==null)
    {	//提取配置文件的构造函数参数
        ConstructorArgumentValues cargs=mbd.getConstructorArgumentValues();
        //承载解析后构造函数参数的值
        resolvedValues = new ConstructorArgumentValues();
        //能解析到的参数个数
		minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
        //根据参数排序
        AutowireUtils.sortConstructors(candidates);
        for(int i=0;i<candidates.length;i++)
        {//匹配参数个数步骤,已省略
            
         //获取参数名,已省略
            
         //根据名称和数据类型创建参数持有者
            argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,getUserDeclaredConstructor(candidate), autowiring);
            
            
            int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
						argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
				// 如果他代表最接近的匹配则选择为构造函数
				if (typeDiffWeight < minTypeDiffWeight) {
                    //里面是对constructorToUse,argsToUse等的赋值
                }
    	}
    
    }
    
    //利用reflect创建Bean,其中还有一些对AOP的增强
    beanInstance = strategy.instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
    //将Bean实例赋值到BeanWrapper对象中,返回
    bw.setBeanInstance(beanInstance);
	return bw;
}
```

##### Ⅱ、`doCreateBean()`里的`populateBean()`:

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    //PropertyValues存放着从XML文件中解析出来的属性注入配置信息
   PropertyValues pvs = mbd.getPropertyValues();

   if (bw == null) {
      if (!pvs.isEmpty()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
         
         return;
      }
   }

  
   boolean continueWithPropertyPopulation = true;

   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               continueWithPropertyPopulation = false;
               break;
            }
         }
      }
   }

   if (!continueWithPropertyPopulation) {
      return;
   }

   if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
         mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

      // Add property values based on autowire by name if applicable.
      if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
          
          //⭐⭐⭐根据名称注入
         autowireByName(beanName, mbd, bw, newPvs);
   →→→{
       for (String propertyName : propertyNames) {
			if (containsBean(propertyName)) {
                //⭐递归初始化bean
				Object bean = getBean(propertyName);
                //将成员变量值 bean放入PropertyValues对象中保存
				pvs.add(propertyName, bean);
				registerDependentBean(propertyName, beanName);
				if (logger.isDebugEnabled()) {
					logger.debug("Added autowiring by name from bean name '" + beanName +
							"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
				}
			}
    }
      }

      // Add property values based on autowire by type if applicable.
      if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
          
          //根据类型注入,主要功能就是类型匹配
	autowireByType(beanName, mbd, bw, newPvs);
 →→→{
        //寻找类型匹配,太复杂了看不懂
        resolveDependency();
    }
      }

      pvs = newPvs;
   }

   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
   boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

   if (hasInstAwareBpps || needsDepCheck) {
      PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      if (hasInstAwareBpps) {
        // 使@Autowired和@Value等发挥作用的AutowiredAnnotationBeanPostProcessor就是在这里被调用的,其实现方法postProcessPropertyValues()将会通过反射使成员变量赋值
         for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
               InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
               pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                //注意:就算AutowiredAnnotationBeanPostProcessor发挥了作用,在此处也不会返回,也就是说基于配置文件属性注入和基于注解属性注入可以混合使用
               if (pvs == null) {
                  return;
               }
            }
         }
      }
      if (needsDepCheck) {
         checkDependencies(beanName, mbd, filteredPds, pvs);
      }
   }
  //上面两个方法只是将属性用Property Value的形式表示,并没有应用到bean里面,这个工作在下面方法执行(指基于XML的注入方式),将BeanDefinition里的属性类型转换为对应的setter参数的类型注入
   applyPropertyValues(beanName, mbd, bw, pvs);
}
```

##### Ⅲ、==`doCreateBean()`里的`initializeBean()`:==

来看看创建完bean后的回调函数：

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged(new PrivilegedAction<Object>() {
         @Override
         public Object run() {
            invokeAwareMethods(beanName, bean);
            return null;
         }
      }, getAccessControlContext());
   }
   else {
      // 如果 bean 实现了 BeanNameAware、BeanClassLoaderAware 或 BeanFactoryAware 接口，回调
       //⭐这个就是BeanFactory的基础实现，ApplicationContext用后处理器对Aware接口进行了扩展
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
      // BeanPostProcessor 的 postProcessBeforeInitialization(前置后处理器) 回调
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
      // 处理 bean 中定义的 init-method，
      // 或者如果 bean 实现了 InitializingBean 接口，调用 afterPropertiesSet() 方法
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }

   if (mbd == null || !mbd.isSynthetic()) {
      // BeanPostProcessor 的 postProcessAfterInitialization(后置后处理器) 回调
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }
   return wrappedBean;
}
```

逻辑比较简单，无非是调用`Aware`接口赋值、调用后处理器和调用<init-method>.

###### ₁.init-method

在bean创建完成后,如果该bean在配置中制定了<init-method>的方法名或该bean实现了`InitializingBean `,则在`invokeInitMethods()`就会利用反射调用相应的方法.

```java
//SqlSessionFactoryBean的入口就是在这个方法中
public class AnotherExampleBean implements InitializingBean {

    public void afterPropertiesSet() {
        // do some initialization work
    }
}
```

```xml
<bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>
```

###### ₂.BeanPostProcessor

```java
public interface BeanPostProcessor {

  
   Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

   
   Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```

太熟悉的一个接口了,Spring非常非常多的功能都是基于这个接口实现的,如`AnnotationAwareAspectJAutoProxyCreator`、`ApplicationListenerDectector`、`AutowiredAnnotationBeanPostProcessor`(虽然这个不在`initializeBean()`里作用)