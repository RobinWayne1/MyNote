# SpringBoot自动配置原理

SpringBoot是依赖于全局配置文件application.properties以配置Bean信息的，那么application.properties中的配置是如何在SpringBoot中生效的呢？这就SpringBoot的启动类注解`@SpringBootApplication`有关系了。

```java
@SpringBootApplication
public class TwitterApplication
{

    public static void main(String[] args)
    {
        SpringApplication.run(TwitterApplication.class, args);
    }
}
```

看看这个注解的定义:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
      @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {}
```

可以看到有一个`@EnableAutoConfiguration`注解,继续看该注解的定义:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {}
```

注意看`@Import`注解,该注解导入的`AutoConfigurationImportSelector`内部将会在SpringBoot启动时扫描所有具有==**META-INF/spring.factories**==文件的jar包(包括类路径,所以自定义的starter就需要这个文件)。

这个spring.factories文件也是一组一组的key=value的形式，其中一个key是`EnableAutoConfiguration`类的全类名，而它的value是一个`xxxxAutoConfiguration`的类名的列表，这些类名以逗号分隔，如下图所示：

![](E:\Typora\MyNote\resources\Spring\spring.factories.png)

### 一、根据条件启用starter

这些就是所谓的stater,说人话其实也就是一些特殊的Spring配置文件。选一个看看：

```java
@Configuration
@ConditionalOnClass({ EnableAspectJAutoProxy.class, Aspect.class, Advice.class,
      AnnotatedElement.class })
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

   @Configuration
   @EnableAspectJAutoProxy(proxyTargetClass = false)
   @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = false)
   public static class JdkDynamicAutoProxyConfiguration {

   }

   @Configuration
   @EnableAspectJAutoProxy(proxyTargetClass = true)
   @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = true)
   public static class CglibAutoProxyConfiguration {

   }

}
```

先来讲几个比较重要的注解：

| @ConditionalOnBean         | 当容器里有指定的bean的条件下。                               |
| -------------------------- | ------------------------------------------------------------ |
| @ConditionalOnMissingBean  | 当容器里不存在指定bean的条件下。                             |
| @ConditionalOnClass        | 当类路径下有指定类的条件下。                                 |
| @ConditionalOnMissingClass | 当类路径下不存在指定类的条件下。                             |
| @ConditionalOnProperty     | 指定的属性是否有指定的值，比如@ConditionalOnProperties(prefix=”xxx.xxx”, value=”enable”, matchIfMissing=true)，代表当xxx.xxx为enable时条件的布尔值为true，如果没有设置的情况下也为true。 |

这几个注解使得`Configuration`可以根据注解条件是否符合来决定是否启用，这样也就达到了根据条件而创建某一类Bean的功能,而这也是自定义stater的核心。

再来看上面的`AopAutoConfiguration`,它的`@ConditionalOnProperty`表示若在全局配置有`spring.aop.auto=true`的配置,则启用该Configuration。**要注意的是，这几个注解可以同时用在方法体和类上。**

### 二、Spring管理配置Bean

然而看完上面的注解有没有发现少了什么？是的，配置的作用是给Bean使用的而不仅限于stater判断是否启用模块，那么怎么讲全局配置传给Bean让其个性初始化呢？所以还需要了解两个注解。

#### Ⅰ、`@ConfigurationProperties`

被`@ConfigurationProperties`声明的类都将被SpringBoot当成是配置信息存储类**,==在该类被注册进SpringBoot后==,SpringBoot会根据配置信息存储类的`setXXX()`方法与全局配置文件中配置的一一对应关系,将配置信息注入进配置信息存储类中,使得配置以SpringBean的形式存在,==此时就可以在任意地方进行依赖注入了。==**

先来看看我的德鲁伊配置

```java
@Configuration
public class DruidConfig
{

    @ConfigurationProperties(prefix = "spring.datasource")
    @Bean
    public DataSource druid()
    {
        return new DruidDataSource();
    }
}
```

在这里`@ConfigurationProperties`指定了全局配置前缀是`"spring.datasource"`,点进`DruidDataSource`可以看到一些如`initialSize`、`minIdle`等成员变量,此时我们就可以在全局配置文件中以`spring.datasource.initialSize=xxx`的形式指定具体的配置。

这是其中一种配置信息存储类为SpringBean的方式。

#### Ⅱ、`@EnableConfigurationProperties(XXXProperties.class)`

还有一种配置方式就是使用`@EnableConfigurationProperties`注解。该注解在Configuration类上标识，作用是将其后指定的类转化为SpringBean交由SpringBoot管理,并对其进行配置信息的注入。但是前提是`@ConfigurationProperties`要标识在相应的配置信息存储类，不然的话无法注入（说的就是`DruidDataSource`）。

### 三、自定义stater

**自定义stater的作用其实就是将整个业务拆成一个个模块，最后将各个模块以maven JAR包的形式整合进主模块中。Starter就是主模块和子模块的桥梁，它的作用就是启用这些导入的子模块，将这些子模块加入进主模块的Spring中进行管理。在starter内部的逻辑可以根据配置选择是否启用某个模块（也就是是否加载某个Configuration中的bean）。这样将模块拆开更能够方便业务扩展。**

自定义stater的具体逻辑其实就是用上面所讲的几个注解，自定义启用的条件。除此之外还需要在resources文件夹下面新建一个`META-INF/spring.factories`将这个starter写进其中，因为以JAR包的形式导入的starter要经过`AutoConfigurationImportSelector`扫描后才能够被其他模块启用。

