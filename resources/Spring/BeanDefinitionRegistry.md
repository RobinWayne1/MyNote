# BeanDefinitionRegistry

先复制一篇别人的,有空整理

### 前言

Spring Framework最重要的一个概念就是Bean，而Spring Bean定义是需要扫描、注册来实现统一的管理的。

前面已经介绍了Spring容器的启动过程、分类、Bean定义信息的详解等。但是发现有读者留言问了Bean定义注册中心得一些问题，因此本文主要是讲解BeanDefinitionRegistry

BeanDefinitionRegistry
BeanDefinitionRegistry是一个接口， 实现了AliasRegistry接口， 定义了一些对 bean的常用操作。

它大概有如下功能：

以Map<String, BeanDefinition>的形式注册bean
根据beanName 删除和获取 beanDefiniation
得到持有的beanDefiniation的数目
根据beanName 判断是否包含beanDefiniation

```java
// 它继承自 AliasRegistry 
public interface BeanDefinitionRegistry extends AliasRegistry {

// 关键 -> 往注册表中注册一个新的 BeanDefinition 实例 
void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException;
// 移除注册表中已注册的 BeanDefinition 实例
void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
// 从注册中心取得指定的 BeanDefinition 实例
BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
// 判断 BeanDefinition 实例是否在注册表中（是否注册）
boolean containsBeanDefinition(String beanName);

// 取得注册表中所有 BeanDefinition 实例的 beanName（标识）
String[] getBeanDefinitionNames();
// 返回注册表中 BeanDefinition 实例的数量
int getBeanDefinitionCount();
// beanName（标识）是否被占用
boolean isBeanNameInUse(String beanName);
}
```

下面看看它的继承体系：

![](E:\Typora\resources\Spring\BeanDefinitionRegistry.png)

可以看出，它的默认实现类，主要有三个：`SimpleBeanDefinitionRegistry`、`DefaultListableBeanFactory`、`GenericApplicationContext`

#### SimpleBeanDefinitionRegistry
是默认的一个实现方式，也是一个非常简单的实现。存储用的是ConcurrentHashMap ，可以保证线程安全

```java
// @since 2.5.2可以看到提供得还是比较晚的
// AliasRegistry和SimpleAliasRegistry都是@since 2.5.2之后才有的~~~
public class SimpleBeanDefinitionRegistry extends SimpleAliasRegistry implements BeanDefinitionRegistry {
	
// 采用的ConcurrentHashMap来存储注册进来的Bean定义信息~~~~
/** Map of bean definition objects, keyed by bean name */
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(64);
... // 各种实现都异常的简单，都是操作map
// 这里只简单说说这两个方法

// 需要注意的是：如果没有Bean定义，是抛出的异常，而不是返回null这点需要注意
@Override
public BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
	BeanDefinition bd = this.beanDefinitionMap.get(beanName);
	if (bd == null) {
		throw new NoSuchBeanDefinitionException(beanName);
	}
	return bd;
}
// beanName是个已存在的别名，或者已经包含此Bean定义了，那就证明在使用了嘛
// 它比单纯的containsBeanDefinition()范围更大些~~~
@Override
public boolean isBeanNameInUse(String beanName) {
	return isAlias(beanName) || containsBeanDefinition(beanName);
}
}
```
#### DefaultListableBeanFactory
该类是 BeanDefinitionRegistry 接口的基本实现类，但同时也实现其他了接口的功能，这里只探究下其关于注册 BeanDefinition 实例的相关方法。

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
	...
	/** Map of bean definition objects, keyed by bean name */
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
	// 保存所有的Bean名称
	/** List of bean definition names, in registration order */
	private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
	
    // 注册Bean定义信息~~
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
		throws BeanDefinitionStoreException {
	...
	BeanDefinition oldBeanDefinition = this.beanDefinitionMap.get(beanName);
	// 如果不为null，说明这个beanName对应的Bean定义信息已经存在了~~~~~
	if (oldBeanDefinition != null) {
		// 是否允许覆盖（默认是true 表示允许的）
		if (!isAllowBeanDefinitionOverriding()) {
			// 抛异常
		}
		// 若允许覆盖  那还得比较下role  如果新进来的这个Bean的role更大			
		// 比如老的是ROLE_APPLICATION（0）  新的是ROLE_INFRASTRUCTURE(2) 
		// 最终会执行到put，但是此处输出一个warn日志，告知你的bean被覆盖啦~~~~~~~（我们自己覆盖Spring框架内的bean显然就不需要warn提示了）
		else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
			// 仅仅输出一个logger.warn
		}
		// 最终会执行put，但是内容还不相同  那就提醒一个info信息吧
		else if (!beanDefinition.equals(oldBeanDefinition)) {
			// 输出一个info信息
		}
		else {
			// 输出一个debug信息
		}
		// 最终添加进去 （哪怕已经存在了~） 
		// 从这里能看出Spring对日志输出的一个优秀处理，方便我们定位问题~~~
		this.beanDefinitionMap.put(beanName, beanDefinition);
		// 请注意：这里beanName并没有再add了，因为已经存在了  没必要了嘛
	}
	else {
		// hasBeanCreationStarted:表示已经存在bean开始创建了（开始getBean()了吧~~~）
		if (hasBeanCreationStarted()) {
			// 注册过程需要synchronized，保证数据的一致性	synchronized (this.beanDefinitionMap) {
				this.beanDefinitionMap.put(beanName, beanDefinition); // 放进去
				List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
				updatedDefinitions.addAll(this.beanDefinitionNames);
				updatedDefinitions.add(beanName);
				this.beanDefinitionNames = updatedDefinitions;
		
				// 
				if (this.manualSingletonNames.contains(beanName)) {
					Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
					updatedSingletons.remove(beanName);
					this.manualSingletonNames = updatedSingletons;
				}
			}
		}
		else {
			// Still in startup registration phase
			// 表示仍然在启动  注册的状态~~~就很好处理了 put仅需，名字add进去
			this.beanDefinitionMap.put(beanName, beanDefinition);
			this.beanDefinitionNames.add(beanName);
			
			// 手动注册的BeanNames里面移除~~~ 因为有Bean定义信息了，所以现在不是手动直接注册的Bean单例~~~~
			this.manualSingletonNames.remove(beanName);
		}

		// 这里的意思是：但凡你新增了一个新的Bean定义信息，之前已经冻结的就清空呗~~~
		this.frozenBeanDefinitionNames = null;
	}
	
	// 最后异步很有意思：老的bean定义信息不为null（beanName已经存在了）,或者这个beanName直接是一个单例Bean了~
	if (oldBeanDefinition != null || containsSingleton(beanName)) {
		// 做清理工作：
		// clearMergedBeanDefinition(beanName)
		// destroySingleton(beanName);  销毁这个单例Bean  因为有了该bean定义信息  最终还是会创建的
		// Reset all bean definitions that have the given bean as parent (recursively).  处理该Bean定义的getParentName  有相同的也得做清楚  所以这里是个递归
		resetBeanDefinition(beanName);
	}
}

@Override
public void removeBeanDefinition(String beanName) throws 		NoSuchBeanDefinitionException {
	// 移除整体上比较简单：beanDefinitionMap.remove
	// beanDefinitionNames.remove
	// resetBeanDefinition(beanName);

	BeanDefinition bd = this.beanDefinitionMap.remove(beanName);
	// 这里发现移除，若这个Bean定义本来就不存在，事抛异常，而不是返回null 需要注意~~~~
	if (bd == null) {
		throw new NoSuchBeanDefinitionException(beanName);
	}

	if (hasBeanCreationStarted()) {
		synchronized (this.beanDefinitionMap) {
			List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames);
			updatedDefinitions.remove(beanName);
			this.beanDefinitionNames = updatedDefinitions;
		}
	} else {
		this.beanDefinitionNames.remove(beanName);
	}
	this.frozenBeanDefinitionNames = null;
	resetBeanDefinition(beanName);
}

// 这个实现非常的简单，直接从map里拿
@Override
public BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
	BeanDefinition bd = this.beanDefinitionMap.get(beanName);
	if (bd == null) {
		throw new NoSuchBeanDefinitionException(beanName);
	}
	return bd;
}
@Override
	public boolean containsBeanDefinition(String beanName) {
		return this.beanDefinitionMap.containsKey(beanName);
	}
	@Override
	public int getBeanDefinitionCount() {
		return this.beanDefinitionMap.size();
	}
	@Override
	public String[] getBeanDefinitionNames() {
		String[] frozenNames = this.frozenBeanDefinitionNames;
		if (frozenNames != null) {
			return frozenNames.clone();
		}
		else {
			return StringUtils.toStringArray(this.beanDefinitionNames);
		}
	}
	// ==========这个方法非常有意思：它是间接的实现的===============
// 因为BeanDefinitionRegistry有这个方法，而它的父类AbstractBeanFactory也有这个方法，所以一步小心，就间接的实现了这个接口方法
public boolean isBeanNameInUse(String beanName) {
	// 增加了一个hasDependentBean(beanName);  或者这个BeanName是依赖的Bean 也会让位被使用了
	return isAlias(beanName) || containsLocalBean(beanName) || hasDependentBean(beanName);
}
}
```


这就是我们最总要的一个Bean工厂实现：DefaultListableBeanFactory它对Bean定义注册中心接口的实现逻辑。

##### XmlBeanFactory
它是DefaultListableBeanFactory的子类。Spring3.1之后建议用DefaultListableBeanFactory代替

```java
@Deprecated
@SuppressWarnings({"serial", "all"})
public class XmlBeanFactory extends DefaultListableBeanFactory {
	private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
// 此处resource资源事传一个xml的文件资源~~~~
// parentBeanFactory：父工厂~~~
public XmlBeanFactory(Resource resource) throws BeansException {
	this(resource, null);
}
public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
	super(parentBeanFactory);
	this.reader.loadBeanDefinitions(resource);
}
}
```

##### GenericApplicationContext
这是一个通用的Spring上下文了，之前也说到了，它基本什么都得靠自己手动来处理：

```java
public static void main(String[] args) {
    GenericApplicationContext ctx = new GenericApplicationContext();
    //手动使用XmlBeanDefinitionReader
    XmlBeanDefinitionReader xmlReader = new XmlBeanDefinitionReader(ctx);
    //加载ClassPathResource
    xmlReader.loadBeanDefinitions(new ClassPathResource("spring.xml"));

    //=========我们可议执行多次loadBeanDefinitions，从不同的地方把Bean定义信息给拿过来，最后再启动容器即可==============
    //// 备注：此处的bean都采用properties的方式去配置（基本没人会这么使用了）
    //PropertiesBeanDefinitionReader propReader = new PropertiesBeanDefinitionReader(ctx);
    //propReader.loadBeanDefinitions(new ClassPathResource("spring.properties"));

    //调用Refresh方法 启动容器（这个都需要手动  哈哈）
    ctx.refresh();

    //和其他ApplicationContext方法一样的使用方式
    System.out.println(ctx.getBean("person")); //com.fsx.bean.Person@1068e947
}
```

spring.xml如下：

```xml
<bean id="person" class="com.fsx.bean.Person"/>
```

因为是手动档，对API的使用有一定的门槛，因此我们一般情况下不会直接使用它。但是他有两个子类我们是比较熟悉的

##### GenericXmlApplicationContext
利用XML来配置Bean的定义信息，借助XmlBeanDefinitionReader去读取~~~

```java
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {
// 这个很重要：它持有一个DefaultListableBeanFactory的引用
private final DefaultListableBeanFactory beanFactory;

// 所以下面所有的 BeanDefinitionRegistry 相关的方法都是委托给beanFactory去做了的，因此省略~~~
}
AbstractApplicationContext最终也是委托给getBeanFactory()去做这些事。而最终的实现都是它：DefaultListableBeanFactory
备注：AnnotationConfigWebApplicationContext、XmlWebApplicationContext都继承自AbstractRefreshableConfigApplicationContext，从而也最终继承了AbstractApplicationContext
```
##### AnnotationConfigApplicationContext
注解驱动去扫描Bean的定义信息。先用`ClassPathBeanDefinitionScanner`把文件都扫描进来，然后用`AnnotatedBeanDefinitionReader`去load没有里面的Bean定义信息。

说明：由上面的继承关系可以看出，上文顶层接口`ApplicationContext`是不提供直接注册Bean定义信息、删除等一些操作的。因为`ApplicationContext`它并没有实现此接口`BeanDefinitionRegistry`，但是它继承了接口`ListableBeanFactory`，所以它有如下三个能力：

```java
boolean containsBeanDefinition(String beanName);
int getBeanDefinitionCount();
String[] getBeanDefinitionNames();
```

ApplicationContext体系里只有子体系GenericApplicationContext下才能直接操作注册中心。比如常用的：GenericXmlApplicationContext和AnnotationConfigApplicationContext

### 手动注册BeanDefinition（编程方式注册Bean定义）
手动注册bean的两种方式：

1. 实现ImportBeanDefinitionRegistrar
2. 实现BeanDefinitionRegistryPostProcessor

第一个不说了，之前讲import时候讲解过。
`BeanDefinitionRegistryPostProcessor`它继承自`BeanFactoryPostProcessor`，提供方法：

```java
// @since 3.0.1 它出现得还是比较晚的
// 父类BeanFactoryPostProcessor  第一版就有了
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
	// 这里可以通过registry，我们手动的向工厂里注册Bean定义信息
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```

>
> 有个重要实现类;ConfigurationClassPostProcessor，它是来处理@Configuration配置文件的。它最终就是解析配置文件里的@Import、@Bean等，然后把定义信息都注册进去~~~

