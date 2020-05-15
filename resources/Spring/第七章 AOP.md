# 第七章 AOP

### 一、增强的执行顺序

`before`→`around`→`after`→不出错`afterReturning`

​												→出错`afterThrowing`

`@Order(value)`修饰切面时,`value`的值越大,则该切面所有的增强则会放在拦截器链的最前面;`value`的值越小,则该切面所有的增强都会放在拦截器链的最后面

### 二、源码解析

如果配置文件有

```xml
<aop:aspectj-autoproxy/>
```

就会开启基于注解的AOP解析,使用解析器AspectJAutoProxyBeanDefinitionParser进行解析,因为所有解析器都是对BeanDefinitionParser的实现，所以解析器的入口是parse函数

```java
parse()
{
	//注册AnnotationAwareAspectJAutoProxyCreator这个Bean,往注册表内添加这个Bean的BeanDefinition
	registerAspectJAnnotationAutoProxyCreatorIfNecessary()
	{
    
	}
}
```

**AnnotationAwareAspectJAutoProxyCreator实现了BeanPostProcessor接口并继承自AbstractAutoProxyCreator,==而且还实现了BeanFactoryAware接口,所以在initializedBean()方法中会将beanfactory传给这个bean==,所以所以当加载这个Bean时会在实例化后调用postProcessAfterInitialization()方法(即每个bean一个代理)**。在`registerBeanPostProcessors()`方法中会将这个BeanPostProcessor注册,即将这个处理器当bean一样反射实例化(getBean())

注意:postProcessAfterInitialization()的调用时间为属性注入完成之后的init-method过程中,在initalizeBean调用的,也就是说此时在getBean()阶段

```java
postProcessAfterInitialization()
{
    //如果它适合被代理,则封装指定bean
    wrapIfNecessary()
    {
        //如果存在增强方法则创建代理
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
        //如果获取到增强就需要针对增强创建代理
        if(){
       Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    }
    
    }
}
```

```java
getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null)
{
    findEligibleAdvisiors();
    {
        //获取增强器
     List<Advisor> candidateAdvisiors =findCandidateAdvisiors();{
        	//对XML配置的支持,调用父类的方法加载配置文件中的AOP声明
       	 	List<Advisor> advisors = super.findCandidateAdvisors();
			//以注解的方式获取增强
			if (this.aspectJAdvisorsBuilder != null) {
			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
			}
			return advisors;
        }
        //根据上面方法返回的增强器寻找与当前类方法匹配的增强器,当前类指的是getBean()方法的得到的类
        List<Advisor> eligibleAdvisors	=findAdvisiorsThatCanApply(candidateAdvisiors,beanClass,beanName);
        {
            if(canApply(candidateAdvisor,beanClass,hasIntroductions)
            {//遍历当前类方法判断是否有匹配切点表达式成功的,成功则返回true
            })
            {
                //向Advisors列表中加入合格的Advisor
                eligibleAdvisors.add(candidateAdvisor)
            }    
                
        }
        //将得到的增强器排序
        sortAdvisor();
    
}
```

```java
//此方法为每个增强方法创建一个advisor,将增强方法包装进去,注意这里是以切面为单位的,即有一个切面beanname和List<advisor>的映射⭐别忘了这里是在bean的后处理器当中
//重点方法,重看
⭐buildAspectJadvisors()
{
    //获取所有beanName
    String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
							this.beanFactory, Object.class, true, false);
    //遍历所有beanName
    for (String beanName : beanNames) 
    {
        //如果存在Aspect注解
        if (this.advisorFactory.isAspect(beanType)) 
        {
            //获取切面的增强方法
            List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
        }
    }
}
```

getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory)方法有三个主要方法:==1.对普通增强器的获取==2.增加同步实例化增强器3.获取DeclareParents注解(主要用于引介增强的注解形式的实现),接下来我们只看对普通增强器的获取方法分析getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);

上述`findCandidateAdvisiors()`方法不将拦截器链首先制作出来的原因是MethodMatcher还有要进行动态匹配的时候,即根据不同的参数产生不同的拦截器链(应该是这样)

重点方法:

```java
getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory)
{
    //getAdvisorMethods()获得了增强方法
    for (Method method : getAdvisorMethods(aspectClass)) {
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}
}
```

```java
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
			int declarationOrderInAspect, String aspectName) 
{
		//获取切点信息
		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		
		//根据切点信息生成增强器
		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}
```

```java
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) 
{
    //获取增强方法上的注解
		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		
		//用于承载切点信息的对象
		AspectJExpressionPointcut ajexp =
				new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
    	//从得到的注解中获取里面的切点表达式,如@After("test()")中的test()
		ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
		
		return ajexp;
	}
```

```java
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
			Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

		this.declaredPointcut = declaredPointcut;
		this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
		this.methodName = aspectJAdviceMethod.getName();
		this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
		this.aspectJAdviceMethod = aspectJAdviceMethod;
		this.aspectJAdvisorFactory = aspectJAdvisorFactory;
		this.aspectInstanceFactory = aspectInstanceFactory;
		this.declarationOrder = declarationOrder;
		this.aspectName = aspectName;

		//...忽略了一部分代码
			// A singleton aspect.
			this.pointcut = this.declaredPointcut;
			this.lazy = false;
    	//重点
			this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
		}
	}
```

instantiateAdvice()方法里的getAdvice()方法,所以getAdvisor()方法内的主要两个功能就是获取切点表达式和创建增强对象

```java
public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
      MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

   Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
   validate(candidateAspectClass);
	//获取增强方法上的注解
   AspectJAnnotation<?> aspectJAnnotation =
       
      AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
   if (aspectJAnnotation == null) {
      return null;
   }

//...省略一部分代码

   AbstractAspectJAdvice springAdvice;
//根据注解的类型生成不同的增强器
   switch (aspectJAnnotation.getAnnotationType()) {
      case AtPointcut:
         if (logger.isDebugEnabled()) {
            logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
         }
         return null;
      case AtAround:
         springAdvice = new AspectJAroundAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtBefore:
         springAdvice = new AspectJMethodBeforeAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtAfter:
         springAdvice = new AspectJAfterAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtAfterReturning:
         springAdvice = new AspectJAfterReturningAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
         if (StringUtils.hasText(afterReturningAnnotation.returning())) {
            springAdvice.setReturningName(afterReturningAnnotation.returning());
         }
         break;
      case AtAfterThrowing:
         springAdvice = new AspectJAfterThrowingAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
         if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
            springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
         }
         break;
      default:
         throw new UnsupportedOperationException(
               "Unsupported advice type on method: " + candidateAdviceMethod);
   }


   return springAdvice;
}
```

再回到postProcessAfterInitialization()的createProxy()方法创建代理

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		/**
		 * 判断是基于接口代理还是基于类代理(即Jdk或Cglib)
		 */
		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);


		//存入AdvisedSupport中的advisor都是进行了第一次过滤,即都是适用与此类的某个方法的
		//真正代理的时候就要以方法为精度再次过滤
		proxyFactory.addAdvisors(advisors);
		//需要代理的类的targetSource,用TargetSource封装它的单例和原型特性
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);
		//如果为true则不允许改动advice
		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			/**
			 * Set whether this proxy configuration is pre-filtered so that it only
			 * contains applicable advisors
			 *
			 * ⭐(matching this proxy's target class)⭐.
			 *
			 * 	<p>Default is "false". Set this to "true" if the advisors have been
			 * 	pre-filtered already,
			 *
			 * 	meaning that the ClassFilter check can be skipped
			 * 	when building the actual advisor chain for proxy invocations.
			 */
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}

```

```java
@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
   if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
      Class<?> targetClass = config.getTargetClass();
      if (targetClass == null) {
         throw new AopConfigException("TargetSource cannot determine target class: " +
               "Either an interface or a target is required for proxy creation.");
      }
      if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
         return new JdkDynamicAopProxy(config);
      }
      return new ObjenesisCglibAopProxy(config);
   }
   else {
      return new JdkDynamicAopProxy(config);
   }
}
```

optimize:用来控制通过CGLIB创建的代理是否使用激进的优化策略

proxyTargetClass:proxy-target-class="true/false"

hasNoUserSuppliedProxyInterfaces:是否存在代理接口



下面讲解jdk动态代理,所以主要两个方法就是继承了InvocationHanlder接口的JdkDynamicAopProxy类的getProxy()和Invoke()方法

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   MethodInvocation invocation;
   Object oldProxy = null;
   boolean setProxyContext = false;

   TargetSource targetSource = this.advised.targetSource;
   Object target = null;
	//获取当前方法的拦截器链,在此时才解析调用的拦截器链,并没有实现解析好
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

      if (chain.isEmpty()) {
   		 //如果没有发现拦截器则直接调用目标方法
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
      }
      else {
        //将拦截器封装在ReflectiveMethodInvocation中方便使用proceed进行链接表用拦截器
         invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
         //执行拦截器链
         retVal = invocation.proceed();
      }
}
```

```java
public Object proceed() throws Throwable {
   // We start with an index of -1 and increment early.
    //如果迭代的当前拦截器下标=拦截器ArrayList的size,则调用目标方法
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint();
   }
	//拦截器下标加1
   Object interceptorOrInterceptionAdvice =
         this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
   if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
      // Evaluate dynamic method matcher here: static part will already have
      // been evaluated and found to match.
      InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
      if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
         return dm.interceptor.invoke(this);
      }
      else {
         // Dynamic matching failed.
         // Skip this interceptor and invoke the next in the chain.
         return proceed();
      }
   }
   else {
	//调用拦截器的invoke()方法
      return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
   }
}
```

而proceed()方法里则控制着拦截器链接调用的计数器,记录当前调用链接的位置,==注意:在getAdvicesAndAdvisorsForBean()方法中有一个将增强器排序的方法,排序是为了在后面拦截器链接调用的时候符合调用的递归顺序,与实际增强方法调用顺序无关,真正的增强方法调用顺序是在各个增强器内部通过proceed()方法调用下一个拦截器和invoke()调用自身拦截器实现,也就是递归调用的实现顺序==

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
   try {
      return mi.proceed();
   }
   finally {
      invokeAdviceMethod(getJoinPointMatch(), null, null);
   }
}
```

例如,这是一个AspectJAfterAdvice类里的方法,拦截器的顺序为after,around,before,所以最先执行这个拦截器,但是进入这个方法后首先调用了mi.proceed()进入了下一个拦截器,意思就是先调用下一个拦截器再调用自己.

```java
public Object invoke(MethodInvocation mi) throws Throwable {
   this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
   return mi.proceed();
}
```

这是MethodBeforeAdviceInterceptor类的方法,则先调用自身的增强再进入下一个拦截器.

环绕增强:

```java
@Around("test()")
public Object aroundTest(ProceedingJoinPoint p)
{
   System.out.println("beforeAround");
   Object o=null;
   try
   {
      o=p.proceed();

   }
   catch (Throwable e)
   {
      e.printStackTrace();
   }
   System.out.println("afterAround");
   return o;
}
```

拦截器顺序为after,around,before,`interceptorAndDynamicMethodMatcher[0]`为一个空拦截器,首先进入调用空拦截器,这个拦截器的`invoke()`方法没有做任何调用增强方法的操作,只是调用下一个拦截器,于是递归到after增强器,由于after增强器先调用下一个拦截器,于是进入around增强,在`invokeAdviceMethodWithGivenArgs()`方法里面直接调用增强方法,输出=="beforeAround"==,然后进入`p.proceed()`,进入了下一个拦截器before,先调用了before增强,输出=="before"==,再进入了下一个拦截器,此时的拦截器下标=ArrayList下标,于是调用目标方法输出=="test"==,调用完后一路返回到`o=p.proceed();`,继续环绕通知的后半部分输出=="afterAround"==,然后一路返回到AspectJAfterAdvice类的`invoke()`的`   return mi.proceed();`,然后调用after增强输出=="after"==,结束AOP