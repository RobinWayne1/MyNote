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



* **`SqlSession`:**`SqlSession`是作为用户代码与数据库的顶层接口,这个对象提供了传统的如`selectOne()`、`selectList()`、`insert()`等CRUD方法以让用户操作数据库。其内部的逻辑其实比较简单，其实就是将`MappedStatement`从`Configuration`中取出,然后交给`Executor`执行。
* **`Executor`:**`SqlSession`是通过`Executor`操作数据库的。如`Executor.query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler)`。从参数中可以看到，`SqlSession`最重要的工作其实是从`Configuration`中取出封装了SQL信息的`MappedStatement`交给`Executor`去执行。有三种基本`Executor`和一种特殊`Executor`实现

  * **`SimpleExecutor`：**每执行一次`update`或`select`，就开启一个`Statement`对象，用完立刻关闭`Statement`对象。
  * **`ReuseExecutor`：**执行`update`或`select`，以sql作为key查找`Statement`对象，**存在就使用，不存在就创建，用完后，不关闭Statement对象，而是放置于Map内，供下一次使用。**简言之，就是重复使用`Statement`对象。
  * **`BatchExecutor`：**在事务中执行update时（没有select，JDBC批处理不支持select），`BatchExecutor`将所有sql都添加到`Statement`批处理中（即调用`stmt.addBatch()`），等待统一执行（==调用`stmt.executeBatch()`,而`BatchExecutor`将会在进行`commit()`时调用此方法，也会在当前Executor第一次执行`doQuery()`时调用此方法==），它缓存了多个`Statement`对象，每个`Statement`对象都是`addBatch()`完毕后，等待逐一执行`executeBatch()`批处理的对象。注意`BatchExecutor`并没有像`ReuseExecutor`那样用Map缓存`Statement`对象,它缓存的只是调用了`addBatch()`的`Statement`。
  * **`CacheExecutor`:**这个`Executor`属于一个装饰器,负责管理二级缓存,但是最终调用都会来到上面三个`delegate(代表)`中。之后将会有大量篇幅介绍缓存。
* **`StatementHandler`**:由于JDBC有三种`Statement`实现:`Statement`(静态SQL)、`Preparetatement`(支持动态参数)、`CallableStatement`(访问数据库`存储过程`的时候使用，也接受动态参数)。而`StatementHandler`就是要根据用户编写的Mapper的`statementType`来选择实例化哪个`Statement`来访问数据库。除此之外他还要管理`ParameterHandler`和`ResultSetHandler`,分别用于`Statement`参数绑定 与 将`ResultSet`转换成POJO类
* **`ParameterHandler`:**这个接口的作用其实就是对用户调用时传入的参数值解析出与`PrepareStatement`的?一一对应的关系,最后则通过`TypeHandler`调用`PrepareStatement.setObject()`设置参数值。
* **`ResultSetHandler`:**这个接口的作用就是处理`Statement`产生的结果集和存储过程执行后产生的输出参数,将被`TypeHandler`处理过类型的结果集映射到POJO中

这里提一嘴,上面的四大组件都支持插件动态代理,有机会讲一下。

* **`TypeHandler`:**用于实现java类型和JDBC类型的相互转换.mybatis使用`prepareStatement`来进行参数设置的时候,需要通过`TypeHandler`将传入的java参数设置成合适的jdbc类型参数,这个过程实际上是通过调用`PrepareStatement`不同的`set()`方法实现的;在获取结果返回之后,也需要将返回的结果转换成我们需要的java类型,这时候是通过调用`ResultSet`对象不同类型的`get()`方法时间的

### 四、`SqlSession.selectList()`

#### ①、`SqlSession`组件

终于来到`SqlSession`了,接下来就看它的`selectList()`方法。

```java
//DefaultSqlSession.java
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
      //从Configuration取出MappedStatement
    MappedStatement ms = configuration.getMappedStatement(statement);
      //这里先不讲二级缓存。三种Executor都有一个共同的抽象父类BaseExecutor，这个query()方法就在此类中
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

就如上面架构图中曾讲述的,`SqlSession`将查询委托给了`Executor`，`SqlSession`将根据用户不同的业务逻辑调用从而以不同的方式操控`Executor`,如调用`SqlSession.selectList()`则会调用`executor.query()`;调用`SqlSession.update()/delete()/insert()`则会调用`executor.update()`。

#### ②、`Executor`组件

```java
//BaseExecutor.java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    //实例化一个BoundSql.
  BoundSql boundSql = ms.getBoundSql(parameter);
    //一二级缓存的key,我将另开一篇详细讲缓存
  CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    //调用重载方法
  return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

> 其中`BoundSql`是一个比较重要的对象。他的一些成员变量如下所示：
> ```java
> //该成员变量就是具体的带?的SQL语句,这个SQL语句是判断完成之后的动态SQL
> private final String sql;
> //parameterMappings存放着参数映射关系,也就是需要绑定参数对象的字段如v_type
> private final List<ParameterMapping> parameterMappings;
> //这个就是从getBoundSql()传进来的parameterObject,代表参数对象.若使用@Param指定参数,则存放类型会变成一个Map:{{v_type→剧情},{param1→剧情},{v_name→蝙蝠侠},{param2→蝙蝠侠}}
> private final Object parameterObject;
> ```
>
> 该对象将会在`ParameterHandler`中发挥重要作用

`query(ms, parameter, rowBounds, resultHandler, key, boundSql)`中有两个调用栈**都是用来维护延迟加载和一级缓存**的,我们跳过他们,直接进入`BaseExecutor`的实现类`SimpleExecutor`的方法。

```java
//SimpleExecutor.java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
        //⭐注意看这里,又是一个重要组件的创建.
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        //⭐得到相应的Statement实例
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```

先进入到`newStatementHandler()`中看看Mybatis是怎么决定创建哪个`StatementHandler`的。

#### ③、`StatementHandler`组件

##### 1、`configuration.newStatementHandler()`

```java
//Configuration.java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    //创建了一个RoutingStatementHandler()实例
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    //StatementHandler是四大组件之一，支持插件动态代理。pluginAll()方法就是来应用用户实现的插件的,有点类似于Spring的后处理器
  statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}

//RoutingStatementHandler.java
public class RoutingStatementHandler implements StatementHandler {
 //本体
  private final StatementHandler delegate;

  public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
	//⭐在这里就根据MappedStatement的配置信息(在Mapper中以<select statementType="PREPARED">形式存在)选择创建哪个Statement对象
    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }

  }
//装饰者模式
  @Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    return delegate.prepare(connection, transactionTimeout);
  }

  @Override
  public void parameterize(Statement statement) throws SQLException {
    delegate.parameterize(statement);
  }
    
}
```

可以看到,`RoutingStatementHandler`其实是一个装饰者,包装了真正需要调用的`StatementHandler`。越看下去你就越会发现，Mybatis就是由装饰者模式和动态代理模式构成的。

##### 2、`prepareStatement()`

回到`doQuery()`方法,看看`prepareStatement()`是怎样得到`Statement`实例的。

```java
//SimpleExecutor.java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
    //从PooledDataSource中获取一个Connection,有空再讲它的池化设计(https://my.oschina.net/mengyuankan/blog/2664784).
    //PooledDataSource是Mybatis自带的数据源,而市面上有许多第三方数据源如阿里的druid，Apache的DBCP，c3p0
  Connection connection = getConnection(statementLog);
    //调用RoutingStatementHandler的prepare(),装饰着最终将调用BaseStatementHandler的prepare()
  stmt = handler.prepare(connection, transaction.getTimeout());
   //应用ParameterHandler组件
  handler.parameterize(stmt);
  return stmt;
}
```

```java
//BaseStatementHandler.java
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
        //这是一个抽象方法,由实现类PrepareStatementHandler实现
      statement = instantiateStatement(connection);
      setStatementTimeout(statement, transactionTimeout);
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }

//PrepareStatementHandler.java
 protected Statement instantiateStatement(Connection connection) throws SQLException {
     //得到具体的SQL
    String sql = boundSql.getSql();
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
        //直接根据Connection创建一个PrepareStatement对象返回
      if (keyColumnNames == null) {
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() == ResultSetType.DEFAULT) {
      return connection.prepareStatement(sql);
    } else {
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    }
  }
```

至此得知,`prepareStatement()`利用了`RoutingStatementHandler`这个装饰者对调用的分派,从而实例化出三种类型的`Statement`对象。

##### 3、`handler.parameterize(stmt)`

在创建了`Statement`对象后,`Executor`组件内就开始应用`StatementHandler`管理的`ParameterHandler`组件。

经过`RoutingStatemengHandler`的分派后来到`PrepareStatementHandler`中。

```java
//BaseStatementHandler.java
//⭐可以看到,在实例化BaseStatementHandler的时候,其构造函数已经将剩余的三个组件都实例化完毕了
 protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
	//···省略了一部分代码
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
	
    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
  }


//PrepareStatementHandler.java
public void parameterize(Statement statement) throws SQLException {
    //使用父类成员parameterHandler
    parameterHandler.setParameters((PreparedStatement) statement);
  }

```

#### ④、`ParameterHandler`组件

##### 3.1、`parameterHandler.setParameters()`

```java
//DefaultParameterHandler.java
public void setParameters(PreparedStatement ps) {
  ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    //获得参数映射列表
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings != null) {
      //遍历参数列表逐个绑定参数
    for (int i = 0; i < parameterMappings.size(); i++) {
      ParameterMapping parameterMapping = parameterMappings.get(i);
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
          //获得字段名,如v_type
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } 
          //如果没有使用@Param或集合作为参数,则会进入这里
          else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          //使用了@Param或集合作为参数
          //这一步的工作就是从当前实际传入的参数中获取到指定key(v_type)的value(剧情)值
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
          //取出该参数对应的TypeHandler
        TypeHandler typeHandler = parameterMapping.getTypeHandler();
          //取出该参数对应的JdbcType
        JdbcType jdbcType = parameterMapping.getJdbcType();
        if (value == null && jdbcType == null) {
          jdbcType = configuration.getJdbcTypeForNull();
        }
        try {
            //重点⭐⭐:将javaType装换为JdbcType,并调用statement.setObject()设置参数值,该组件将
          typeHandler.setParameter(ps, i + 1, value, jdbcType);
        } catch (TypeException | SQLException e) {
          throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
        }
      }
    }
  }
}
```

#### ⑤、`ResultSetHandler`组件

继续看`doQuery()`方法,现在已经来到了最后一步——向数据库交互。

```java
//RoutingStatementHandler。java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  return delegate.query(statement, resultHandler);
}
//PrepareStatementHandler.java
 public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
     //开始应用ResultSetHandler组件将结果集转换POJO,在内部也会应用TypeHandler转换JdbcType
    return resultSetHandler.handleResultSets(ps);
  }
```

#### ⑥、`TypeHandler`组件

`TypeHandler`是唯一可供用户扩展的组件，他用做实现java类型和JDBC类型的相互转换。继续跟进`typeHandler.setParameter(ps, i + 1, value, jdbcType)`方法。

##### 1、`typeHandler.setParameter()`

先来了解`TypeHandler`的运行机制。由于`TypeHandler`的目的是将`javaType`转化为`JdbcType`,所以需要根据传入的参数类型从而选择对应的`TypeHandler`,而处理这个选择逻辑的就是`UnknownTypeHandler`,之后再由它来调用目标`TypeHandler.setParameter()`方法。

```java
//所有TypeHandler的抽象父类
//BaseTypeHandler.java
public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
  if (parameter == null) {
//省略代码
    try {
      ps.setNull(i, jdbcType.TYPE_CODE);
    }//省略代码
  } else {
    try {
       //⭐参数不为Null则会进入这里.这个方法是一个抽象方法,由子类实现,而首次的动态分派则会来到UnknownTypeHandler中
      setNonNullParameter(ps, i, parameter, jdbcType);
    } //省略代码
  }
}
//UnknownTypeHandler.java
 public void setNonNullParameter(PreparedStatement ps, int i, Object parameter, JdbcType jdbcType)
      throws SQLException {
     //这就是根据参数类型从而选择TypeHandler的逻辑.此时我的参数javaType是String,则返回的TypeHandler则是StringTypeHandler
    TypeHandler handler = resolveTypeHandler(parameter, jdbcType);
     //再次调用setParameter()方法来到BaseTypeHandler类,而最终又会分派到StringTypeHandler
    handler.setParameter(ps, i, parameter, jdbcType);
  }

//StringTypeHandler.java
 public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType)
      throws SQLException {
     //这就是我们要找的设置参数方法,就是这么简单的一行
    ps.setString(i, parameter);
  }
```



至此,Mybatis工作流程就讲完了。

总结：

## 二、用户扩展的接口

### Ⅰ、`TypeHandler`

直接上例子。

```java
@MappedJdbcTypes(JdbcType.VARCHAR)
@MappedTypes({List.class})
public class ListTypeHandler implements TypeHandler<List<String>> {
 
    @Override
    public void setParameter(PreparedStatement ps, int i, List<String> parameter, JdbcType jdbcType) throws SQLException {
        String hobbys = StringUtils.join(parameter, ",");
        try {
            ps.setString(i, hobbys);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
 
    @Override
    public List<String> getResult(CallableStatement cs, int columnIndex) throws SQLException {
        String hobbys = cs.getString(columnIndex);
        return Arrays.asList(hobbys.split(","));
    }
 
    @Override
    public List<String> getResult(ResultSet rs, int columnIndex) throws SQLException {
        return Arrays.asList(rs.getString(columnIndex).split(","));
    }
 
    @Override
    public List<String> getResult(ResultSet rs, String columnName) throws SQLException {
        return Arrays.asList(rs.getString(columnName).split(","));
    }
}

```



### Ⅱ、`Inteceptror`






















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
>
> ListTypeHandler源码拷贝自:
>
> https://blog.csdn.net/weixin_43184769/article/details/91126687