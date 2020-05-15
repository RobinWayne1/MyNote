~~~java
@EnableAutoConfiguration
{
	@Import(AutoConfigurationImportSelector.class)
	{
		public String[] selectImports(AnnotationMetadata annotationMetadata) 
		{
			//得到候选配置
			List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
		}
	}
}
~~~

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) 
{
    //其中getSpringFactoriesLoaderFactoryClass()方法得到的是EnableAutoConfiguration.class,即从下面所讲的properties中获取EnableAutoConfiguration.class（类名）对应的值添加到容器中（看下图）
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
				getBeanClassLoader());
    	{
            
            MultiValueMap<String, String> result = cache.get(classLoader);
            //扫描jar包类路径下的"META-INF/spring.factories"文件
        	Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
            //把扫描到的文件内容包装成properties对象
              ...
            //提取properties对象中的classname属性
             result.add(factoryClassName, factoryName.trim());     
    	}
		return configurations;
}

```

![](E:\Typora\resources\Spring\微信截图_20190901215842.png)

[^图1]: 

以HttpEncodingAutoConfiguration为例：

```java
@Configuration
//将配置文件中特定的值与HttpProperties绑定起来，添加到ioc容器中，使得在创建bean时可以从容器中拿
@EnableConfigurationProperties(HttpProperties.class)
//spring底层注解，根据不同条件，满足指定条件该配置类才会生效（这里是判断是否是web应用，如果是则配置生效）
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
//判断容器中有没有CharacterEncodingFilter.class
@ConditionalOnClass(CharacterEncodingFilter.class)
//判断application.properties中有没有spring.http.encoding.enabled(.true)
@ConditionalOnProperty(prefix = "spring.http.encoding", value = "enabled", matchIfMissing = true)
public class HttpEncodingAutoConfiguration 
{
    //当上述注解条件满足时配置文件则生效，则开始创建bean
   
    @Bean
    //如果缺少这个bean，则创建这个bean添加到容器中
	@ConditionalOnMissingBean
	public CharacterEncodingFilter characterEncodingFilter() {
		CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
        //从application.properties里获取值放进bean的属性中
		filter.setEncoding(this.properties.getCharset().name());
		filter.setForceRequestEncoding(this.properties.shouldForce(Type.REQUEST));
		filter.setForceResponseEncoding(this.properties.shouldForce(Type.RESPONSE));
		return filter;
	}
}
```

```java
//根据前缀为spring.http取出值放入HttpProperties类中
@ConfigurationProperties(prefix = "spring.http")
public class HttpProperties {}
```

首先将类路径下的 META-INF/spring.factories中的自动配置组件(autoconfigration)全部在容器中创建**(图1)**(根据不同的条件判断这些组件是否要被启用),然后不同的这些组件的Properties对象再根据application.properties的前缀(server,spring.http.encoding)将与自身相同的前缀扫描进来,放进自己的properties对象中==(如HttpEncodingAutoConfigration的properties对象是HttpEncodingProperties)==,Properties对象则会被放进容器中,供autoconfigration组件创建bean时使用这些属性(比如在HttpEncodingAutoConfigration中创建的springmvc的过滤器就要用到charset属性,就要用到Properties对象中的属性)

autoconfigration组件的作用就是根据开发人员的不同需求而根据springboot的配置创建一系列所需要的bean以供使用

