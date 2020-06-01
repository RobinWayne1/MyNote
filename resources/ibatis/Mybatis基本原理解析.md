# Mybatis基本原理解析

接下来我将从两个角度讲解:一个是单独的Mybatis框架运行角度,一个是与Spring整合的Mybatis运行角度。

## 一、单独的Mybatis工作流程

先来看一个传统的单独Mybatis框架工作demo

```java
public static void main(String[]args){
        SqlSessionFactory ssf = new SqlSessionFactoryBuilder().build(new FileInputStream("mybatis-config.xml"));
        SqlSession sqlSession = ssf.openSession();
   		String params="传记";
    	List<User> list = sqlSession.selectList("com.robin.mapper.SubtitleMapper.selectVideoByType",params);
}
```

1. 首先通过`SqlSessionFactoryBuilder`的`build()`方法将mybatis配置文件读取进来,并将其包装成`Configuration`对象,并创建一个`SqlSessionFactory`对象持有这个`Configuration`对象
2. 由`SqlSessionFactory`创建`SqlSession` 对象,`SqlSession`的概念后面将会详细讲
3. 调用`SqlSession`中的api，传入`Statement Id`和参数，内部进行复杂的处理，最后调用jdbc执行SQL语句，封装结果返回。

### Ⅰ、`SqlSessionFactoryBuilder.build()`

```java
// 1.我们最初调用的build
public SqlSessionFactory build(InputStream inputStream) {
	//调用了重载方法
    return build(inputStream, null, null);
  }

// 2.调用的重载方法
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      //  XMLConfigBuilder是专门解析mybatis的配置文件的类
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      //这里又调用了一个重载方法。parser.parse()的返回值是Configuration对象
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } //省略部分代码
  }
//将Configuration封装进SqlSessionFactory中
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```

==`Configuration`对象的结构和xml配置文件的对象几乎相同,除此之外更重要的是`Configuration`在解析`<mappers>`标签时还会对mapper文件进行解析一并放入`Configuration`对象中。==

```java
//在创建XMLConfigBuilder时，它的构造方法中解析器XPathParser已经读取了配置文件
//3. 进入XMLConfigBuilder 中的 parse()方法。
public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    //parser是XPathParser解析器对象，读取节点内数据，<configuration>是MyBatis配置文件中的顶层标签
    parseConfiguration(parser.evalNode("/configuration"));
    //最后返回的是Configuration 对象
    return configuration;
}

//4. 进入parseConfiguration方法

private void parseConfiguration(XNode root) {
    try {
      //读取mybatis配置文件的各个标签内容并封装到Configuration中的属性中。
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      //⭐注意看此处,这里将会对在mybatis声明的mapper文件进行解析
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}

```

`build()`方法其实就是对配置文件的解析,没有什么逻辑可探究,但这里有一个比较重要的知识点是`MappedStatement`。

#### ①、`MappedStatement`

`MappedStatement`与`Mapper`配置文件中的一个`select/update/insert/delete`节点相对应。`MappedStatement`对象在上面所说的`mapperElement()`方法解析mapper的过程中被初始化,经过这个方法后mapper中的增删改查语句的信息都会被封装到`MappedStatement`里并存放到`Configuration`里的一个map中。

```java
//Configuration.java
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")
```

`MappedStatement`除了有描述一条SQL语句的作用外,**更重要的是这个对象持有着二级缓存对象`Cache`**。

### Ⅱ、`SqlSessionFactory.openSession()`

```java
//DefaultSqlSessionFactory
public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }

  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
        //事务相关,由于后面会讲一下Spring事务与mybatis的整合,所以这里就不讲mybatis的事务了
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        //根据配置文件参数创建指定类型的Executor
      final Executor executor = configuration.newExecutor(tx, execType);
        //将Configuration对象和Executor都放入SqlSession中
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

同样没有什么具体的逻辑,就是创建了一个`SqlSession`对象。

### Ⅲ、⭐`SqlSession`架构

由于即将进入最核心的`SqlSession`,这里先带读者了解`SqlSession`的架构，读源码的最终目的就是将这张图完全理解透。



<img src="E:\Typora\MyNote\resources\ibatis\SqlSession架构.png" style="zoom:75%;" />



* **`SqlSession`:**`SqlSession`是作为用户代码与数据库的顶层接口,这个对象提供了传统的如`selectOne()`、`selectList()`、`insert()`等CRUD方法以让用户操作数据库。
* **`Executor`:**`SqlSession`是通过`Executor`操作数据库的。如`Executor.query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler)`。从参数中可以看到，`SqlSession`最重要的工作其实是从`Configuration`中取出封装了SQL信息的`MappedStatement`交给`Executor`去执行。有三种基本`Executor`和一种特殊`Executor`实现

  * **`SimpleExecutor`：**每执行一次`update`或`select`，就开启一个`Statement`对象，用完立刻关闭`Statement`对象。
  * **`ReuseExecutor`：**执行`update`或`select`，以sql作为key查找`Statement`对象，**存在就使用，不存在就创建，用完后，不关闭Statement对象，而是放置于Map内，供下一次使用。**简言之，就是重复使用`Statement`对象。
  * **`BatchExecutor`：**在事务中执行update时（没有select，JDBC批处理不支持select），`BatchExecutor`将所有sql都添加到`Statement`批处理中（即调用`stmt.addBatch()`），等待统一执行（==调用`stmt.executeBatch()`,而`BatchExecutor`将会在进行`commit()`时调用此方法，也会在当前Executor第一次执行`doQuery()`时调用此方法==），它缓存了多个`Statement`对象，每个`Statement`对象都是`addBatch()`完毕后，等待逐一执行`executeBatch()`批处理的对象。注意`BatchExecutor`并没有像`ReuseExecutor`那样用Map缓存`Statement`对象,它缓存的只是调用了`addBatch()`的`Statement`。
  * **`CacheExecutor`:**这个`Executor`属于一个装饰器,负责管理二级缓存,但是最终调用都会来到上面三个`delegate(代表)`中。之后将会有大量篇幅介绍缓存。
* **`StatementHandler`**:由于JDBC有三种`Statement`实现:`Statement`(静态SQL)、`Preparetatement`(支持动态参数)、`CallableStatement`(访问数据库`存储过程`的时候使用，也接受动态参数)。而`StatementHandler`就是要根据用户编写的Mapper的`statementType`来选择使用哪个`Statement`来访问数据库。除此之外他还要管理`ParameterHandler`和`ResultSetHandler`,分别用于`Statement`参数绑定 与 将`ResultSet`转换成POJO类
* **`ParameterHandler`:**这个接口的作用其实就是对用户调用时传入的参数值解析出与`PrepareStatement`的?一一对应的关系,最后则通过`TypeHandler`调用`PrepareStatement.setObject()`设置参数值。
* **`ResultSetHandler`:**这个接口的作用就是处理`Statement`产生的结果集和存储过程执行后产生的输出参数,将被`TypeHandler`处理过类型的结果集映射到POJO中

这里提一嘴,上面的四大组件都支持插件动态代理,有机会讲一下。

* **`TypeHandler`:**用于实现java类型和JDBC类型的相互转换.mybatis使用`prepareStatement`来进行参数设置的时候,需要通过`TypeHandler`将传入的java参数设置成合适的jdbc类型参数,这个过程实际上是通过调用`PrepareStatement`不同的set方法实现的;在获取结果返回之后,也需要将返回的结果转换成我们需要的java类型,这时候是通过调用`ResultSet`对象不同类型的get方法时间的

### 四、






































由此可以看出在这里先简单介绍一下`SqlSession`和`Executor`的概念。

`SqlSession`是应用程序与数据库交互的顶层对象，它有两个重要的成员变量。

```java
  private final Configuration configuration;
  private final Executor executor;
```

在调用



> 参考资料
>
> 三种Executor源自:
>
> 1. https://www.iteye.com/blog/elim-2353672
> 2. https://www.jianshu.com/p/96ddaec4aea7]
>
> StatementHandler源自:
>
> https://www.jianshu.com/p/6346ce3f3567
>
> ParameterHandler源自:
>
> https://www.jianshu.com/p/39debc2ee0f2
>
> ResultSetHandler源自：
>
> https://blog.csdn.net/qq924862077/article/details/52704191
>
> TypeHandler源自：
>
> https://blog.csdn.net/jokemqc/article/details/81326109