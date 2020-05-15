# ` Tomcat`的NIO部分解析

## 一、Tomcat架构

<img src="E:\Typora\resources\Java\IO\connector架构.png" style="zoom:67%;" />

## 二、源码分析

### Ⅰ、`Connector`的构造方法

```java
public Connector(String protocol) {
    boolean aprConnector = AprLifecycleListener.isAprAvailable() &&
            AprLifecycleListener.getUseAprConnector();

    if ("HTTP/1.1".equals(protocol) || protocol == null) {
        if (aprConnector) {
            protocolHandlerClassName = "org.apache.coyote.http11.Http11AprProtocol";
        } else {
            protocolHandlerClassName = "org.apache.coyote.http11.Http11NioProtocol";
        }
    } else if ("AJP/1.3".equals(protocol)) {
        if (aprConnector) {
            protocolHandlerClassName = "org.apache.coyote.ajp.AjpAprProtocol";
        } else {
            protocolHandlerClassName = "org.apache.coyote.ajp.AjpNioProtocol";
        }
    } else {
        protocolHandlerClassName = protocol;
    }

    // Instantiate protocol handler
    ProtocolHandler p = null;
    try {
        //利用反射创建ProtocolHandler对象
        Class<?> clazz = Class.forName(protocolHandlerClassName);
        p = (ProtocolHandler) clazz.getConstructor().newInstance();
    } catch (Exception e) {
        log.error(sm.getString(
                "coyoteConnector.protocolHandlerInstantiationFailed"), e);
    } finally {
        //将其赋值到成员变量中
        this.protocolHandler = p;
    }

    // Default for Connector depends on this system property
    setThrowOnFailure(Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE"));
}
```

在`Connector`构造方法中,通过指定的协议名称决定使用哪个 `ProtocolHandler`,指定了`HTTP/1.1`后将会使用`Http11NioProtocol`这个`ProtocolHandler`。



我们再来看看这个`Http11NioProtocol`的构造方法





### Ⅱ、启动Tomcat

首先来看`tomcat.start()`方法

```java
//Tomcat.java
public void start() throws LifecycleException {
    getServer();
    server.start();
}
```

#### ①、准备启动Tomcat服务器

```java
//Tomcat.java
//初始化Server与Service
public Server getServer() {

        if (server != null) {
            return server;
        }

        System.setProperty("catalina.useNaming", "false");
		//创建一个Server对象
        server = new StandardServer();

        initBaseDir();

        server.setPort( -1 );
		//创建一个Service对象
        Service service = new StandardService();
        service.setName("Tomcat");
    	//并将该Service对象加入进Server中
        server.addService(service);
        return server;
    }
```

可以看到，`getServer()`就是创建了一个`StandardServer`,并将一个`StandardService`对象放进了该`server`中。在这里出现的`Server`和`Service`与架构图中讲的一致，只不过这两个`Server`和`Service`是一个接口，而`StandardServer`与`StandardService`就是它们的默认实现。

#### ②、组件的生命周期

在讲解`server.start()`之前,我先来讲解一下Tomcat是怎样管理各组件间的生命周期的

<img src="E:\Typora\resources\Java\IO\Tomcat架构.png" style="zoom:80%;" />

如此庞大的一个架构,如何优雅的管理这么多组件的生命周期是一个难题。所以Tomcat使用了一个`Lifeycle`接口用于管理所有组件的生命周期。

<img src="E:\Typora\resources\Java\IO\Lifecycle.png" style="zoom:67%;" />

```java
//LifeCycle.java

/*
*初始化当前组件所管理的组件,如Server的init()就用来初始化Service,而Service的init()就用来初始化Connector
*/
public void init() throws LifecycleException;

/*
*启动组件,该方法由LifeCycleBase类实现
*/
public void start() throws LifecycleException;



public void stop() throws LifecycleException;


public void destroy() throws LifecycleException;
```

```java
//LifeCycleBase.java
@Override
public final synchronized void start() throws LifecycleException {

    if (LifecycleState.STARTING_PREP.equals(state) || LifecycleState.STARTING.equals(state) ||
            LifecycleState.STARTED.equals(state)) {

        if (log.isDebugEnabled()) {
            Exception e = new LifecycleException();
            log.debug(sm.getString("lifecycleBase.alreadyStarted", toString()), e);
        } else if (log.isInfoEnabled()) {
            log.info(sm.getString("lifecycleBase.alreadyStarted", toString()));
        }

        return;
    }
	//对state的判断会进来这个分支,在启动当前对象的子组件之前先初始化其子组件
    if (state.equals(LifecycleState.NEW)) {
        init();
    } else if (state.equals(LifecycleState.FAILED)) {
        stop();
    } else if (!state.equals(LifecycleState.INITIALIZED) &&
            !state.equals(LifecycleState.STOPPED)) {
        invalidTransition(Lifecycle.BEFORE_START_EVENT);
    }

    try {
        setStateInternal(LifecycleState.STARTING_PREP, null, false);
        //启动子组件,由实现类实现的方法
        startInternal();
        if (state.equals(LifecycleState.FAILED)) {
            // This is a 'controlled' failure. The component put itself into the
            // FAILED state so call stop() to complete the clean-up.
            stop();
        } else if (!state.equals(LifecycleState.STARTING)) {
            // Shouldn't be necessary but acts as a check that sub-classes are
            // doing what they are supposed to.
            invalidTransition(Lifecycle.AFTER_START_EVENT);
        } else {
            setStateInternal(LifecycleState.STARTED, null, false);
        }
    } catch (Throwable t) {
        // This is an 'uncontrolled' failure so put the component into the
        // FAILED state and throw an exception.
        handleSubClassException(t, "lifecycleBase.startFail", toString());
    }
}
@Override
public final synchronized void init() throws LifecycleException {
    if (!state.equals(LifecycleState.NEW)) {
        invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
    }

    try {
        setStateInternal(LifecycleState.INITIALIZING, null, false);
        //初始化子组件,由实现类实现的方法
        initInternal();

        setStateInternal(LifecycleState.INITIALIZED, null, false);
    } catch (Throwable t) {
        handleSubClassException(t, "lifecycleBase.initFail", toString());
    }
}
```

由上面的方法调用可以看到,每个子组件的生命周期都由父组件通过调用**子组件的`LifeCycle`实现方法**来进行管理。当对某组件调用`start()`时,该组件就会先调用的`init()`初始化自己的子组件,而子组件又会调用`init()`来初始化它的子组件。调用`startInternal()`同理,实现该方法的组件也同样会调用子组件的`start()`启动子组件。所以当对`server`调用`start()`时,牵一发而动全身,所有组件都将会被启动。

#### ③、启动Tomcat

所以具体的启动流程如下

<img src="E:\Typora\resources\Java\IO\Tomcat启动流程.png" style="zoom:150%;" />

### Ⅲ、`EndPoint`

上层的组件的存在意义主要就是根据功能划分子组件以方便管理,并没有涉及到具体的NIO代码逻辑。而这个逻辑从实现了`EndPoint`的`NioEndPoint`开始体现。

#### ①、`init()`

先来看由`protocolHandler.init()`中调用的`endPoint.init()`

```java
//AbstractEndPoint.java
public final void init() throws Exception {
    if (bindOnInit) {
        //主要逻辑体现在该方法中
        bind();
        bindState = BindState.BOUND_ON_INIT;
    }
    if (this.domain != null) {
        // Register endpoint (as ThreadPool - historical name)
        oname = new ObjectName(domain + ":type=ThreadPool,name=\"" + getName() + "\"");
        Registry.getRegistry(null, null).registerComponent(this, oname, null);

        for (SSLHostConfig sslHostConfig : findSslHostConfigs()) {
            registerJmx(sslHostConfig);
        }
    }
}
//NioEndPoint.java
	 /**
     * Initialize the endpoint.
     */
    @Override
    public void bind() throws Exception {
        //实例化ServerSocketChannel
        initServerSocket();
	  →→{
         serverSock = ServerSocketChannel.open();
        socketProperties.setProperties(serverSock.socket());
        InetSocketAddress addr = (getAddress()!=null?new	 		InetSocketAddress(getAddress(),getPort()):new InetSocketAddress(getPort()));
        serverSock.socket().bind(addr,getAcceptCount());
          //注意看这里,将channel设置成了阻塞模式(想了解阻塞和非阻塞含义的话请去NIO源码解析看)
        serverSock.configureBlocking(true); //mimic APR behavior 
      	}
        //Acceptor线程默认为一条,使用多个 acceptor 线程并不见得性能会更好
        if (acceptorThreadCount == 0) {
            // FIXME: Doesn't seem to work that well with multiple accept threads
            acceptorThreadCount = 1;
        }
    // poller 线程数，默认值定义如下，所以在多核模式下，默认为 2
    // pollerThreadCount = Math.min(2,Runtime.getRuntime().availableProcessors());
        if (pollerThreadCount <= 0) {
            //minimum one poller thread
            pollerThreadCount = 1;
        }
        
        setStopLatch(new CountDownLatch(pollerThreadCount));

        //SSL相关,忽略
        initialiseSsl();

        selectorPool.open();
    }
```

`NioEndPoint()`的`init()`做的事比较简单,就是根据用户配置的端口号和ip创建一个`ServerSocketChannel`实例,比较需要关注的一点就是,该channel被设置成了阻塞模式，用法和我的NIO DEMO用法不同，这个逻辑会在`Acceptor`中体现。

#### ②、`start()`

```java
//AbstractEndPoint.java
public final void start() throws Exception {
    //init()的时候已经绑定过了,此时就不会再绑定
    if (bindState == BindState.UNBOUND) {
        bind();
        bindState = BindState.BOUND_ON_START;
    }
    //NioEndPoint实现了该方法
    startInternal();
}
//NioEndPoint.java
 	 /**
     * Start the NIO endpoint, creating acceptor, poller threads.
     */
    @Override
    public void startInternal() throws Exception {

        if (!running) {
            running = true;
            paused = false;
			// 以下几个是缓存用的，为了减少 new 很多对象出来
            processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getProcessorCache());
            eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                            socketProperties.getEventCache());
            nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getBufferPool());

             // 创建【工作线程池】，Tomcat 自己包装了一下 ThreadPoolExecutor,默认 corePoolSize 10, maxPoolSize 200,Processor组件的任务将会提交到该线程池中以处理请求(processKey()方法中)
            if ( getExecutor() == null ) {
                createExecutor();
            }
			// 设置一个栅栏（tomcat 自定义了类 LimitLatch），控制最大的连接数，默认是 10000
            initializeConnectionLatch();

         	//⭐开启Poller线程
            pollers = new Poller[getPollerThreadCount()];
            for (int i=0; i<pollers.length; i++) {
                pollers[i] = new Poller();
                Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
                pollerThread.setPriority(threadPriority);
                pollerThread.setDaemon(true);
                pollerThread.start();
            }
			//⭐开启Acceptor线程
            startAcceptorThreads();
        }
    }

```

`NioEndPoint`的`start()`方法主要就做了三件事:

1. 创建线程池以处理请求,而处理请求这个任务是放在了`Processor`组件中完成的
2. 开启`Poller`线程
3. 开启`Acceptor`线程

### Ⅳ、`Acceptor`线程

`Acceptor`类其实就相当于`Reactor`模型里的连接建立处理器,如下

<img src="E:\Typora\resources\Java\IO\NIO网络模型.png" style="zoom:60%;" />

但是由于Tomcat的`ServerSocketChannel`设置成了阻塞模式,这不同于我们的DEMO,因为Tomcat的`ServerSocketChannel`并没有(也不能,因为`Selector`不支持阻塞模式的channel)加入进`Selector`中,所以现在相当于`Acceptor Handler`与`Client`是直连着的。

<img src="E:\Typora\resources\Java\IO\Tomcat的NIO模型.png" style="zoom:67%;" />

只要弄清楚了这个架构，其实Tomcat的NIO还是非常简单的。

接下来看在`NioEndPoint.start()`中启动的`Acceptor`线程做了什么

```java
@Override
public void run() {

    int errorDelay = 0;

    // 只要 endpoint 处于 running，这里就一直循环
    while (endpoint.isRunning()) {

        // 如果 endpoint 处于 pause 状态，这边 Acceptor线程 用一个 while 循环将自己也挂起
        while (endpoint.isPaused() && endpoint.isRunning()) {
            state = AcceptorState.PAUSED;
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                // Ignore
            }
        }
		// endpoint 结束了，Acceptor线程 自然也要结束
        if (!endpoint.isRunning()) {
            break;
        }
        state = AcceptorState.RUNNING;

        try {
            // 如果此时达到了最大连接数(之前我们说过，默认是10000)，就等待
            endpoint.countUpOrAwaitConnection();

            // Endpoint might have been paused while waiting for latch
            // If that is the case, don't accept new connections
            if (endpoint.isPaused()) {
                continue;
            }

            U socket = null;
            try {
                //调用ServerSocketChannel.accept()获取新的连接建立请求.如果没有新的连接建立请求,那么这个方法将会阻塞
                socket = endpoint.serverSocketAccept();
            } catch (Exception ioe) {
                // We didn't get a socket
                endpoint.countDownConnection();
                if (endpoint.isRunning()) {
                    // Introduce delay if necessary
                    errorDelay = handleExceptionWithDelay(errorDelay);
                    // re-throw
                    throw ioe;
                } else {
                    break;
                }
            }
            // Successful accept, reset the error delay
            errorDelay = 0;

            // Configure the socket
            if (endpoint.isRunning() && !endpoint.isPaused()) {
                //⭐往Poller中注册SocketChannel的逻辑就在这里
                if (!endpoint.setSocketOptions(socket)) {
                    endpoint.closeSocket(socket);
                }
            } else {
                endpoint.destroySocket(socket);
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            String msg = sm.getString("endpoint.accept.fail");
            // APR specific.
            // Could push this down but not sure it is worth the trouble.
            if (t instanceof Error) {
                Error e = (Error) t;
                if (e.getError() == 233) {
                    // Not an error on HP-UX so log as a warning
                    // so it can be filtered out on that platform
                    // See bug 50273
                    log.warn(msg, t);
                } else {
                    log.error(msg, t);
                }
            } else {
                    log.error(msg, t);
            }
        }
    }
    state = AcceptorState.ENDED;
}
//NioEndPoint.java
    @Override
    protected boolean setSocketOptions(SocketChannel socket) {
        // Process the connection
        try {
            //将SocketChannel设置成非阻塞模式,对连接可读/可写事件就以多路复用IO的方式获取数据
            socket.configureBlocking(false);
            Socket sock = socket.socket();
            socketProperties.setProperties(sock);

            NioChannel channel = nioChannels.pop();
            if (channel == null) {
                SocketBufferHandler bufhandler = new SocketBufferHandler(
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer());
                if (isSSLEnabled()) {
                    channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
                } else {
                    channel = new NioChannel(socket, bufhandler);
                }
            } else {
                channel.setIOChannel(socket);
                channel.reset();
            }
            // getPoller0() 会选取所有 poller 中的一个 poller
            getPoller0().register(channel);
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            try {
                log.error("",t);
            } catch (Throwable tt) {
                ExceptionUtils.handleThrowable(tt);
            }
            // Tell to close the socket
            return false;
        }
        return true;
    }

```

```java
//Poller.java
public void register(final NioChannel socket) {
    socket.setPoller(this);
    //NioSocketWrapper将会作为Attachment在Selector中传递
    NioSocketWrapper ka = new NioSocketWrapper(socket, NioEndpoint.this);
    socket.setSocketWrapper(ka);
    ka.setPoller(this);
    ka.setReadTimeout(getConnectionTimeout());
    ka.setWriteTimeout(getConnectionTimeout());
    ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
    ka.setSecure(isSSLEnabled());
    PollerEvent r = eventCache.pop();
    //看清楚这个是NioSocketWrapper的方法而不是SelectionKey的方法
    ka.interestOps(SelectionKey.OP_READ);
    //在调用PollerEvent.run()时会将NioSocketWrapper与Poller.selector()一同放进该SocketChannel中
    //设置PollerEvent的ops为OP_REGISTER,在PollerEvent.run()时将会根据此成员选择向Selector注册该socketChannel
    if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
    else r.reset(socket,ka,OP_REGISTER);
    //向SynchronizedQueue类型的成员events加入该PollerEvent,SynchronizedQueue是一个线程安全的队列,这就是Acceptor和Poller的共享成员,沟通桥梁
    //⭐PollerEvent的作用其实就是包装SocketChannel和NioSocketWrapper,之后由Poller线程中将这两个对象放入Selector.而不在Acceptor线程中就将该channel放入Selector个人认为是性能考虑吧
    addEvent(r);
}
```

### Ⅴ、`Poller`线程

先来看看`Poller`的成员变量

```java
public class Poller implements Runnable {

    public Poller() throws IOException {
        // 每个 poller 开启一个 Selector
        this.selector = Selector.open();
    }
    private Selector selector;
    // events 队列，此类的核心,Acceptor和Poller的共享变量
    private final SynchronizedQueue<PollerEvent> events =
            new SynchronizedQueue<>();
}
```

```java
//Poller.java
@Override
public void run() {
    // Loop until destroy() is called
    while (true) {

        boolean hasEvents = false;

        try {
            if (!close) {
                //⭐下面将介绍此方法,主要功能是将SocketChannel注册进Selector
                hasEvents = events();
                if (wakeupCounter.getAndSet(-1) > 0) {
                    //if we are here, means we have other stuff to do
                    //do a non blocking select
                    keyCount = selector.selectNow();
                } else {
                    //select(1),即若没有channel IO完毕,则阻塞1s之后返回
                    keyCount = selector.select(selectorTimeout);
                }
                wakeupCounter.set(0);
            }
            if (close) {
                events();
                timeout(0, false);
                try {
                    selector.close();
                } catch (IOException ioe) {
                    log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                }
                break;
            }
        } catch (Throwable x) {
            ExceptionUtils.handleThrowable(x);
            log.error("",x);
            continue;
        }
        //either we timed out or we woke up, process events first
        if ( keyCount == 0 ) hasEvents = (hasEvents | events());

        Iterator<SelectionKey> iterator =
            keyCount > 0 ? selector.selectedKeys().iterator() : null;
        //在这里也就是我们打的NIOServer的DEMO部分了,遍历SelectonKeys并取出其SocketChannel,处理其中的数据
        while (iterator != null && iterator.hasNext()) {
            SelectionKey sk = iterator.next();
            NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
            // Attachment may be null if another thread has called
            // cancelledKey()
            if (attachment == null) {
                iterator.remove();
            } else {
                iterator.remove();
                //⭐处理SocketChannel发来的数据
                processKey(sk, attachment);
            }
        }//while

        //process timeouts
        timeout(keyCount,hasEvents);
    }//while

    getStopLatch().countDown();
}
        
protected void processKey(SelectionKey sk, NioSocketWrapper attachment) {
    try {
        if ( close ) {
            cancelledKey(sk);
        } else if ( sk.isValid() && attachment != null ) {
            if (sk.isReadable() || sk.isWritable() ) {
                if ( attachment.getSendfileData() != null ) {
                    processSendfile(sk,attachment, false);
                } else {
                    // unregister 相应的 interest set,即将SocketChannel的interest set设置为0
                    // 如接下来是处理 SocketChannel 进来的数据，那么就不再监听该 channel 的 OP_READ 事件(⭐注意看这个方法,将SelectionKey的感兴趣集合改为0,Selector就会取消在epoll中的这个fd)
                    unreg(sk, attachment, sk.readyOps());
                    boolean closeSocket = false;
                    // Read goes before write
                    if (sk.isReadable()) {
                         // ⭐处理读事件
                        if (!processSocket(attachment, SocketEvent.OPEN_READ, true)) {
                            closeSocket = true;
                        }
                    }
                    if (!closeSocket && sk.isWritable()) {
                        if (!processSocket(attachment, SocketEvent.OPEN_WRITE, true)) {
                            closeSocket = true;
                        }
                    }
                    if (closeSocket) {
                        cancelledKey(sk);
                    }
                }
            }
        } else {
            //invalid key
            cancelledKey(sk);
        }
    } catch ( CancelledKeyException ckx ) {
        cancelledKey(sk);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error("",t);
    }
}
//很简单的逻辑,创建线程池处理请求
  public boolean processSocket(SocketWrapperBase<S> socketWrapper,
            SocketEvent event, boolean dispatch) {
        try {
            if (socketWrapper == null) {
                return false;
            }
            SocketProcessorBase<S> sc = processorCache.pop();
            if (sc == null) {
                sc = createSocketProcessor(socketWrapper, event);
            } else {
                sc.reset(socketWrapper, event);
            }
            Executor executor = getExecutor();
            if (dispatch && executor != null) {
                executor.execute(sc);
            } else {
                sc.run();
            }
        } catch (RejectedExecutionException ree) {
            getLog().warn(sm.getString("endpoint.executor.fail", socketWrapper) , ree);
            return false;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            // This means we got an OOM or similar creating a thread, or that
            // the pool and its queue are full
            getLog().error(sm.getString("endpoint.process.fail"), t);
            return false;
        }
        return true;
    }
```

```java
//PollerEvents.java
public boolean events() {
            boolean result = false;

            PollerEvent pe = null;
    	//取出 events 中的所有PollerEvents
            for (int i = 0, size = events.size(); i < size && (pe = events.poll()) != null; i++ ) {
                result = true;
                try {
                    //调用其run()方法
                    pe.run();
                    pe.reset();
                    if (running && !paused) {
                        eventCache.push(pe);
                    }
                } catch ( Throwable x ) {
                    log.error("",x);
                }
            }

            return result;
        }

	 @Override
        public void run() {
            // 对于新来的连接，Acceptor线程会新建一个将ops设置为OP_REGISTER的PollerEvent将其放入events 成员变量中,ops的用处就在这里体现
            if (interestOps == OP_REGISTER) {
                try {
                     // 将这个新连接 SocketChannel 注册到该 poller 的 Selector 中，
            		// 设置监听 OP_READ 事件，
            		// 将 socketWrapper 设置为 attachment 进行传递
                    socket.getIOChannel().register(
                            socket.getPoller().getSelector(), SelectionKey.OP_READ, socketWrapper);
                } catch (Exception x) {
                    log.error(sm.getString("endpoint.nio.registerFail"), x);
                }
            } else {
               //省略了很多代码
        } 		
```

