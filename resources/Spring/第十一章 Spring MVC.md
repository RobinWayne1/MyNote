# 第十一章 Spring MVC

### SpringMVC基本原理

![](E:\Typora\MyNote\resources\Spring\SpringMVC原理.png)

### 二、ContextLoaderListener

 ContextLoaderListener的作用是再启动web容器时,自动装配ApplicationContext的配置信息.由于它实现了ServletContextListener这个接口,==在web.xml配置这个监听器,启动容器时就会默认执行它实现的方法(即创建ServletContext对象的时候调用`contextInitialized()`),使用这个接口可在为客户端请求提供服务之前向ServletContext中添加任意对象==。这个对象在ServletContext启动的时候被初始化，然后在ServletContext整个运行期间都可见，且每个应用都有一个ServletContext。

ServletContext启动之后会调用ServletContextListener的contextInitialized（）方法

```java
contextInitialized(ServletContextEvent event)
{	
    
    this.contextLoader.initWebApplicationContext(event.getServletContext());
    {
        //判断是否已存在WebApplicationContext,如果已存在则表明已经有其他ServletContextListener被创建,而Spring只允许声明一次ServletContextListener
        if (servletContext.getAttribute
            (WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null){
            //利用反射创建WebApplicationContext实例,作为SpringMVC的父容器,其中会根据ContextLoader.properties文件提取要实现WebApplicationContext接口的实现类,根据这个实现类通过反射的方式创建实例
        this.context = createWebApplicationContext(servletContext);
            //⭐仔细重看这个方法,有疑问.加载IOC容器(父容器)的bean
        configureAndRefreshWebApplicationContext(cwac, servletContext);
            
        //⭐将WebApplicationContext记录在servleContext中供全局使用(所以IOC容器是全局唯一并且是全局共享的,所以不用在IOC容器内实现全局共享了,ContextLoaderListener就是结合SpringMVC时加载容器的入口,一般情况下就是通过ApplicationContext)
        servletContext.setAttribute(
                WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, 	                  	  this.context);
        }
    }
}
```

### 三、DispatcherServlet

DispatcherServlet实现HttpServlet接口,也就是实现了init(),service(),和destory()方法,当Web容器接收到某个servlet请求时,servlet把请求封装成一个HttpServletRequest对象传给Servlet对应的服务方法.

#### 1.初始化阶段 

* 初始化时servlet容器把servlet类的class文件数据读到内存
* servlet容器创建一个ServletConfig对象,包含了servlet的初始化配置信息
* servlet容器创建一个servlet对象
* servlet容器调用init()方法初始化

首先我们则来介绍dispatcherServlet的父类HttpServletBean重写的init()方法

```java
init()
{
    //解析init-param并封装在pvs里
	PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), 		this.requiredProperties);
    //将当前servlet转化成BeanWrapper,从而能够以Spring的方式对init-param的值进行注入
    BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
    //属性注入
    bw.setPropertyValues(pvs, true);
    //初始化servletBean
    initServletBean();
}
```

initServletBean()方法的作用则是创建子IOC容器,也就是子WebApplicationContext,用来放置SpringMVC所要用的Bean,而ContextLoaderListener创建的父容器则用来放Spring使用的Bean,其中后面dispatcherServlet将ContextLoaderListener的容器设置为父容器使得Spring的bean在SpringMVC里也可以用

~~~java
initServletBean();
{
    this.webApplicationContext = initWebApplicationContext();
    {
        //webApplicationContext已存在的情况
        
        //通过contextAttribute初始化
        
        //重新创建WebApplicationContext,这个context就是SpringMVC子容器
        createWebApplicationContext();
        {
            //利用反射实例化ApplicationContext
            ConfigurableWebApplicationContext wac =
				(ConfigurableWebApplicationContext)
                BeanUtils.instantiateClass(contextClass);
            //设置父容器
            wac.setParent(parent);
            configureAndRefreshWebApplicationContext(wac);
            {
                //applicationContext里的refresh()进行配置文件加载及功能扩展
                wac.refresh();
                {
                    //最后会调用此方法进行SpringMVC所需要的组件bean的创建,因为refresh()结束,配置文件已经解析完成,bean可以真正被创建使用
                    initStrategies();
                }
            }
        }
    }
}
~~~

```java
protected void initStrategies(ApplicationContext context) {
   initMultipartResolver(context);
   initLocaleResolver(context);
   initThemeResolver(context);
    //初始化HanlderMappings
   initHandlerMappings(context);
    //初始化HanlderAdapters
   initHandlerAdapters(context);
   initHandlerExceptionResolvers(context);
   initRequestToViewNameTranslator(context);
   initViewResolvers(context);
   initFlashMapManager(context);
}
```

其中,在初始化HanlderMappings中,我们可以提供多个HanlderMapings供其使用,DispatcherServlet选用HanlderMapings的过程中将根据我们指定的HanlderMapings优先级排序,优先遍历在前的HanlderMapings,如果当前HanlderMapings能够返回可用的Hanlder,则使用当前返回的Handler进行web请求的处理,不在询问其他HanlderMapings

init()方法结束后则进入等待请求的状态

#### 2.DispatcherSerlvet的逻辑处理

主要的处理逻辑的方法doDispatch()

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {


    	//对于multipart的处理
         processedRequest = checkMultipart(request);
         //根据request信息寻找对应handler
         mappedHandler = getHandler(processedRequest);
         if (mappedHandler == null || mappedHandler.getHandler() == null) {
             //如果没有找到hanlder的情况的错误处理
            noHandlerFound(processedRequest, response);
            return;
         }
		//根据当前hanlder寻找对应的适配器
        
         HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

     	//如果当前hanlder支持last-modified的处理,已省略代码
		
    	//prehanlder()的调用
         if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
         }
		
         //调用controller方法,进行数据绑定等功能
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

    	//调用postHandle方法
         mappedHandler.applyPostHandle(processedRequest, response, mv);
      
   //根据视图跳转页面
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
 
}
```

首先介绍getHandler()方法:

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   for (HandlerMapping hm : this.handlerMappings) {
      if (logger.isTraceEnabled()) {
         logger.trace(
               "Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
      }
      HandlerExecutionChain handler = hm.getHandler(request);
      if (handler != null) {
         return handler;
      }
   }
   return null;
}
```

方法通过遍历所有的HandlerMapping,并调用getHandler()方法进行封装处理,以SimpleUrlHandlerMapping为例

```java
getHandler()
{
    Object handler = getHandlerInternal(request);
    //将用户自定义配置的拦截器加入到执行链中
    return getHandlerExecutionChain(handler,request);
}
```

```java
getHandlerInternal(request)
{
	Object handler = lookupHandler(lookupPath, request);
    {
        //handlerMap中存放的就是controller的requestMapping注解的url-->method映射
        Object handler=this.handlerMap.get(urlPath);
    }
    //将handler封装成一个执行链,并加入了拦截器,用户自定义的拦截器也是加入这个对象中
    handler = buildPathExposingHandler(rawHandler, lookupPath, lookupPath, null);
}
```

