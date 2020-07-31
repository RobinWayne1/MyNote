# Mybatis一二级缓存原理

## 一、SqlSession的生命周期与事务管理

在讲缓存的实现之前,我们先来讲`SqlSessionTemplate`。在Spring-Mybatis整合中，`SqlSession`的生命周期是一个非常重要的知识点。

回到上一篇中的`result = sqlSession.selectList(command.getName(), param);`调用，该调用最终会分派到`SqlSessionTemplate`中。

```java
//SqlSessionTemplate.java 
@Override
  public <E> List<E> selectList(String statement, Object parameter) {
    return this.sqlSessionProxy.<E> selectList(statement, parameter);
  }
```

而这个`sqlSessionProxy`是什么呢?答案就在`SqlSessionTemplate`的构造函数中。回顾一下：`SqlSessionTemplate`的实例化在Spring对`MapperFactoryBean`的属性过程中。

```java
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
    PersistenceExceptionTranslator exceptionTranslator) {
//···省略代码

  this.sqlSessionFactory = sqlSessionFactory;
//可以看到，这是一个以SqlSessionInterceptor为InvocationHandler的动态代理
  this.sqlSessionProxy = (SqlSession) newProxyInstance(
      SqlSessionFactory.class.getClassLoader(),
      new Class[] { SqlSession.class },
      new SqlSessionInterceptor());
}
```

```java
//SqlSessionInterceptor.java
 public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
     //关注点在这里.获取一个SqlSession.
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
          //这里就是使用真正的DefaultSqlSession调用session.selectList()了
        Object result = method.invoke(sqlSession, args);
//---------------------------------------------------------------------------------------          
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)
      	→→→{
            //⭐其实就是对ThreadLocalMap中的SqlSessionHolder做存在性判断,存在则代表目前的调用依然在Spring事务中,不需要提交;若不在则可以直接提交了
             SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    return (holder != null) && (holder.getSqlSession() == session);
       	   }) 
        
        {
          //⭐如果当前调用不处于Spring事务中，则直接向数据库提交
          sqlSession.commit(true);
        }
        return result;
//---------------------------------------------------------------------------------------
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
//···省略代码
      } finally {
        if (sqlSession != null) {
            //⭐在该方法中同样会做ThreadLocalMap中的SqlSessionHolder的存在性判断,存在则代表目前的调用依然在Spring事务中,不需要将Connection归还给连接池(也就是关闭SqlSession);若不在Spring事务中,则归还Connection
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
```

`SqlSessionInterceptor`是`SqlSessionTemplate`的内部类,由上面的调用情况可以看到,这个类控制着`SqlSession`的生命周期和Spring事务的提交,不可谓不重要。

```java
package org.mybatis.spring;
//SqlSessionUtils.java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

//⭐注意：TransactionSynchronizationManager是Spring框架下的类
  SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

  SqlSession session = sessionHolder(executorType, holder);
     →{
     session = holder.getSqlSession();
 	}
  if (session != null) {
    return session;
  }
    //⭐若没有从ThreadLocalMap中得到SqlSession，则新建一个SqlSession
 	session = sessionFactory.openSession(executorType);
	//没有干什么实事
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
  return session;
}
```

```java
//TransactionSynchronizationManager.java
	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<Map<Object, Object>>("Transactional resources");

public static Object getResource(Object key) {
    //在这里,actualKey实际上是DefaultSqlSessionFactory对象
   Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
   Object value = doGetResource(actualKey);
//省略代码
   return value;
}
private static Object doGetResource(Object actualKey) {
		Map<Object, Object> map = resources.get();
		if (map == null) {
			return null;
		}
		Object value = map.get(actualKey);
//···省略代码
		return value;
	}
```

可以看到，Spring的`TransactionSynchronizationManager`通过一个`ThreadLocal`变量存放着一个线程私有的Map,这个Map里面就存放着`SqlSessionHolder`。如果读者真正去debug一次就会发现，第一次执行到这个方法的时候`SqlSessionHolder`就已经存在于`ThreadLocal`中了,那么是谁将它放进`ThreadLocal`中的。答案显而易见，切面事务。在上一篇原理解析中我因为碍于版面没有讲到在掉用Mapper方法 之前所产生的事务，所以在这里简单讲解一下原理。

### Ⅰ、Spring事务

众所周知，Spring事务是基于AOP编程的。所以他跟`MethodInterceptor`脱不了干系。

```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {


   public TransactionInterceptor() {
   }


   public TransactionInterceptor(PlatformTransactionManager ptm, Properties attributes) {
      setTransactionManager(ptm);
      setTransactionAttributes(attributes);
   }


   public TransactionInterceptor(PlatformTransactionManager ptm, TransactionAttributeSource tas) {
      setTransactionManager(ptm);
      setTransactionAttributeSource(tas);
   }


   @Override
   public Object invoke(final MethodInvocation invocation) throws Throwable {
      // Work out the target class: may be {@code null}.
      // The TransactionAttributeSource should be passed the target class
      // as well as the method, which may be from an interface.
      Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

      //核心方法
      return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
         @Override
         public Object proceedWithInvocation() throws Throwable {
            return invocation.proceed();
         }
      });
   }


}
```

这里就是方法的入口。Spring把这个`TransactionInterceptor`做成了类似`@Around`通知的效果,但他实现的是`MethodInterceptor`。

```java
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
      throws Throwable {

   final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
   final PlatformTransactionManager tm = determineTransactionManager(txAttr);
   final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

   if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager))
//---------------------------------------------------------------------------------------
     //就是在这个方法内,👇
      TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
      →{
          //DatasourceTransactionManager.java
          protected void doBegin(Object transaction, TransactionDefinition definition) {
              //从数据源中获取新的连接
          Connection newCon = this.dataSource.getConnection();
              //⭐就是在这里将连接放入ThreadLocalMap中
              TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
              //开启事务
              con.setAutoCommit(false);
          }
      }
//---------------------------------------------------------------------------------------
      Object retVal = null;
      try {
    
          //这里就是调用ReflectiveMethodInvocation.proceed()方法,继续执行拦截器链,最终调用增强的目标方法
         retVal = invocation.proceedWithInvocation();
      }
      catch (Throwable ex) {
         // target invocation exception
         completeTransactionAfterThrowing(txInfo, ex);
         throw ex;
      }
      finally {
         cleanupTransactionInfo(txInfo);
      }
    //⭐最终在这里对这个事务进行提交,并将ThreadLocalMap中的SqlSession移除
      commitTransactionAfterReturning(txInfo);
      return retVal;
   }
    //省略一部分代码
}
```

总结：Spring事务利用`TransactionInterceptor`实现了`MethodInterceptor`,对需要做事务增强的方法进行了一个环绕通知（只是效果如此）。

1. `TransactionInterceptor`在调用目标方法之前，调用`DatasourceTranscationManager.doBegin()`向连接池申请连接并将该连接(`SqlSession`)放入`ThreadLocalMap`中,并通过该连接开启事务。之后则继续执行拦截器链,直到调用目标方法
2. 在目标方法中调用Mapper对象操作数据库时，`SqlSessionInterceptor`尝试从`ThreadLocalMap`中获取`SqlSession`以调用如`Mapper.selectList()`方法。若获取失败,则说明当前调用并不在事务中,新建一个`SqlSession`以供当前这条调用语句使用。
3. 当通过`SqlSession`成功对数据库进行操作后,`SqlSessionInterceptor`将对`ThreadLocalMap`的`SqlSessionHolder`进行存在性判断,若存在则说明当前调用在事务中，直接返回。若不存在，直接提交,至此这个`SqlSession`对象将被GC。
4. 此时目标方法执行完毕，重新返回到拦截器链，执行`TransactionInterceptor`剩下的方法，此时`TransactionInterceptor`将会对这个事务做一个提交,并将`ThreadLocalMap`中的`SqlSession`移除,这样就能保证当前线程剩余代码都不会与事务共用`SqlSession`。

说了这么多，`SqlSession`的生命周期就是一句话:**当Mapper的调用没有被包含在事务增强中时,`SqlSession`的生命周期也就是在 Mapper对象方法 调用开始时创建,在掉用完毕后结束。而若Mapper的调用被包含在事务增强中时，`SqlSession`的生命周期将会持续整个事务,也就是说事务中所有对Mapper的调用都会公用一个`SqlSession`对象(一条连接)。**

## 二、一级缓存

接下来就可以讲一级缓存。一级缓存是由四个`Executor`的抽象父类`BaseExecutor`负责的,在调用`sqlSession.selectList()`后,其内部就会调用`BaseExecutor.query()`方法。

```java
//BaseExecutor.java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    //在基本原理篇曾讲过，这个对象存放着参数映射关系，执行完动态语句的SQL以及参数值
  BoundSql boundSql = ms.getBoundSql(parameter);
    //⭐
  CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    //进入重载方法
  return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

在`createCacheKey()`中,将`MappedStatement`的Id、SQL的offset、SQL的limit、SQL本身以及SQL中的参数传入了CacheKey这个类，最终构成`CacheKey`。

```java
//BaseExecutor.java
//缓存
protected PerpetualCache localCache;
@Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
//···省略代码
    List<E> list;
    try {
      queryStack++;
        //根据CacheKey从缓存中得到数据。一级缓存的存储对象是PerpetualCache，其内部也就是用一个HashMap来存储数据
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
          //向数据库进行查询
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
        //延迟加载相关
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
   
      deferredLoads.clear();
        //这个是mybatis-config.xml可以控制的<setting>,默认LocalCacheScope为session,如果将其设置为statement,可以看到每次在数据库得到数据后都会将缓存清空,使得一级缓存失效
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        //⭐清空缓存，其中在调用BaseExecutor的update()或commit()时,都会调用该方法刷新缓存
        clearLocalCache();
      }
    }
    return list;
  }

  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        //在这里调用实现类Executor的具体方法
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
      //将结果放入缓存中
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```

上面就是在上一篇中所讲的缺失的两个调用栈，其实并没有什么逻辑可言,主要是注意某些缓存失效的地方。

在源码分析的最后，我们确认一下，如果是`insert/delete/update/commit`方法，缓存就会刷新的原因。

`SqlSession`的`insert`方法和`delete`方法，都会统一走`update`的流程，代码如下所示：

```java
@Override
public int insert(String statement, Object parameter) {
    return update(statement, parameter);
  }
   @Override
  public int delete(String statement) {
    return update(statement, null);
}
  @Override
  public void commit(boolean force) {
    try {
      executor.commit(isCommitOrRollbackRequired(force));
      dirty = false;
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error committing transaction.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

`update`和`commit`和`rollback`方法也是委托给了`Executor`执行。`BaseExecutor`的执行方法如下所示：

```java
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
}

  @Override
  public void commit(boolean required) throws SQLException {
    if (closed) {
      throw new ExecutorException("Cannot commit, transaction is already closed");
    }
    clearLocalCache();
    flushStatements();
    if (required) {
      transaction.commit();
    }
  }
  @Override
  public void rollback(boolean required) throws SQLException {
    if (!closed) {
      try {
        clearLocalCache();
        flushStatements(true);
      } finally {
        if (required) {
          transaction.rollback();
        }
      }
    }
  }
```

每次执行这三个方法前都会清空`localCache`。

小结:

1. 生命周期:在Spring-Mybatis整合中,由于不在Spring事务中时每次调用Mapper方法都会创建一个新的`SqlSession`对象,而由于`BaseExecutor`中的`localCache`是`SqlSession`对象持有的,所以不在Spring事务中的Mapper调用相当于每次调用都创建一个新缓存,也就是失效状态。而在Spring事务中的Mapper方法调用会公用一个`SqlSession`,所以也就能使用缓存,**也就是以事务为单位的缓存**。

2. 由上面的源码解析就可得出的结论，`<setting name="localCacheValue" value="statement">`即可完全禁用一级缓存,因为每次Mapper调用返回结果时都会清空缓存

3. `BaseExecutor`的`update()`和`commit()`同样会刷新缓存(无论是否在事务中),前者在用户向`SqlSession`进行CUD时调用,后者在Spring事务或`SqlSession`对`Connection`事务进行提交时调用

缺陷:

4. MyBatis一级缓存内部设计简单，只是一个没有容量限定的HashMap，在缓存的功能性上有所欠缺。

5. MyBatis的一级缓存最大范围是`SqlSession`内部，有多个`SqlSession`或者分布式的环境下，数据库写操作会引起脏数据。

## 三、二级缓存

### Ⅰ、二级缓存使用配置

要正确的使用二级缓存，需完成如下配置的。

1. 在MyBatis的配置文件中开启二级缓存。

```xml
<setting name="cacheEnabled" value="true"/>
```

1. 在MyBatis的映射XML中配置cache或者 cache-ref 。

cache标签用于声明这个namespace使用二级缓存，并且可以自定义配置。

```xml
<cache/>   
```

- `type`：cache使用的类型，默认是`PerpetualCache`，这在一级缓存中提到过。
- `eviction`： 定义回收的策略，常见的有FIFO，LRU。
- `flushInterval`： 配置一定时间自动刷新缓存，单位是毫秒。
- `size`： 最多缓存对象的个数。
- `readOnly`： 是否只读，若配置可读写，则需要对应的实体类能够序列化。
- `blocking`： 若缓存中找不到对应的key，是否会一直blocking，直到有对应的数据进入缓存。

`cache-ref`代表引用别的命名空间的Cache配置，两个命名空间的操作使用的是同一个Cache。

```xml
<cache-ref namespace="mapper.StudentMapper"/>
```

二级缓存是由`CachingExecutor`负责管理的,但实际的缓存对象的存储却是在`MappedStatement`中的,**每个属于同一个命名空间的`MappedStatement`都存放着整个命名空间公用的缓存对象**。

```java
//MappedStatement.java
private Cache cache;
```

这就是缓存成员,在实际运行中它使用了装饰者模式,但最终的本体也还是`PerpetualCache`。

以下是具体这些Cache实现类的介绍，他们的组合为Cache赋予了不同的能力。

- `SynchronizedCache`：同步Cache，实现比较简单，直接使用`synchronized`修饰方法。
- `LoggingCache`：日志功能，装饰类，用于记录缓存的命中率，如果开启了DEBUG模式，则会输出命中率日志。
- `SerializedCache`：序列化功能，将值序列化后存到缓存中。该功能用于缓存返回一份实例的Copy，用于保证线程安全。
- `LruCache`：采用了Lru算法的Cache实现，移除最近最少使用的Key/Value。
- `PerpetualCache`： 作为为最基础的缓存类，底层实现比较简单，直接使用了HashMap。

该成员的赋值在解析Mapper时的`MappedBuilderAssiant.addMappedStatement()`构造`MappedStatement`的过程中,在构造时会使得同一个命名空间的`MappedStatement`都持有同一个缓存引用。

由此可见，**二级缓存是一个以namespace为单位的缓存，其生命周期不与`SqlSession`绑定。**

### Ⅱ、二级缓存管理逻辑

二级缓存是由`CachingExecutor`负责管理，他是一个装饰者，本体其实还是我们配置的那三个`Executor`,调用`sqlSession.selectList()`将会分派到`CachingExecutor.query()`中。

```java
//CachingExecutor.java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    //和BaseExecutor一样,构造CacheKey
  BoundSql boundSql = ms.getBoundSql(parameterObject);
  CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    //进入重载方法
  return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
//事务缓存管理器
 private final TransactionalCacheManager tcm = new TransactionalCacheManager();

  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
      
    if (cache != null) {
        //检查是否要刷新缓存
      flushCacheIfRequired(ms);
    }
      if (ms.isUseCache() && resultHandler == null) {
          //存储过程相关
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
          //从二级缓存中获取与cacheKey映射的结果
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
            //调用本体Executor的query()获取数据
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
            //将数据放入二级缓存中
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

```

可以看到,`CachingExecutor`并没有亲自处理缓存,而是都交给了`TransactionalCacheManager`。

### Ⅱ、`TransactionalCache`

```java
//TransactionalCacheManager.java
private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<>();

```

`TransactionalCacheManager`只有一个成员变量,这个Map保存了`Cache`和用`TransactionalCache`包装后的`Cache`的映射关系,`TransactionalCacheManager`的方法都是在对`TransactionalCache`的操作。`TransactionalCache`最重要的作用就是管理写入缓存的时机,它实现的机制是:==**如果事务提交，对缓存的操作才会生效，如果事务回滚或者不提交事务，则不对缓存产生影响。**==

`TransactionalCache`的成员变量如下所示。

```java
  //TransactionalCache.java 
  //这就是缓存对象
  private final Cache delegate;
  //用作控制删除缓存的标志
  private boolean clearOnCommit;
  //⭐在未提交事务前，数据将存放在这个Map中；待调用commit()时将数据写入Cache中
  private final Map<Object, Object> entriesToAddOnCommit;

```

#### ①、`TransactionalCache.putObject()`

```java
@Override
public void putObject(Object key, Object object) {
  entriesToAddOnCommit.put(key, object);
}
```

可以看到,`putObject()`并没有直接往缓存中写数据,而是存在了`entriesToAddOnCommit`成员中。



#### ②、`TransactionalCache.clear()`

```java
@Override
  public void clear() {
    clearOnCommit = true;
      //⭐清空 将要在commit()时写缓存的数据
    entriesToAddOnCommit.clear();
  }  
//CachingExecutor.java
//只有这个方法有调用clear()
private void flushCacheIfRequired(MappedStatement ms) {
    Cache cache = ms.getCache();
    //⭐ms.isFlushCacheRequired()这个判断语句,对于update(CUD)操作将会返回true,而对于R操作则会返回false.但这个是可以配置的,若在<update id="updateName" flushCache="true">时,则在判断该条update语句时将永远返回true,也即不会刷新二级缓存	
    if (cache != null && ms.isFlushCacheRequired()) {
      tcm.clear(cache);
    }
  }  
```

在调用`clear()`时,会清空需要在提交时加入缓存的列表，同时设定在调用`commit()`提交时清空缓存（请看a's'dad）。`CachingExecutor`的`update()`和`query()`都有调用`flushCacheIfRequired()`,默认`update()`时会调用`clear()`。

#### ③、`TransactionalCache.getObject()`

```java
@Override
public Object getObject(Object key) {
  //从缓存中获取数据
  Object object = delegate.getObject(key);
    //用来输出缓存命中率的成员,不管
  if (object == null) {
    entriesMissedInCache.add(key);
  }
  //⭐
  if (clearOnCommit) {
    return null;
  } else {
    return object;
  }
}
```

`getObject()`就是从缓存中获取数据,但是在返回之前还要判断。如果在当前`SqlSession`中已经调用过`clear()`,(由于未`commit()`,此时还能在缓存中找到数据),此时缓存中的数据不予返回。简单来说,就是事务中`update()`后的`select()`不会使用二级缓存。

#### ④、`TransactionalCache.clear()`

```java
  public void commit() {
      //⭐判断是否应该刷新缓存
    if (clearOnCommit) {
      delegate.clear();
    }
      //⭐将成员entriesToAddOnCommit中的所有数据刷到缓存中
    flushPendingEntries();
    reset();
  }
```

在`commit()`的时候,就会判断标志是否清空缓存,将事务执行中临时存放在`entriesToAddOnCommit`的数据刷回缓存,并将所有成员变量状态重置,以迎接该`SqlSession`的下一个事务。

#### ⑤、`TransactionalCache.flushPendingEntries()`和`TransactionalCache.reset()`

```java
 private void flushPendingEntries() {
     //如上所述,刷数据到缓存
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
      delegate.putObject(entry.getKey(), entry.getValue());
    }
    for (Object entry : entriesMissedInCache) {
      if (!entriesToAddOnCommit.containsKey(entry)) {
        delegate.putObject(entry, null);
      }
    }
  }
 private void reset() {
    clearOnCommit = false;
    entriesToAddOnCommit.clear();
    entriesMissedInCache.clear();
  }
```

#### ⑥、`TransactionalCache.rollback()`和`TransactionalCache.unlockMissedEntries()`

```java

  public void rollback() {
    unlockMissedEntries();
    reset();
  }
  private void unlockMissedEntries() {
    for (Object entry : entriesMissedInCache) {
      try {
        delegate.removeObject(entry);
      } catch (Exception e) {
        log.warn("Unexpected exception while notifiying a rollback to the cache adapter. "
            + "Consider upgrading your cache adapter to the latest version. Cause: " + e);
      }
    }
  }
```

若是出现回滚，直接重置所有成员。

小结：

1. 生命周期:二级缓存的存储粒度是**每个命名空间共用一个缓存对象**，相比于一级缓存二级缓存的粒度更小。但二级缓存是一个全局缓存，所有`SqlSession`都可以使用,因为该缓存是存放在`Configuration`的`MappedStatement`中的;而一级缓存只能在相同`SqlSession`下使用,因为他是`BaseExecutor`持有的。
2. 二级缓存可以通过不同的装饰者实现对Cache的管理，如FIFO或者LRU，比一级缓存的可控性更强
3. 在二级缓存中,只有事务提交后才会将事务内`select`的数据存入缓存。而一级缓存中，数据将不断地随着`select`的执行成功而写入缓存。
4. 两者的`update()`、`commit()`、`rollback()`都默认刷新整个缓存,不过二级缓存还能够通过在Statement中使用`<flushCache>`配置以强制`update()`不刷新缓存。

缺点：

5. 在分布式系统下，两者的缓存都会有脏数据
6. ⭐由于二级缓存以namespace为单位，所以最好对一个表的所有操作都放在一个Mapper中。因为如果放在两个Mapper中，业务代码调用其中一个Mapper的`update()`,那么`TransactionalCache`只会刷新该namespace的缓存,而不会刷新另一个namespace的缓存,这就导致了两者的不一致使得脏数据产生。
7. 与上面差不多的道理，如何解决多表查询的问题？对于其他Mapper对表的修改 该多表查询语句所在的Mapper并不能感应到。在多表查询中，若设计两个表的多表查询，则可以通过`<cache-ref>`配置,使得两个命名空间使用同一个`Cache`对象。不过这样做的后果是，缓存的粒度变粗了，多个`Mapper namespace`下的所有操作都会对缓存使用造成影响。