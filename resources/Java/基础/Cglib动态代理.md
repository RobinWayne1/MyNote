# Cglib动态代理源码解析

## 一、基本使用

使用Cglib动态代理步骤：

1. 创建被代理类
2. 创建实现`MethodInterceptor`的类,在`intercept()`中实现代理逻辑
3. 创建实现`CallbackFilter`的类,在`accept()`中写出 对被代理类的不同方法执行不同的拦截调用 逻辑
4. 在主方法中创建`Enhancer`对象,将被代理类、`MethodInterceptor`实现类、`CallbackFilter`实现类都`set()`进去,调用`enhancer.create()`即可创建出一个代理对象

### ①、创建被代理类

```java
public class Dao {
    
    public void update() {
        System.out.println("PeopleDao.update()");
    }
    
    public void select() {
        System.out.println("PeopleDao.select()");
    }
}
```

### ②、创建实现`MethodInterceptor`的类

```java
public class DaoProxy implements MethodInterceptor {

    /*
    	Object表示要进行增强的对象
		Method表示拦截的方法
		Object[]数组表示参数列表，基本数据类型需要传入其包装类型，如int-->Integer、long-Long、double-->Double
		MethodProxy表示对方法的代理，invokeSuper方法表示对被代理对象方法的调用(这种调用是Cglib经过优化的调用,并不是反射调用)
		*/
    @Override
    public Object intercept(Object object, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("Before Method Invoke");
        proxy.invokeSuper(object, args);
        System.out.println("After Method Invoke");
        
        return object;
    }
    
}
```

```java
public class DaoAnotherProxy implements MethodInterceptor {

    @Override
    public Object intercept(Object object, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        
        System.out.println("StartTime=[" + System.currentTimeMillis() + "]");
        method.invoke(object, args);
        System.out.println("EndTime=[" + System.currentTimeMillis() + "]");
        return object;
    }
    
}
```

### ③、创建实现`CallbackFilter`的类

```java
public class DaoFilter implements CallbackFilter {

    @Override
    public int accept(Method method) {
        if ("select".equals(method.getName())) {
            //这里返回的0和1其实就是Callback数组的下标
            return 0;
        }
        return 1;
    }
    
}
```

### ④、主方法

```java
public class CglibTest {

    @Test
    public void testCglib() {
        DaoProxy daoProxy = new DaoProxy();
        DaoAnotherProxy daoAnotherProxy = new DaoAnotherProxy();
        
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Dao.class);
        enhancer.setCallbacks(new Callback[]{daoProxy, daoAnotherProxy, NoOp.INSTANCE});
        enhancer.setCallbackFilter(new DaoFilter());
        
        Dao dao = (Dao)enhancer.create();
        dao.update();
        dao.select();
    }
    
}
```

## 二、源码解析

主要创建代理类的方法即`enhancer.create()`方法，但是我们在执行这个方法之前设置了几个值，可以分别看下方法体。

* `setCallbacks()`，即设置回调，我们创建出代理类后调用方法则是使用的这个回调接口，类似于jdk动态代理中的InvocationHandler

  ```java
  public void setCallbacks(Callback[] callbacks) {
          if (callbacks != null && callbacks.length == 0) {
              throw new IllegalArgumentException("Array cannot be empty");
          }
          this.callbacks = callbacks;
      }
  ```
  
* `setSuperClass()` 即设置被代理类，这儿可以看到做了个判断，如果为interface则设置interfaces,如果是Object则设置为null(因为所有类都自动继承Object),如果为普通class则设置class，==可以看到cglib代理不光可以代理接口，也可以代理普通类。只不过在`intercept()`方法中不能调用接口的方法就是了，也就是说被代理接口就是一个空壳。==

  ```java
  public void setSuperclass(Class superclass) {
          if (superclass != null && superclass.isInterface()) {
  　　　　　　　　//设置代理接口
              setInterfaces(new Class[]{ superclass });
          } else if (superclass != null && superclass.equals(Object.class)) {
              // 未Object则用设置
              this.superclass = null;
          } else {
  　　　　　　 //设置代理类
              this.superclass = superclass;
          }
      }
  
  public void setInterfaces(Class[] interfaces) {
      this.interfaces = interfaces;
  }
  ```

### ①、`enhancer.create()`

```java
//Enhancer.java
public Object create() {
        //不作代理类限制
        classOnly = false;
        //没有构造参数类型
        argumentTypes = null;
        //执行创建
        return createHelper();
    }
private Object createHelper() { 
    //进行验证 并确定CallBack类型,我们用的是MethodInterceptor,此外还有InvocationHandler(这是cglib自己的)等等的Callback
    preValidate();
    
    //⭐获取当前代理类的标识类Enhancer.EnhancerKey的代理
    Object key = KEY_FACTORY.newInstance((superclass != null) ? superclass.getName() : null,
            ReflectUtils.getNames(interfaces),
            filter == ALL_ZERO ? null : new WeakCacheKey<CallbackFilter>(filter),
            callbackTypes,
            useFactory,
            interceptDuringConstruction,
            serialVersionUID);
    //设置当前enhancer的代理类的key标识
    this.currentKey = key;
    //调用父类即 AbstractClassGenerator的创建代理类
    Object result = super.create(key);
    return result;
}
```

#### 1、`EnhancerKey`

先来讲讲这个`KeyFactory`。

```java
private static final EnhancerKey KEY_FACTORY =
  (EnhancerKey)KeyFactory.create(EnhancerKey.class, KeyFactory.HASH_ASM_TYPE, null);
//KeyFactory.java
public static KeyFactory create(ClassLoader loader, Class keyInterface, KeyFactoryCustomizer customizer,
                                    List<KeyFactoryCustomizer> next) {
        Generator gen = new Generator();
        gen.setInterface(keyInterface);

        if (customizer != null) {
            gen.addCustomizer(customizer);
        }
        if (next != null && !next.isEmpty()) {
            for (KeyFactoryCustomizer keyFactoryCustomizer : next) {
                gen.addCustomizer(keyFactoryCustomizer);
            }
        }
        gen.setClassLoader(loader);
        return gen.create();
    }
```

`Key_Factory`在类加载时就调用了`KeyFactory.create()`进行初始化。有意思的是，这个`create()`方法内部最终会调用到`AbstractClassGenerator.create()`中去,也就是说,这个`EnhancerKey`实际上也是一个动态代理!

```java
//Enhancer.java
//这是Enhancer的内部类
public interface EnhancerKey {
        public Object newInstance(String type,
                                  String[] interfaces,
                                  WeakCacheKey<CallbackFilter> filter,
                                  Type[] callbackTypes,
                                  boolean useFactory,
                                  boolean interceptDuringConstruction,
                                  Long serialVersionUID);
    }
```

它还是一个面向接口的代理类。直接来看看`EnhancerKey`最终的生成类。

```java
//注意实现了EnhancerKey
public class Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517 extends KeyFactory implements EnhancerKey {
    //存放newInstance()传入的参数值
    private final String FIELD_0;
    private final String[] FIELD_1;
    private final WeakCacheKey FIELD_2;
    private final Type[] FIELD_3;
    private final boolean FIELD_4;
    private final boolean FIELD_5;
    private final Long FIELD_6;

    public Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517() {
    }

    public Object newInstance(String var1, String[] var2, WeakCacheKey var3, Type[] var4, boolean var5, boolean var6, Long var7) {
        return new Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517(var1, var2, var3, var4, var5, var6, var7);
    }
	//创建一个EnhancerKey对象
    public Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517(String var1, String[] var2, WeakCacheKey var3, Type[] var4, boolean var5, boolean var6, Long var7) {
        this.FIELD_0 = var1;
        this.FIELD_1 = var2;
        this.FIELD_2 = var3;
        this.FIELD_3 = var4;
        this.FIELD_4 = var5;
        this.FIELD_5 = var6;
        this.FIELD_6 = var7;
    }

    public int hashCode() {
        int var10002 = 3691 * 7247;
        String var10001 = this.FIELD_0;
        int var10000 = var10002 + (var10001 != null ? var10001.hashCode() : 0);
        String[] var5 = this.FIELD_1;
        if (var5 != null) {
            String[] var1 = var5;

            for(int var2 = 0; var2 < var1.length; ++var2) {
                var10000 = var10000 * 7247 + (var1[var2] != null ? var1[var2].hashCode() : 0);
            }
        }

        var10002 = var10000 * 7247;
        WeakCacheKey var6 = this.FIELD_2;
        var10000 = var10002 + (var6 != null ? var6.hashCode() : 0);
        Type[] var7 = this.FIELD_3;
        if (var7 != null) {
            Type[] var3 = var7;

            for(int var4 = 0; var4 < var3.length; ++var4) {
                var10000 = var10000 * 7247 + (var3[var4] != null ? var3[var4].hashCode() : 0);
            }
        }

        var10002 = ((var10000 * 7247 + (this.FIELD_4 ^ 1)) * 7247 + (this.FIELD_5 ^ 1)) * 7247;
        Long var8 = this.FIELD_6;
        return var10002 + (var8 != null ? var8.hashCode() : 0);
    }

    public boolean equals(Object var1) {
        if (var1 instanceof Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517) {
            String var10000 = this.FIELD_0;
            String var10001 = ((Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517)var1).FIELD_0;
            if (var10001 == null) {
                if (var10000 != null) {
                    return false;
                }
            } else if (var10000 == null || !var10000.equals(var10001)) {
                return false;
            }

            String[] var8 = this.FIELD_1;
            String[] var10 = ((Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517)var1).FIELD_1;
            if (var10 == null) {
                if (var8 != null) {
                    return false;
                }
            } else {
                label178: {
                    if (var8 != null) {
                        if (var10.length == var8.length) {
                            String[] var2 = var10;
                            String[] var3 = var8;
                            int var4 = 0;

                            while(true) {
                                if (var4 >= var2.length) {
                                    break label178;
                                }

                                var10000 = var2[var4];
                                var10001 = var3[var4];
                                if (var3[var4] == null) {
                                    if (var10000 != null) {
                                        return false;
                                    }
                                } else if (var10000 == null || !var10000.equals(var10001)) {
                                    return false;
                                }

                                ++var4;
                            }
                        }
                    }

                    return false;
                }
            }

            WeakCacheKey var9 = this.FIELD_2;
            WeakCacheKey var13 = ((Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517)var1).FIELD_2;
            if (var13 == null) {
                if (var9 != null) {
                    return false;
                }
            } else if (var9 == null || !var9.equals(var13)) {
                return false;
            }

            Type[] var11 = this.FIELD_3;
            Type[] var15 = ((Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517)var1).FIELD_3;
            if (var15 == null) {
                if (var11 != null) {
                    return false;
                }
            } else {
                if (var11 == null) {
                    return false;
                }

                if (var15.length != var11.length) {
                    return false;
                }

                Type[] var5 = var15;
                Type[] var6 = var11;

                for(int var7 = 0; var7 < var5.length; ++var7) {
                    Type var12 = var5[var7];
                    Type var16 = var6[var7];
                    if (var6[var7] == null) {
                        if (var12 != null) {
                            return false;
                        }
                    } else if (var12 == null || !var12.equals(var16)) {
                        return false;
                    }
                }
            }

            if (this.FIELD_4 == ((Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517)var1).FIELD_4 && this.FIELD_5 == ((Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517)var1).FIELD_5) {
                Long var14 = this.FIELD_6;
                Long var17 = ((Enhancer$EnhancerKey$$KeyFactoryByCGLIB$$b5777517)var1).FIELD_6;
                if (var17 == null) {
                    if (var14 == null) {
                        return true;
                    }
                } else if (var14 != null && var14.equals(var17)) {
                    return true;
                }
            }
        }

        return false;
    }

    public String toString() {
        StringBuffer var10000 = new StringBuffer();
        String var10001 = this.FIELD_0;
        var10000 = (var10001 != null ? var10000.append(var10001.toString()) : var10000.append("null")).append(", ");
        String[] var6 = this.FIELD_1;
        if (var6 != null) {
            var10000 = var10000.append("{");
            String[] var1 = var6;

            for(int var2 = 0; var2 < var1.length; ++var2) {
                var10000 = (var1[var2] != null ? var10000.append(var1[var2].toString()) : var10000.append("null")).append(", ");
            }

            var10000.setLength(var10000.length() - 2);
            var10000 = var10000.append("}");
        } else {
            var10000 = var10000.append("null");
        }

        var10000 = var10000.append(", ");
        WeakCacheKey var9 = this.FIELD_2;
        var10000 = (var9 != null ? var10000.append(var9.toString()) : var10000.append("null")).append(", ");
        Type[] var10 = this.FIELD_3;
        if (var10 != null) {
            var10000 = var10000.append("{");
            Type[] var3 = var10;

            for(int var4 = 0; var4 < var3.length; ++var4) {
                var10000 = (var3[var4] != null ? var10000.append(var3[var4].toString()) : var10000.append("null")).append(", ");
            }

            var10000.setLength(var10000.length() - 2);
            var10000 = var10000.append("}");
        } else {
            var10000 = var10000.append("null");
        }

        var10000 = var10000.append(", ").append(this.FIELD_4).append(", ").append(this.FIELD_5).append(", ");
        Long var13 = this.FIELD_6;
        return (var13 != null ? var10000.append(var13.toString()) : var10000.append("null")).toString();
    }
}
```

容易看出,这个动态生成的`EnhancerKey`类其实就是用来存放`newInstance()`参数值的一个对象,这个`EnhancerKey` 用来唯一标识代理类工厂`Enhancer`对象(其实也可以说是代理类,因为一个`Enhancer`负责创建一个代理类),这个`EnhancerKey`将作为key用在存储代理类Class的缓存中。可以看见这个`Enhancerkey`中存储了很多东西,因为对于一个代理类来说,目标被代理类、回调类型、回调过滤器等等都是这个代理类的唯一标识,所以才要用这种`mutil-key`的形式存储。至于为什么不手动实现`EnhancerKey`而要使用动态代理生成`EnhancerKey`,则可能是为了方便,因为只要接口有指定了`newInstance()`方法且里面有参数,cglib就可以动态生成这个接口的实现类并且`hashcode()`和`equals()`这两个很重要的方法都已经动态生成了。

#### 2、`AbstractClassGenerator.create()`

```java

//WeakHashMap与ThreadLocalMap类似,都继承了WeakReference并且将key作为弱引用
//⭐一级缓存
    private static volatile Map<ClassLoader, ClassLoaderData> CACHE = new WeakHashMap<ClassLoader, ClassLoaderData>();


protected Object create(Object key) {
        try {
            //获取到当前生成器的类加载器
            ClassLoader loader = getClassLoader();
            //当前类加载器对应的缓存  缓存key为类加载器，缓存的value为ClassLoaderData,而这个ClassLoaderData里面也有缓存映射,即(EnhancerKey→代理类Class),后面会详细讲
            Map<ClassLoader, ClassLoaderData> cache = CACHE;
            //先从缓存中获取下当前类加载器所有加载过的类
            ClassLoaderData data = cache.get(loader);
            //如果为空
            if (data == null) {
                synchronized (AbstractClassGenerator.class) {
                    cache = CACHE;
                    data = cache.get(loader);
                    //经典的防止并发修改 二次判断
                    if (data == null) {
                        //新建一个缓存Cache,将之前的缓存Cache的数据添加进来,并且WeakHashMap内部会将已经被gc回收的数据给清除掉
                        Map<ClassLoader, ClassLoaderData> newCache = new WeakHashMap<ClassLoader, ClassLoaderData>(cache);
                        //新建一个当前加载器对应的ClassLoaderData 并加到缓存中  但ClassLoaderData中此时还没有数据
                        data = new ClassLoaderData(loader);
                        newCache.put(loader, data);
                        //刷新全局缓存
                        CACHE = newCache;
                    }
                }
            }
            //设置一个全局key 
            this.key = key;
            
            //在刚创建的data(ClassLoaderData)中调用get方法 并将当前生成器，
            //以及是否使用缓存的标识穿进去 系统参数 System.getProperty("cglib.useCache", "true")  
            //⭐返回的是生成好的代理类的class信息,在这里将生成代理类
            Object obj = data.get(this, getUseCache());
            //如果为class则实例化class并返回  就是我们需要的代理类
            if (obj instanceof Class) {
                return firstInstance((Class) obj);
            }
            //如果不是则说明是实体  则直接执行另一个方法返回实体,总之最终都是调用Constructor.newInstance()生成代理类实例
            return nextInstance(obj);
        } catch (RuntimeException e) {
            throw e;
        } catch (Error e) {
            throw e;
        } catch (Exception e) {
            throw new CodeGenerationException(e);
        }
    }
```

#### 3、⭐`ClassLoaderData`——二级映射缓存

##### Ⅰ、构造方法

```java
  private static final Function<AbstractClassGenerator, Object> GET_KEY = new Function<AbstractClassGenerator, Object>() {
      //下面会讲.返回代理类工厂标识EnhancerKey
            public Object apply(AbstractClassGenerator gen) {
                return gen.key;
            }
        };

//Enhancer.java
//同样是Enhancer的内部类
public ClassLoaderData(ClassLoader classLoader) {
    if (classLoader == null) {
        throw new IllegalArgumentException("classLoader == null is not yet supported");
    }
    //指定类加载器
    this.classLoader = new WeakReference<ClassLoader>(classLoader);
    //⭐这个Funcition非常重要。其内部实现的apply()方法将会在 在缓存中没有找到value时调用从而构造一个代理类，这是LodingCache的内部调用。
    Function<AbstractClassGenerator, Object> load =
            new Function<AbstractClassGenerator, Object>() {
                public Object apply(AbstractClassGenerator gen) {
                    //⭐调用AbstractClassGenerator.generate()方法创建代理类
                    Class klass = gen.generate(ClassLoaderData.this);
                    return gen.wrapCachedClass(klass);
                }
            };
    //这就是二级映射缓存.由此可以看出缓存的映射:一个ClassLoader对应着一个ClassLoaderData,而ClassLoaderData内部的LoadingCache又有着工厂标识EnhancerKey→Class对象的映射
    //其中这个GET_KEY也是一个Function,用作LoadingCache寻找真正的key,即EnhancerKey(外部调用get()的参数是AbstractClassGenerator)
    generatedClasses = new LoadingCache<AbstractClassGenerator, Object, Object>(GET_KEY, load);
}
```

##### Ⅱ、`classLoaderData.get(this, getUseCache())`

```java
public Object get(AbstractClassGenerator gen, boolean useCache) {
    //如果不使用缓存,那么直接重新创建代理类
    if (!useCache) {
      return gen.generate(ClassLoaderData.this);
    } else {
        //使用缓存,则先去LoadingCache中取
      Object cachedValue = generatedClasses.get(gen);
        //将Class对象包装成WeakReference
      return gen.unwrapCachedValue(cachedValue);
    }
}
```

##### Ⅲ、`LoadingCache`

```java
//其中K代表调用调用LoadingCache.get(K key)传入的参数类型,这个参数在这里是AbstractClassGenerator,充当存储KK(EnhancerKey)的地方.而KK就是map中kv键值对的key了,v就是map中kv键值对的value.
public class LoadingCache<K, KK, V> {
    //这个map的value有可能存放FutureTask也有可能存放被弱引用的Class对象(也就是WeakReference对象)
    //当value是FutureTask时,说明有线程正在构造一个代理类,若不是FutureTask则说明这个缓存可用
    protected final ConcurrentMap<KK, Object> map;
    //这就是代理类构造器
    protected final Function<K, V> loader;
    //相当于寻找真正的key的工厂
    protected final Function<K, KK> keyMapper;

    public static final Function IDENTITY = new Function() {
        public Object apply(Object key) {
            return key;
        }
    };

    public LoadingCache(Function<K, KK> keyMapper, Function<K, V> loader) {
        this.keyMapper = keyMapper;
        this.loader = loader;
        this.map = new ConcurrentHashMap<KK, Object>();
    }
}
```

##### Ⅳ、`LoadingCache.get()`

```java
public V get(K key) {
    //通过keyMapper这个Function获取key,其实也就是获取AbstractClassGenerator的EnhancerKey
    final KK cacheKey = keyMapper.apply(key);
    //缓存映射中取出对象
    Object v = map.get(cacheKey);
    
    if (v != null && !(v instanceof FutureTask)) {
        return (V) v;
    }
	//⭐主要逻辑在这里
    return createEntry(key, cacheKey, v);
}
```

```java
protected V createEntry(final K key, KK cacheKey, Object v) {
    FutureTask<V> task;
    boolean creator = false;
    if (v != null) {
        // Another thread is already loading an instance
        //官方注释写的很清楚,其他线程正在创建代理类Class对象以放入map中
        task = (FutureTask<V>) v;
    } else {
        //来到这里则说明v==null,此时由本线程来创建代理类
        task = new FutureTask<V>(new Callable<V>() {
            public V call() throws Exception {
                //⭐这就是loader的用处,创建代理类
                return loader.apply(key);
            }
        });
        //先将futuerTask放进map中占个位,以表示这个代理类已经有线程在创建了
        Object prevTask = map.putIfAbsent(cacheKey, task);
        if (prevTask == null) {
            // creator does the load
            creator = true;
            //注意是直接调用run(),没有开启新线程
            task.run();
        }
        //下面两个判断就是用作处理并发创建代理类的情况
        else if (prevTask instanceof FutureTask) {
            task = (FutureTask<V>) prevTask;
        } else {
            return (V) prevTask;
        }
    }

    V result;
    try {
        result = task.get();
    } catch (InterruptedException e) {
        throw new IllegalStateException("Interrupted while loading cache item", e);
    } catch (ExecutionException e) {
        Throwable cause = e.getCause();
        if (cause instanceof RuntimeException) {
            throw ((RuntimeException) cause);
        }
        throw new IllegalStateException("Unable to load cache item", cause);
    }
    if (creator) {
        //将Class对象放入map中
        map.put(cacheKey, result);
    }
    return result;
}
```

通过上面的分析可以得知，这个类主要作用是传入代理类生成器，根据这个代理类生成器以及代理类生成器的key来获取缓存，如果没有获取到则构建一个`FutureTask`来回调我们之前初始化时传入的 回调函数，并调用其中的apply方法，而具体调用的则是我们传入的代理类生成器的`generate(ClassLoaderData)`方法，将返回值覆盖之前的`FutureTask`成为真正的缓存。

接下来就是创建代理类了

##### Ⅴ、`abstractClassGenerator.generate()`

```java
//ClassLoaderData.java 
Function<AbstractClassGenerator, Object> load =
            new Function<AbstractClassGenerator, Object>() {
                public Object apply(AbstractClassGenerator gen) {
                    //⭐调用AbstractClassGenerator.generate()方法创建代理类
                    Class klass = gen.generate(ClassLoaderData.this);
                    return gen.wrapCachedClass(klass);
                }
            };
//AbstractClassGenerator.java
protected Class generate(ClassLoaderData data) {
        Class gen;
        Object save = CURRENT.get();
        //当前的代理类生成器存入ThreadLocal中
        CURRENT.set(this);
        try {
            //获取到ClassLoader
            ClassLoader classLoader = data.getClassLoader();
            //判断不能为空
            if (classLoader == null) {
                throw new IllegalStateException("ClassLoader is null while trying to define class " +
                        getClassName() + ". It seems that the loader has been expired from a weak reference somehow. " +
                        "Please file an issue at cglib's issue tracker.");
            }
            synchronized (classLoader) {
             //生成代理类名字
              String name = generateClassName(data.getUniqueNamePredicate()); 
             //缓存中存入这个名字
              data.reserveName(name);
              //当前代理类生成器设置类名
              this.setClassName(name);
            }
            //尝试从缓存中获取类
            if (attemptLoad) {
                try {
                    //要是能获取到就直接返回了  即可能出现并发 其他线程已经加载
                    gen = classLoader.loadClass(getClassName());
                    return gen;
                } catch (ClassNotFoundException e) {
                    // 发现异常说明没加载到 不管了
                }
            }
            //生成字节码
            byte[] b = strategy.generate(this);
            //获取到字节码代表的class的名字
            String className = ClassNameReader.getClassName(new ClassReader(b));
            //核实是否为protect
            ProtectionDomain protectionDomain = getProtectionDomain();
            synchronized (classLoader) { // just in case
                //如果不是protect
                if (protectionDomain == null) {
                    //根据字节码 类加载器 以及类名字  将class加载到内存中
                    gen = ReflectUtils.defineClass(className, b, classLoader);
                } else {
                    //根据字节码 类加载器 以及类名字 以及找到的Protect级别的实体 将class加载到内存中
                    gen = ReflectUtils.defineClass(className, b, classLoader, protectionDomain);
                }
            }
　　　　　　　 //返回生成的class信息
            return gen;
        } catch (RuntimeException e) {
            throw e;
        } catch (Error e) {
            throw e;
        } catch (Exception e) {
            throw new CodeGenerationException(e);
        } finally {
            CURRENT.set(save);
        }
    }
```

到这里就没有什么有意思的东西了,也就是生成代理类的`byte`数组并将其加载到虚拟机里了。其中`GeneratorStrategy`将负责利用ASM往类文件写入方法。

最终生成的代理类如下所示：

```java
//⭐⭐⭐可以看到这个代理类继承了被代理类Dao并且实现了Factory接口。由于实现了Factory接口的setgetCallback(s)方法,所以代理类还可以在生成之后重新指定Callback,相较于JDK动态代理这是其中一亮点
public class Dao$$EnhancerByCGLIB$$27019bc7 extends Dao implements Factory {
    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    private MethodInterceptor CGLIB$CALLBACK_0;
    private MethodInterceptor CGLIB$CALLBACK_1;
    private static Object CGLIB$CALLBACK_FILTER;
    private static final Method CGLIB$update$0$Method;
    private static final MethodProxy CGLIB$update$0$Proxy;
    private static final Object[] CGLIB$emptyArgs;
    private static final Method CGLIB$select$1$Method;
    private static final MethodProxy CGLIB$select$1$Proxy;
    private static final Method CGLIB$equals$2$Method;
    private static final MethodProxy CGLIB$equals$2$Proxy;
    private static final Method CGLIB$toString$3$Method;
    private static final MethodProxy CGLIB$toString$3$Proxy;
    private static final Method CGLIB$hashCode$4$Method;
    private static final MethodProxy CGLIB$hashCode$4$Proxy;
    private static final Method CGLIB$clone$5$Method;
    private static final MethodProxy CGLIB$clone$5$Proxy;

        static {
        CGLIB$STATICHOOK1();
    }
    //初始化被代理类方法的Method对象和MethodProxy对象等，以在真正调用时将这两个值作为参数传给intercept()方法
    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$emptyArgs = new Object[0];
        Class var0 = Class.forName("cglibProxy.cglib2.Dao$$EnhancerByCGLIB$$27019bc7");
        Class var1;
        Method[] var10000 = ReflectUtils.findMethods(new String[]{"equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
        CGLIB$equals$2$Method = var10000[0];
        CGLIB$equals$2$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$2");
        CGLIB$toString$3$Method = var10000[1];
        CGLIB$toString$3$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/String;", "toString", "CGLIB$toString$3");
        CGLIB$hashCode$4$Method = var10000[2];
        CGLIB$hashCode$4$Proxy = MethodProxy.create(var1, var0, "()I", "hashCode", "CGLIB$hashCode$4");
        CGLIB$clone$5$Method = var10000[3];
        CGLIB$clone$5$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/Object;", "clone", "CGLIB$clone$5");
        var10000 = ReflectUtils.findMethods(new String[]{"update", "()V", "select", "()V"}, (var1 = Class.forName("cglibProxy.cglib2.Dao")).getDeclaredMethods());
        CGLIB$update$0$Method = var10000[0];
        CGLIB$update$0$Proxy = MethodProxy.create(var1, var0, "()V", "update", "CGLIB$update$0");
        CGLIB$select$1$Method = var10000[1];
        CGLIB$select$1$Proxy = MethodProxy.create(var1, var0, "()V", "select", "CGLIB$select$1");
    }

    final void CGLIB$update$0() {
        super.update();
    }

    public final void update() {
        //使用的是CGLIB$CALLBACK_1
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_1;
        if (var10000 == null) {
            //根据CallbackFilter的逻辑绑定Callback
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_1;
        }

        if (var10000 != null) {
            //直接调用实现的MethodInterceptor接口
            var10000.intercept(this, CGLIB$update$0$Method, CGLIB$emptyArgs, CGLIB$update$0$Proxy);
        } else {
            //如果某方法没有受到CallbackFilter代理指定，则直接调用父类的原方法
            super.update();
        }
    }


    public final void select() {
        //使用的是CGLIB$CALLBACK_0
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$select$1$Method, CGLIB$emptyArgs, CGLIB$select$1$Proxy);
        } else {
            super.select();
        }
    }

    //····删除了一大堆重写clone()、hashcode()、equals()、toString()
    
     public Callback getCallback(int var1) {
        CGLIB$BIND_CALLBACKS(this);
        MethodInterceptor var10000;
        switch(var1) {
        case 0:
            var10000 = this.CGLIB$CALLBACK_0;
            break;
        case 1:
            var10000 = this.CGLIB$CALLBACK_1;
            break;
        default:
            var10000 = null;
        }

        return var10000;
    }

    public void setCallback(int var1, Callback var2) {
        switch(var1) {
        case 0:
            this.CGLIB$CALLBACK_0 = (MethodInterceptor)var2;
            break;
        case 1:
            this.CGLIB$CALLBACK_1 = (MethodInterceptor)var2;
        }

    }

    public Callback[] getCallbacks() {
        CGLIB$BIND_CALLBACKS(this);
        return new Callback[]{this.CGLIB$CALLBACK_0, this.CGLIB$CALLBACK_1};
    }

    public void setCallbacks(Callback[] var1) {
        this.CGLIB$CALLBACK_0 = (MethodInterceptor)var1[0];
        this.CGLIB$CALLBACK_1 = (MethodInterceptor)var1[1];
    }
    public Dao$$EnhancerByCGLIB$$27019bc7() {
        CGLIB$BIND_CALLBACKS(this);
    }

    public static void CGLIB$SET_THREAD_CALLBACKS(Callback[] var0) {
        CGLIB$THREAD_CALLBACKS.set(var0);
    }

    public static void CGLIB$SET_STATIC_CALLBACKS(Callback[] var0) {
        CGLIB$STATIC_CALLBACKS = var0;
    }

    private static final void CGLIB$BIND_CALLBACKS(Object var0) {
        Dao$$EnhancerByCGLIB$$27019bc7 var1 = (Dao$$EnhancerByCGLIB$$27019bc7)var0;
        if (!var1.CGLIB$BOUND) {
            var1.CGLIB$BOUND = true;
            Object var10000 = CGLIB$THREAD_CALLBACKS.get();
            if (var10000 == null) {
                //这个CGLIB$STATIC_CALLBACKS其实就是我们自己编写的MethodInterceptor
                var10000 = CGLIB$STATIC_CALLBACKS;
                if (var10000 == null) {
                    return;
                }
            }
			//⭐⭐开始按照CallbackFilter的逻辑对不同的方法绑定MethodInterceptor
            Callback[] var10001 = (Callback[])var10000;
            var1.CGLIB$CALLBACK_1 = (MethodInterceptor)((Callback[])var10000)[1];
            var1.CGLIB$CALLBACK_0 = (MethodInterceptor)var10001[0];
        }

    }


}
```

比较有意思就是,`CallbackFilter`的逻辑内嵌在了代理类中,通过代理类内部生成的代码解析哪个方法应该对应哪个`Callback`

### ②、总结

整个cglib动态代理也就是两级映射里的`EnhancerKey`和`LoadingCache`的使用、最终生成的代理类比较有意思了。

#### 1、JDK动态代理与Cglib动态代理的区别

在使用上的区别：

1. JDK动态代理针对的是类的接口中的方法进行代理，而Cglib可以针对任何类或接口的方法进行代理，只是被代理的类和方法不能声明为`final`

2. JDK动态代理生成的代理类实现了 被代理类实现的接口;而Cglib动态代理若针对的是类,则生成的代理类将会继承目标类.而若针对的是接口,则生成的代理类将会实现目标接口.

3. Cglib可以真正做到像JDK动态代理那样基于接口的代理(不是上面所说的无用的接口代理)。如下所示

   ```java
   public class MyMethodInterceptor implements MethodInterceptor
   {
       private HelloService h;
       public MyMethodInterceptor(HelloService h)
       {
           this.h=h;
       }
       /**
        * sub：cglib生成的代理对象
        * method：被代理对象方法
        * objects：方法入参
        * methodProxy: 代理方法
        */
   
       public Object intercept(Object sub, Method method, Object[] args, MethodProxy methodProxy) throws Throwable
       {
           System.out.println("======插入前置通知1======");
   //        Object object = methodProxy.invokeSuper(sub, objects);
           //⭐使用传进来的被代理接口的子对象调用方法
           Object object=method.invoke(h,args);
           System.out.println("======插入后者通知1======");
           return object;
       }
   
   }
   //------------------------------------------------------------------------------------
   public interface HelloService
   {
       void sayHello();
   }
   //------------------------------------------------------------------------------------
   public class HelloServiceImpl implements HelloService
   {
   
       public HelloServiceImpl()
       {
           System.out.println("HelloService构造"); 
       }
   
   
       public void sayHello()
       {
           System.out.println("HelloServiceImpl:sayHello");
       }
   
   }
   //------------------------------------------------------------------------------------
   public class Client
   {
       public static void main(String[] args)
       {
           // 代理类class文件存入本地磁盘方便我们反编译查看源码
           System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D:");
           // 通过CGLIB动态代理获取代理对象的过程
           Enhancer enhancer = new Enhancer();
           // 设置enhancer对象的父类为接口
           enhancer.setSuperclass(HelloService.class);
           // 设置enhancer的回调对象
           enhancer.setCallback(new MyMethodInterceptor(new HelloServiceImpl()));
           // 创建代理对象
           HelloService proxy = (HelloService) enhancer.create();
           // 通过代理对象调用目标方法
           proxy.sayHello();
       }
   }
   ```

   这样就可以基于接口代理了

4. JDK动态代理的被代理类的内部调用不会触发`invoke()`方法(因为被代理类和代理类是两个完全不同的对象),而Cglib动态代理被代理类的内部调用将会触发`intercept()`(因为代理类是被代理类的子类,所以方法会动态分派到子类来执行)

5. Cglib动态代理在生成代理类后还能重新制定`Callback`,而JDK动态代理的代理类一旦生成就无法再指定`InvokecationHandler`了

源码上的区别:

1. 两者都是二级映射缓存,只不过JDK动态代理是`(ClassLoader包装对象→(interface接口包装对象→Class包装对象))`,而Cglib动态代理是`(ClassLoader→(由各类信息生成的EnhancerKey→Class包装对象))`

2. JDK动态代理中,`InvocationHandler`实现类只能通过反射调用原方法;而Cglib动态代理不仅可以通过反射,也可以使用`MethodProxy.invokeSupr()`来调用原方法。这个调用的原理就是Cglib动态代理同时生成了一个`FastClass`实现类,该实现类的`invoke()`实现如下

   ```java
   //Dao$$EnhancerByCGLIB$$27019bc7$$FastClassByCGLIB$$e95c9f53.java
   public Object invoke(int var1, Object var2, Object[] var3) throws InvocationTargetException {
       27019bc7 var10000 = (27019bc7)var2;
       int var10001 = var1;
   
       try {
           switch(var10001) {
   	//·······
           case 15:
               var10000.CGLIB$update$0();
               return null;
           case 16:
               var10000.CGLIB$select$1();
               return null;
                   
        //·······
   
   }
   ```

   最终还是会调用代理类的`CGLIB$update$0()`方法,追溯到代理类的源码就可以看到只有短短的一行

   ```java
   //Dao$$EnhancerByCGLIB$$27019bc7.java 
   final void CGLIB$update$0() {
           super.update();
       }
   ```

   所以通过`FastClass`实现类的调用不是反射,而是正常的子类调用父类方法

> 参考资料
>
> https://www.cnblogs.com/hetutu-5238/p/11996529.html