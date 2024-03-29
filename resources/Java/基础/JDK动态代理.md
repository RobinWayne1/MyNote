# JDK动态代理源码解析

 代理模式是指给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用。

## 一、静态代理

静态代理的代码结构和装饰者模式几乎相同，只不过这两个模式的具体代码逻辑不相同。装饰者模式是对被装饰者对象的功能延伸，而静态代理关注的是对对象的控制访问，对它的用户隐藏对象的具体信息。

### Ⅰ、实现静态代理

实现静态代理有四个步骤：

1. 定义业务接口；
2. 被代理类实现业务接口；
3. 定义代理类并实现业务接口，其中代理类还要有一个被代理类的成员变量，以到真正进行调用时调用被代理类的方法；
4. 最后便可通过客户端进行调用。

#### ①、业务接口

```java
//IUserService.java
public interface IUserService {
    void add(String name);  
}
```

#### ②、被代理类实现业务接口

```java
//UserServiceImpl.java
public class UserServiceImpl implements IUserService{

    @Override
    public void add(String name) {
        System.out.println("向数据库中插入名为：  "+name+" 的用户");
    }

}
```

#### ③、定义代理类并实现业务接口

   因为代理对象和被代理对象需要实现相同的接口。所以代理类源文件UserServiceProxy.java这么写：

```java
//UserServiceProxy.java
public class UserServiceProxy implements IUserService {
    // 被代理对象
    private IUserService target;

    // 通过构造方法传入被代理对象
    public UserServiceProxy(IUserService target) {
        this.target = target;
    }

    @Override
    public void add(String name) {
        System.out.println("准备向数据库中插入数据");
        target.add(name);
        System.out.println("插入数据库成功");

    }
}
```

   由于代理类(`UserServiceProxy` )和被代理类(`UserServiceImpl `)都实现了`IUserService`接口，所以都有add方法，在代理类的add方法中调用了被代理类的add方法，并在其前后各打印一条日志。

#### ④、客户端调用

```java

public class StaticProxyTest {

    public static void main(String[] args) {
        IUserService target = new UserServiceImpl();
        UserServiceProxy proxy = new UserServiceProxy(target);
        proxy.add("robin");
    }
}
```

### Ⅱ、静态代理的缺点

1. 可以看出，被代理类每多编写一个接口，代理类就又要多重写一个方法，增加了代码维护的难度。
2. 代理对象只服务于一种类型的对象，如果要服务多类型的对象，势必要为每一种对象都进行代理，静态代理在程序规模稍大时就无法胜任了。

## 二、动态代理

**所谓动态代理是指：在程序运行期间根据需要动态创建代理类及其实例来完成具体的功能。也就是说，只要创建了一个代理类，那么动态代理就可以被使用于所有类型的对象以及所有接口，弥补了静态代理的缺点**

### Ⅰ、使用JDK动态代理步骤

1. 创建被代理的接口和类；
2. 创建`InvocationHandler`接口的实现类，在`invoke`方法中实现代理逻辑；
3. 通过Proxy的静态方法`newProxyInstance( ClassLoaderloader, Class[] interfaces, InvocationHandler h)`创建一个代理对象
4. 使用代理对象。

#### ①、创建`InvocationHandler`接口的实现类

```java

public class MyInvocationHandler implements InvocationHandler {
    //被代理对象，Object类型
    private Object target;
    
    public MyInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        System.out.println("准备向数据库中插入数据");
        Object returnvalue = method.invoke(target, args);
        System.out.println("插入数据库成功");

        return returnvalue;
    }

}
```

#### ②、通过Proxy的静态方法创建代理对象并使用代理对象

```java

public class DynamicProxyTest {

    public static void main(String[] args) {

        IUserService target = new UserServiceImpl();
        MyInvocationHandler handler = new MyInvocationHandler(target);
        //第一个参数是指定代理类的类加载器（我们传入当前测试类的类加载器）
        //第二个参数是代理类需要实现的接口（我们传入被代理类实现的接口，这样生成的代理类和被代理类就实现了相同的接口）
        //第三个参数是invocation handler，用来处理方法的调用。这里传入我们自己实现的handler
        IUserService proxyObject = (IUserService) Proxy.newProxyInstance(DynamicProxyTest.class.getClassLoader(),
                target.getClass().getInterfaces(), handler);
        proxyObjec.add("robin");
    }

}
```

### Ⅱ、源码分析

先从`Proxy.newProxyInstance()`看起

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
  	//检验h不为空，h为空抛异常
    Objects.requireNonNull(h);
    //接口的类对象拷贝一份
    final Class<?>[] intfs = interfaces.clone();
    //进行一些安全性检查
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    //⭐查询（在缓存中已经有）或生成指定的代理类的class对象。
    Class<?> cl = getProxyClass0(loader, intfs);

    /*
     * Invoke its constructor with the designated invocation handler.
     */
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }
		//获取代理类的构造器
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        //反射创建一个对象
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

#### ①、`getProxyClass0(loader, intfs)`从缓存中获取或创建一个代理类

```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
   //限定代理的接口不能超过65535个
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

	//从WeakCache中获取一个代理类，若缓存中没有则创建一个
    return proxyClassCache.get(loader, interfaces);
}
```

在看`get()`之前先看看`proxyClassCache`这个成员。

```java
//全局共享的一个成员变量
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
    //KeyFactory和ProxyClassFactory是两个比较重要工厂，等下会讲
    proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```

可以看到，代理类的生成逻辑都是依靠`WeakCache`类,看看这个类。

#### ②、`WeakCache`:负责存放代理类缓存(映射结构)以及创建代理类

```java
final class WeakCache<K, P, V> {

    private final ReferenceQueue<K> refQueue
        = new ReferenceQueue<>();
    //⭐缓存,它的映射结构是 ClassLoader被包装后的对象→(被代理类的接口 被包装后的对象→生成的代理类Class对象工厂/被CacheValue包装过后的Class对象)
    private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
        = new ConcurrentHashMap<>();
    private final ConcurrentMap<Supplier<V>, Boolean> reverseMap
        = new ConcurrentHashMap<>(); 
    //这个工厂生成了一个对象,这个对象包装了被代理类的所有接口
    private final BiFunction<K, P, ?> subKeyFactory;
    //这个工厂的apply()方法调用ProxyGenerator的方法构造代理类字节码,然后将字节码提交给类加载器生成代理类Class对象
    private final BiFunction<K, P, V> valueFactory;

  
    public WeakCache(BiFunction<K, P, ?> subKeyFactory,
                     BiFunction<K, P, V> valueFactory) {
        this.subKeyFactory = Objects.requireNonNull(subKeyFactory);
        this.valueFactory = Objects.requireNonNull(valueFactory);
    }
}
```

#### ③、`weakCache.get()`

```java
public V get(K key, P parameter) {
     //检查parameter不为空
    Objects.requireNonNull(parameter);
	//清除无效缓存,后面讲什么是无效缓存
    expungeStaleEntries();
	//由CacheKey类生成一个 包装了ClassLoader的对象
    Object cacheKey = CacheKey.valueOf(key, refQueue);

    //尝试根据包装了ClassLoader的对象从缓存中获取(被代理类的接口 被包装后的对象→生成的代理类Class对象的工厂)映射,若缓存中没有该映射则创建一个空映射并放入缓存
    ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
    if (valuesMap == null) {
        ConcurrentMap<Object, Supplier<V>> oldValuesMap
            = map.putIfAbsent(cacheKey,
                              valuesMap = new ConcurrentHashMap<>());
        if (oldValuesMap != null) {
            //若是第一次生成代理类,则valuesMap就会使一个空的ConcurentHashMap
            valuesMap = oldValuesMap;
        }
    }

    //构造subKey,也就是 由被代理类的接口包装成的对象
    Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    //通过sub-key得到supplier(生成的代理类Class对象),第一次生成代理类那就是null
    Supplier<V> supplier = valuesMap.get(subKey);
    //Factory是Supplier的实现类
    Factory factory = null;

    while (true) {
        if (supplier != null) {
            //⭐生成代理类的入口
            V value = supplier.get();
            if (value != null) {
                return value;
            }
        }
		//第一次创建代理则会创建一个Supplier工厂
        if (factory == null) {
            factory = new Factory(key, parameter, subKey, valuesMap);
        }

        if (supplier == null) {
            //将该工厂放入二级映射中。作用就是占个位,考虑到并发创建代理类的情况，要提前暴露出工厂使得之后的代码在 supplier.get()(sync修饰的方法)中阻塞,以表示此时已有其他线程正在创建该代理类(⭐这里的处理方式和LoadingCahce如出一辙)
            supplier = valuesMap.putIfAbsent(subKey, factory);
            if (supplier == null) {
                // successfully installed Factory
                supplier = factory;
            }
            // else retry with winning supplier
        } else {
            if (valuesMap.replace(subKey, supplier, factory)) {
                // successfully replaced
                // cleared CacheEntry / unsuccessful Factory
                // with our Factory
                supplier = factory;
            } else {
                // retry with current supplier
                supplier = valuesMap.get(subKey);
            }
        }
    }
}
```

虽然逻辑看起来有点乱,但无非就是在构造代理类Class对象的key和代理类Class对象。接下来就开始逐个方法分析。

##### 1、`CacheKey`类与`expungeStaleEntries()`

`CacheKey`类是`WeakCache`的内部类.

```java
//继承了弱引用
private static final class CacheKey<K> extends WeakReference<K> {

    // a replacement for null keys
    private static final Object NULL_KEY = new Object();

    static <K> Object valueOf(K key, ReferenceQueue<K> refQueue) {
        return key == null
               // null key means we can't weakly reference it,
               // so we use a NULL_KEY singleton as cache key
               ? NULL_KEY
               //ClassLoader不为null则生成一个CacheKey，其中key会被弱引用
               : new CacheKey<>(key, refQueue);
    }
	//存放ClassLoader对象的hashcode
    private final int hash;

    private CacheKey(K key, ReferenceQueue<K> refQueue) {
        //⭐这里讲一下WeakReference的这个构造函数。key当然就是代表弱引用的对象，而ReferenceQueue的用法就是当垃圾收集器要GC某个只被WeakReference对象引用的对象时，就会将这个WeakReference对象放入ReferenceQueue中。如此处的ClassLoader，若它只被WeakReference引用则会在要回收时将CacheKey放入ReferenceQueue中。注意一点：map引用CacheKey引用ClassLoader，这样ClassLoader就会被回收
        super(key, refQueue);
        this.hash = System.identityHashCode(key);  // compare by identity
    }
      @Override
        public int hashCode() {
            return hash;
        }

        @Override
        public boolean equals(Object obj) {
            K key;
            return obj == this ||
                   obj != null &&
                   obj.getClass() == this.getClass() &&
                   // cleared CacheKey is only equal to itself
                   (key = this.get()) != null &&
                   // compare key by identity
                   key == ((CacheKey<K>) obj).get();
        }

        void expungeFrom(ConcurrentMap<?, ? extends ConcurrentMap<?, ?>> map,
                         ConcurrentMap<?, Boolean> reverseMap) {
            //此时map的状态是(cacheKey(null)→(Object,Supplier)),classLoader已经被回收,所以将本CacheKey从map中清掉
            ConcurrentMap<?, ?> valuesMap = map.remove(this);
            // remove also from reverseMap if needed
            if (valuesMap != null) {
                for (Object cacheValue : valuesMap.values()) {
                    reverseMap.remove(cacheValue);
                }
            }
        }
```

```java
private void expungeStaleEntries() {
    CacheKey<K> cacheKey;
    //从ReferenceQueue中获取失效的CacheKey
    //要注意,平常我们都是使用AppClassLoader来加载类,这个类就不用说了肯定被某些地方持有着的,所以用AppClassLoader加载的代理类缓存绝对不可能被回收,只有自定义的类加载器才有可能被回收.试验了一天,始终没法进去expungeFrom(),意味着自定义的ClassLoader没有被回收,而我也确定了没有任何地方有对这个ClassLoader引用.唯一的解释就是defineClass0()有缓存对该类加载器的JNI引用了
    while ((cacheKey = (CacheKey<K>)refQueue.poll()) != null) {
        cacheKey.expungeFrom(map, reverseMap);
    }
}
```

所以无效的缓存指的就是 只被`WeakReference`引用的自定义类加载器 的映射

##### 2、 `subKeyFactory.apply()`

该方法其实就是包装接口从而构造一个二级映射的键

```java
private static final class KeyFactory
    implements BiFunction<ClassLoader, Class<?>[], Object>
{
    @Override
    public Object apply(ClassLoader classLoader, Class<?>[] interfaces) {
        switch (interfaces.length) {
            case 1: return new Key1(interfaces[0]); // the most frequent
            case 2: return new Key2(interfaces[0], interfaces[1]);
            case 0: return key0;
            default: return new KeyX(interfaces);
        }
    }
}
```

##### 3、`supplier.get()`

`Factory`是`Supplier`的实现类,直接看这个类

```java
private final class Factory implements Supplier<V> {

    private final K key;
    private final P parameter;
    private final Object subKey;
    private final ConcurrentMap<Object, Supplier<V>> valuesMap;

    Factory(K key, P parameter, Object subKey,
            ConcurrentMap<Object, Supplier<V>> valuesMap) {
        this.key = key;
        this.parameter = parameter;
        this.subKey = subKey;
        this.valuesMap = valuesMap;
    }

    @Override
    public synchronized V get() { // serialize access
        //重新检查一遍,这个是并发检测,控制只有一条线程创建代理类
        Supplier<V> supplier = valuesMap.get(subKey);
        if (supplier != this) {
            // something changed while we were waiting:
            // might be that we were replaced by a CacheValue
            // or were removed because of failure ->
            // return null to signal WeakCache.get() to retry
            // the loop
            return null;
        }
        // else still us (supplier == this)

        // create new value
        V value = null;
        try {
            //⭐核心方法,创建代理类
            value = Objects.requireNonNull(valueFactory.apply(key, parameter));
        } finally {
            if (value == null) { // remove us on failure
                valuesMap.remove(subKey, this);
            }
        }
        // the only path to reach here is with non-null value
        assert value != null;

        //CacheValue继承了WeakReference且实现了Supplier,其中value则会被弱引用
        CacheValue<V> cacheValue = new CacheValue<>(value);

        // put into reverseMap
        reverseMap.put(cacheValue, Boolean.TRUE);

        // try replacing us with CacheValue (this should always succeed)
        if (!valuesMap.replace(subKey, this, cacheValue)) {
            throw new AssertionError("Should not reach here");
        }

        // successfully replaced us with new CacheValue -> return the value
        // wrapped by it
        return value;
    }
}
```

主要逻辑在`ProxyClassFactor`中体现

```java
private static final class ProxyClassFactory
    implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
    //代理类的前缀
    private static final String proxyClassNamePrefix = "$Proxy";

    //表示全局的代理类生成个数,也是用于类名
    private static final AtomicLong nextUniqueNumber = new AtomicLong();

    
    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        //验证被代理的接口
        for (Class<?> intf : interfaces) {
            /*
             * Verify that the class loader resolves the name of this
             * interface to the same Class object.
             */
            Class<?> interfaceClass = null;
            try {
                interfaceClass = Class.forName(intf.getName(), false, loader);
            } catch (ClassNotFoundException e) {
            }
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
            }
            /*
             * Verify that the Class object actually represents an
             * interface.
             */
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            /*
             * Verify that this interface is not a duplicate.
             */
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
        }
		//生成的代理类的包名
        String proxyPkg = null;     // package to define proxy class in
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

        /*
         * Record the package of a non-public proxy interface so that the
         * proxy class will be defined in the same package.  Verify that
         * all non-public proxy interfaces are in the same package.
         */
         //验证所有非公共的接口在同一个包内；公共的就无需处理
         //生成包名和类名的逻辑，包名默认是com.sun.proxy，类名默认是$Proxy 加上一个自增的整数值
         //如果被代理类是 non-public proxy interface ，则用和被代理类接口一样的包名
        for (Class<?> intf : interfaces) {
            int flags = intf.getModifiers();
            if (!Modifier.isPublic(flags)) {
                accessFlags = Modifier.FINAL;
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }

        if (proxyPkg == null) {
            // if no non-public proxy interfaces, use com.sun.proxy package
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
        }

        /*
         * Choose a name for the proxy class to generate.
         */
        long num = nextUniqueNumber.getAndIncrement();
        //代理类的完全限定名，如com.sun.proxy.$Proxy0.class
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        //⭐核心部分，生成代理类的字节码
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
            //生成Class对象
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            /*
             * A ClassFormatError here means that (barring bugs in the
             * proxy class generation code) there was some other
             * invalid aspect of the arguments supplied to the proxy
             * class creation (such as virtual machine limitations
             * exceeded).
             */
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

##### 4、`ProxyGenerator.generateProxyClass()`

```java
public static byte[] generateProxyClass(final String name,
                                            Class<?>[] interfaces,
                                            int accessFlags)
    {
        ProxyGenerator gen = new ProxyGenerator(name, interfaces, accessFlags);
    //生成代理类字节码文件
        final byte[] classFile = gen.generateClassFile();
    if (saveGeneratedFiles) {
        java.security.AccessController.doPrivileged(
        new java.security.PrivilegedAction<Void>() {
            public Void run() {
                try {
                    int i = name.lastIndexOf('.');
                    Path path;
                    if (i > 0) {
                        Path dir = Paths.get(name.substring(0, i).replace('.', File.separatorChar));
                        Files.createDirectories(dir);
                        path = dir.resolve(name.substring(i+1, name.length()) + ".class");
                    } else {
                        path = Paths.get(name + ".class");
                    }
                    Files.write(path, classFile);
                    return null;
                } catch (IOException e) {
                    throw new InternalError(
                        "I/O exception saving generated file: " + e);
                }
            }
        });
    }

    return classFile;
}
```
```java
//这个成员用于记录 代理类方法名→(参数列表,返回值,抛出异常等)
private Map<String, List<ProxyMethod>> proxyMethods = new HashMap<>();

private byte[] generateClassFile() {
    //第一步, 将所有的方法组装成ProxyMethod对象
    //首先为代理类生成toString, hashCode, equals等代理方法,方法内部会将解析后的ProxyMethod加入proxyMethods成员
    addProxyMethod(hashCodeMethod, Object.class);
    addProxyMethod(equalsMethod, Object.class);
    addProxyMethod(toStringMethod, Object.class);
    
    //⭐遍历每一个接口的每一个方法, 并且为其生成ProxyMethod对象并加入proxyMethods
    for (int i = 0; i < interfaces.length; i++) {
        Method[] methods = interfaces[i].getMethods();
        for (int j = 0; j < methods.length; j++) {
            addProxyMethod(methods[j], interfaces[i]);
        }
    }
    //对于具有相同签名的代理方法, 检验方法的返回值是否兼容
    for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
        checkReturnTypes(sigmethods);
    }
    
    //第二步, 组装要生成的class文件的所有的字段信息和方法信息
    try {
        //添加构造器方法
        methods.add(generateConstructor());
        //遍历proxyMethods中的代理方法
        for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
            for (ProxyMethod pm : sigmethods) {
                //添加代理类的静态字段, 例如:private static Method m1;
                fields.add(new FieldInfo(pm.methodFieldName,
                        "Ljava/lang/reflect/Method;", ACC_PRIVATE | ACC_STATIC));
                //添加代理类的代理方法
                methods.add(pm.generateMethod());
            }
        }
        //添加代理类的静态字段初始化方法
        methods.add(generateStaticInitializer());
    } catch (IOException e) {
        throw new InternalError("unexpected I/O Exception");
    }
    
    //验证方法和字段集合不能大于65535
    if (methods.size() > 65535) {
        throw new IllegalArgumentException("method limit exceeded");
    }
    if (fields.size() > 65535) {
        throw new IllegalArgumentException("field limit exceeded");
    }

    //第三步, 写入最终的class文件
    //验证常量池中存在代理类的全限定名
    cp.getClass(dotToSlash(className));
    //验证常量池中存在代理类父类的全限定名, 父类名为:"java/lang/reflect/Proxy"
    cp.getClass(superclassName);
    //验证常量池存在代理类接口的全限定名
    for (int i = 0; i < interfaces.length; i++) {
        cp.getClass(dotToSlash(interfaces[i].getName()));
    }
    //接下来要开始写入文件了,设置常量池只读
    cp.setReadOnly();
    
    ByteArrayOutputStream bout = new ByteArrayOutputStream();
    DataOutputStream dout = new DataOutputStream(bout);
    try {
        //1.写入魔数
        dout.writeInt(0xCAFEBABE);
        //2.写入次版本号
        dout.writeShort(CLASSFILE_MINOR_VERSION);
        //3.写入主版本号
        dout.writeShort(CLASSFILE_MAJOR_VERSION);
        //4.写入常量池
        cp.write(dout);
        //5.写入访问修饰符
        dout.writeShort(ACC_PUBLIC | ACC_FINAL | ACC_SUPER);
        //6.写入类索引
        dout.writeShort(cp.getClass(dotToSlash(className)));
        //7.写入父类索引, 生成的代理类都继承自Proxy
        dout.writeShort(cp.getClass(superclassName));
        //8.写入接口计数值
        dout.writeShort(interfaces.length);
        //9.写入接口集合
        for (int i = 0; i < interfaces.length; i++) {
            dout.writeShort(cp.getClass(dotToSlash(interfaces[i].getName())));
        }
        //10.写入字段计数值
        dout.writeShort(fields.size());
        //11.写入字段集合 
        for (FieldInfo f : fields) {
            f.write(dout);
        }
        //12.写入方法计数值
        dout.writeShort(methods.size());
        //13.写入方法集合
        for (MethodInfo m : methods) {
            m.write(dout);
        }
        //14.写入属性计数值, 代理类class文件没有属性所以为0
        dout.writeShort(0);
    } catch (IOException e) {
        throw new InternalError("unexpected I/O Exception");
    }
    //转换成二进制数组输出
    return bout.toByteArray();
}
```

主要就是为`Proxy$0`类添加`toString()`、`hashcode()`、`构造器`、`equals()`还有代理的接口方法。

#### ⑤、⭐`Proxy$0`类分析

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import proxy.InstanceInterface;

//⭐注意这里,继承了Proxy类
public final class $Proxy0 extends Proxy implements InstanceInterface {
    
    private static Method m1;
    private static Method m2;
    //被代理类的方法实例Method,作为参数传给InvocationHandler的invoke()方法
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
	
    public final void add() throws  {
        try {
            //调用Proxy的InvocationHandler类型成员h的invoke方法
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            //初始化需要代理的方法的Method对象	
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("proxy.InstanceInterface").getMethod("add");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

可以看到`$Proxy0`是一个继承了`Proxy`以及实现了被代理类的接口`InstanceInterface`的类,这就是动态代理的动态所在,它可以在运行时生成 那个在静态代理下只能由程序员编写的代理类。

### Ⅲ、为何JDK动态代理一定要针对接口

如果JDK动态代理针对的是一个类，由于Java的单继承特性，当出现 被代理类继承了其他类的情况时，==对代理类方法的调用中势必就肯定要包括对 被代理类的父类方法的调用==，此时`$Proxy0`就又要继承`Proxy`又要继承父类才能实现这个功能了,所以这不可行,所以才必须使用一个接口来规范 被代理的方法。`CGLib`基于子类的思想也是为了解决这个多继承的问题。

如果硬要基于目标类实现代理，我觉得直接将`InvocationHandler`的`invoke()`方法内联进`$Proxy0`方法中其实也可行,因为继承`Proxy`的作用就是为了得到用户编写的的`InvocationHandler`并调用其`invoke()`方法。只不过这样实现不太合理就是了。