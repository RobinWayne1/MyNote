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
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          //⭐如果当前调用不处于Spring事务中，则直接向数据库提交
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
//···省略代码
      } finally {
        if (sqlSession != null) {
            //⭐里面的代码逻辑将会判断，如果当前调用处于Spring事务中，则保持连接；若不处于事务中，则将连接返还给连接池
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
//···省略代码
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









总结：