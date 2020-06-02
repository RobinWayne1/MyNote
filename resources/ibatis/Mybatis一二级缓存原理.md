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

众所周知，Spring事务是基于Aop编程的。所以他跟`MethodInterceptor`脱不了干系。

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
     //就是在这个方法内
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
4. 此时目标方法执行完毕，重新返回到拦截器链，执行`TransactionInterceptor`剩下的方法，此时`TransactionInterceptor`将会对这个事务做一个提交,并将ThreadLocalMap中的SqlSession移除。

说了这么多，`SqlSession`的生命周期就是一句话:**当Mapper的调用没有被包含在事务增强中时,`SqlSession`的生命周期也就是在 Mapper对象方法 调用开始时创建,在掉用完毕后结束。而若Mapper的调用被包含在事务增强中时，`SqlSession`的生命周期将会持续整个事务,也就是说事务中所有对Mapper的调用都会公用一个`SqlSession`对象(一条连接)。**