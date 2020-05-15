# NIO源码解析

我们以demo的代码为顺序看看NIO是怎样运作的

```java
public class NIOServer
{
    public static void main(String[] args) throws IOException
    {
        /**
         * 实例化选择器
         */
        Selector selector = Selector.open();
        /**
         * 开启服务端套接字管道
         */
        ServerSocketChannel server = ServerSocketChannel.open();
        /**
         * 绑定端口
         */
        server.socket().bind(new InetSocketAddress(8080));
        /**
         * 记得设置非阻塞模式
         */
        server.configureBlocking(false);

        /**
         * 将ServerSocketChannel注册进selector中,并将其设定为连接建立时事件
         * 只能设置为OP_ACCEPT,这是channel内部规定了的.同样SocketChannel不能设置为连接建立事件,只能设置为其他三样
         */
        //⭐⭐⭐⭐其中这个SelectionKey.OP_ACCEPT称为感兴趣的集合,作用是在该channel的网络IO完成后   事件分离器根据该channel的感兴趣集合 从而调用对应的事件处理器
        //⭐注意一下这奇怪的调用,将Selector放进了ServerSocketChannel中,而不是将ServerSocketChannel放进了Selector中
        server.register(selector, SelectionKey.OP_ACCEPT);

        for (; ; )
        {
            // 判断是否有事件准备好。如果没有任何通道准备好，这个方法会阻塞，直到至少有一个通道准备好。
            /**
             * 出现了个问题:一旦有一个SocketChannel准备好之后,以后的循环select()就不会再阻塞,
             * 而是返回1,即这个已经读过了的SocketChannel在第二次循环中还会在当selectionKeys中出现,
             * 以至于若客户端连接断开而服务器端继续执行read()则会报错
             * 所以要使得不再select()这个SocketChannel只能手动断开连接(大多数情况下还是需要此连接的),即socketChannel.close()
             */
            /**⭐⭐⭐⭐
             *一定要先调用此方法.这个方法的底层实现会调用内核的epoll_ctl(将要注册的sockets加入到一个称作eventpoll的等待队列中,
             * 没看错,register只是将要注册的事件放进了一个HashSet成员变量中,并没有真正注册进内核)
             * 和epoll_wait(内核中等待感兴趣事件的生成,导致线程阻塞),只有调用了这个方法才能够得到IO已完成的channel
             */
            int readyChannels = selector.select();
            if (readyChannels == 0)
            {
                continue;
            }
            /**
             * 要注意一定要先调用select()方法
             */
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> i = selectionKeys.iterator();
			//⭐下面的if判断就相当于事件分离器
            while (i.hasNext())
            {
                SelectionKey currentEvents = i.next();
                //通过迭代器将集合内的这个元素删除,以免重复迭代
                i.remove();
                //有客户端建立连接
                //⭐真正操作时是需要用线程池开启线程操作的
                if (currentEvents.isAcceptable())
                {
                    SocketChannel clientSocket = server.accept();
                    clientSocket.configureBlocking(false);
                    //建立了连接并不代表有数据可读
                    /**⭐⭐⭐⭐
                     * 请注意,调用socketChannel.connect()即可select到ServerSocketChannel,这就是已连接,但此时客户端还没有对SocketChannel进行写操作
                     * ,所以此时的SocketChannel还没有数据可读的,所以要将SocketChannel注册进Selector中等待客户端的IO完成
                     */
                    clientSocket.register(selector, SelectionKey.OP_READ);
                }
                //真正处理有数据的SocketChannel的代码块
                else if (currentEvents.isReadable())
                {
                    SocketChannel clientSocket = (SocketChannel) currentEvents.channel();
                    ByteBuffer buffer = ByteBuffer.allocate(1024);

                    int nums = clientSocket.read(buffer);
                    //为了解决上面所说的问题,在第二次执行到这里时就把过期的SocketChannel连接断开
                    if (nums > 0)
                    {
                        System.out.println(new String(buffer.array()));
                        ByteBuffer returnBuffer = ByteBuffer.wrap("true".getBytes());
                        clientSocket.write(returnBuffer);
                    } else
                    {
                        clientSocket.close();
                    }
                }
            }
        }
    }
}
```

### 一、`Selector.open()`

```java
// Selector.java
public static Selector open() throws IOException {
    // 这里的静态方法provider()会使用DefaultSelectorProvider.create():方法根据系统选择一个SelectorProvider,注意不是单例的
    // windows平台的话是WindowsSelectorProvider，
    // Linux平台是一个EPollSelectorProvider，这里主要分析Linux平台下的
    // 之后openSelector方法(一会看下面)会返回一个EPollSelectorImpl作为Selector的实现，我们一般提及的Selector就是它了
    return SelectorProvider.provider().openSelector();
}
 public AbstractSelector openSelector() throws IOException {
     //返回一个Selector实现
        return new EPollSelectorImpl(this);
}
```

<img src="E:\Typora\resources\Java\IO\EPollSelectorImpl.png" style="zoom:67%;" />

这就是`EPollSelectorImpl`的类结构,接下来要注意各个构造方法的调用.

```java
class EPollSelectorImpl
    extends SelectorImpl
{

    // File descriptors used for interrupt
    protected int fd0;
    protected int fd1;

    // The poll object
    EPollArrayWrapper pollWrapper;

    // Maps from file descriptors to keys
    private Map<Integer,SelectionKeyImpl> fdToKey;

    // True if this Selector has been closed
    private volatile boolean closed = false;

    // Lock for interrupt triggering and clearing
    private final Object interruptLock = new Object();
    private boolean interruptTriggered = false;

    /**
     * Package private constructor called by factory method in
     * the abstract superclass Selector.
     */
    EPollSelectorImpl(SelectorProvider sp) throws IOException {
        super(sp);
        long pipeFds = IOUtil.makePipe(false);
        //一些文件描述符的初始化
        fd0 = (int) (pipeFds >>> 32);
        fd1 = (int) pipeFds;
        //创建EPollArrayWrapper实例,该实例封装了EPOLL的JNI调用
        //创建实例过程中会顺带调用epoll_create创建一个eventpoll对象
        pollWrapper = new EPollArrayWrapper();
        //初始化用作中断的文件描述符,不太懂什么意思
        pollWrapper.initInterrupt(fd0, fd1);
        fdToKey = new HashMap<>();
    }
}

//---------------------------------------------------------------------------------------
public abstract class SelectorImpl
    extends AbstractSelector
{

    //⭐存放已经IO完成的SelectionKey
    protected Set<SelectionKey> selectedKeys;

    //⭐存放已注册进Selector的SelectionKey
    protected HashSet<SelectionKey> keys;

    //⭐上面两个key是NIO框架内部使用的,下面这两个就是用户可以调用selector.selectedKeys();获取的key只是下面的两个成员返回给用户后不允许用户代码对其做某些修改
    private Set<SelectionKey> publicKeys;             // Immutable
    private Set<SelectionKey> publicSelectedKeys;     // Removal allowed, but not addition

    protected SelectorImpl(SelectorProvider sp) {
        super(sp);
        //初始化存放SelectionKey的两个set
        keys = new HashSet<SelectionKey>();
        selectedKeys = new HashSet<SelectionKey>();
        if (Util.atBugLevel("1.4")) {
            publicKeys = keys;
            publicSelectedKeys = selectedKeys;
        } else {
            //注意publicKeys和keys内部指向同一个对象
            publicKeys = Collections.unmodifiableSet(keys);
            //注意publicSelectedKeys和selectedKeys指向同一个对象
            publicSelectedKeys = Util.ungrowableSet(selectedKeys);
        }
    }
}

//---------------------------------------------------------------------------------------

public abstract class AbstractSelector
    extends Selector
{ 		//将创建Selector实例的SelectorProvider放进Selector成员中
		protected AbstractSelector(SelectorProvider provider) {
        this.provider = provider;
    }
}

```

这就是`open()`方法,其实并没有干什么实事,主要还是各个成员变量的初始化

### 二、`server.register()`

```java
//SelectableChannel.java,是ServerSocketChannel的父类
public final SelectionKey register(Selector sel, int ops)
        throws ClosedChannelException
    {
    	//会调用实现类的register()方法
        return register(sel, ops, null);
    }
public abstract SelectionKey register(Selector sel, int ops, Object att)
        throws ClosedChannelException;

//---------------------------------------------------------------------------------------

//AbstractSelectableChannel.java,是ServerSocketChannel的父类,SelectableChannel的子类,它实现了register()方法
public abstract class AbstractSelectableChannel
    extends SelectableChannel
{
	// Keys that have been created by registering this channel with selectors.
    // They are saved because if this channel is closed the keys must be
    // deregistered.  Protected by keyLock.
    //意思就是当用户要关闭channel时,AbstractSelectableChannel.close()方法也要将该channel与Selector之间的关联SelectionKey给取消掉.等下就将channel.close()
//注意processDeregisterQueue()方法,在SelectorImpl中
    private SelectionKey[] keys = null;

public final SelectionKey register(Selector sel, int ops,
                                       Object att)
        throws ClosedChannelException
    {
        synchronized (regLock) {
            if (!isOpen())
                throw new ClosedChannelException();
            if ((ops & ~validOps()) != 0)
                throw new IllegalArgumentException();
            if (blocking)
                throw new IllegalBlockingModeException();
            //注意一下这个细节
            SelectionKey k = findKey(sel);
           →→→{
        		synchronized (keyLock) 
                {
            		if (keys == null)
                		return null;
            		for (int i = 0; i < keys.length; i++)
                        //⭐如果发现之前channel已经关联过该Selector了,那么就使用已经关联过的SelectionKey
                        //为什么要使用已关联过的SelectionKey呢?要明确的是,当一个在Selector的channel完成网络IO时, 一个Reactor模型 的事件分离器只能将该事件分配给其中一个事件处理器,因为对于一个Selector来说，一个IO事件总不能是又可读又可写、又是可连接的吧(这是由epoll_ctl的event参数决定的,只能监听一种事件)
                        /*
                        *就譬如下面这种写法，
                        *clientSocket.register(selector, SelectionKey.OP_READ);
                    	*clientSocket.register(selector, SelectionKey.OP_WRITE);
                        *在这个方法中，SelectionKey.OP_WRITE就会覆盖掉SelectionKey.OP_READ这个                         *感兴趣集合
                        */
                        //如果实在想一个IO事件调用多个事件处理器的话，那就只能创建多个Reactor模型，也就是多个Selector了
                		if ((keys[i] != null) && (keys[i].selector() == sel))
                    		return keys[i];
            		return null;
        		}
    		 }
            if (k != null) {
                k.interestOps(ops);
                k.attach(att);
            }
            
            if (k == null) {
             //来到这里则说明之前没有创建过该 channel与Selector 的关联，则创建一个SelectionKey
                synchronized (keyLock) {
                    if (!isOpen())
                        throw new ClosedChannelException();
                 //看这里，变成了 selector.register() ,这里才是真正将channel注册进Selector的地方
                    k = ((AbstractSelector)sel).register(this, ops, att);
                    //往AbstractSelectableChannel.keys[]中添加一个SelectionKey,记录该关联
                    addKey(k);
                }
            }
            return k;
        }
    }
}
```

上面的代码都在 `Channel`的相关类中,其流程也就是维护该`channel`与`Selector`的关系（所以才会出现这个奇怪的调用）。

接下来来到实现了`AbstractSelector`的`register()`的`SelectorImpl`

```java
//SelectorImpl.java
  protected final SelectionKey register(AbstractSelectableChannel ch,
                                          int ops,
                                          Object attachment)
    {
        if (!(ch instanceof SelChImpl))
            throw new IllegalSelectorException();
      //创建一个SelectionKeyImpl实例,并将对应的channel与selector关系放入其中
        SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
      //⭐添加需要传递给事件处理器的对象,其用法就是在注册期间将对象放入该IO事件,然后当该事件IO完成交由事件处理器处理时,再将该对象取出
        k.attach(attachment);
        synchronized (publicKeys) {
            //子类实现的方法
            implRegister(k);
        }
      //为SelectionKey设置感兴趣事件
        k.interestOps(ops);
        return k;
    }
//---------------------------------------------------------------------------------------
//EpollSelectorImpl.java
 protected void implRegister(SelectionKeyImpl ski) {
        if (closed)
            throw new ClosedSelectorException();
        SelChImpl ch = ski.channel;
        int fd = Integer.valueOf(ch.getFDVal());
     //⭐向Map中存放文件描述符与SelectionKey的映射,只需要知道每个channel都被看成一个文件,而每个channel都会有一个独特的文件描述符fd(可以的话讲讲channel初始化)
        fdToKey.put(fd, ski);
     //⭐将文件描述符加入EPollArrayWrapper实例,注意:add()方法只是将fd记录到了EPollArrayWrapper.eventsLow[]中,甚至连感兴趣的事件event都没有设置(默认为IO_READ),并没有将该fd注册进操作系统epoll
        pollWrapper.add(fd);
     	//往AbstractSelectableChannel.keys[]中添加一个SelectionKey,记录该关联
        keys.add(ski);
    }
```

### 三、EPollArrayWrapper

接下来我们先初窥一下`EPollArrayWrapper`实例。

在此之前要先知道什么是fd，下面是引用https://blog.csdn.net/cywosp/article/details/38965239的一段话

> 在Linux系统中一切皆可以看成是文件，文件又可分为：普通文件、目录文件、链接文件和设备文件。文件描述符（file descriptor）是内核==为了高效管理已被打开的文件所创建的索引==，其是一个非负整数（通常是小整数），用于指代被打开的文件，所有执行I/O操作的系统调用都通过文件描述符。
> 

我们只需要知道Linux的Socket和epoll都是一个文件，并且都要通过文件描述符fd来访问就可以了。

```java
class EPollArrayWrapper {

    // EPOLL_EVENTS(感兴趣事件),是epoll_ctl方法的第四个参数,只有initInterrupt()会使用到这个成员,在EPollSelectorImpl初始化时会调用
    private static final int EPOLLIN      = 1;
    // Special value to indicate that an update should be ignored
    //当某channel被删除时,就会将eventsLow[channel.fd]置KILLED
    private static final byte  KILLED = (byte)-1;
	//而其它的EPOLL_EVENTS就是这四个事件了
    /*
    public static final int OP_READ = 1;
    public static final int OP_WRITE = 4;
    public static final int OP_CONNECT = 8;
    public static final int OP_ACCEPT = 16;
    */
    //注意一个细节:在调用SelectionKeyImpl.intrestOps()方法修改感兴趣事件时,如果该SelectionKey管理的是ServersocketChannel,那么方法内调用的channel.translateAndSetInterestOps()就会将ops从16改为1,之后调用epoll_ctl时用的参数就是1(并无大碍,因为ServerSocketChannel只接收ACCEPT事件,ops不用代表READ事件)
//---------------------------------------------------------------------------------------   
    // Used by release and updateRegistrations to track whether a file
    // descriptor is registered with epoll.
    //BitSet其实就是BitMap,该成员用于记录fd在epoll中的存在性,该位图的更新和epoll内fd的更新是同步的
    private final BitSet registered = new BitSet(); 
//---------------------------------------------------------------------------------------    
    //⭐存放 某channel的感兴趣事件,如将一个ServSocketChannel注册进Selector时,eventsLow[channel.fd]就会存放OP_ACCEPT(8)这个事件
    private final byte[] eventsLow = new byte[MAX_UPDATE_ARRAY_SIZE];
//---------------------------------------------------------------------------------------	    
    // number of file descriptors with registration changes pending
    //记录更新过事件的fd数目,被用作updateDescriptors的下标指针
    private int updateCount;
    // file descriptors with registration changes pending
    //用作存放 改变过事件类型的fd
    private int[] updateDescriptors = new int[INITIAL_PENDING_UPDATE_SIZE];
//--------------------------------------------------------------------------------------- 
    // opcodes
    //用作epoll_ctl方法的第二个参数,指要对fd和events进行的操作如EPOLL_CTL_ADD就是向操作系统注册要监听的事件类型和 监听channel 的文件描述符
    private static final int EPOLL_CTL_ADD      = 1;
    private static final int EPOLL_CTL_DEL      = 2;
    private static final int EPOLL_CTL_MOD      = 3;
//--------------------------------------------------------------------------------------	
    //由 epoll_create 生成的epoll专用的文件描述符
    private final int epfd;
	
//---------------------------------------------------------------------------------------
        
     //⭐JVM底层操作的一个对象,epoll_wait方法会将网络IO完成的fd信息记录在这个对象中,可将该对象看成数组
    private final AllocatedNativeObject pollArray;
     //记录某次epoll_wait返回时IO完成的fd个数
    int updated;
//---------------------------------------------------------------------------------------    
    //该 函数生成一个epoll专用的文件描述符。它其实是在内核申请一空间，用来存放你想关注的socket fd上是否发生以及发生了什么事件。
	private native int epollCreate();
    //该函数用于控制某个epoll文件描述符上的事件，可以注册事件，修改事件，删除事件。
    /*
    参数：
		epfd：由 epoll_create 生成的epoll专用的文件描述符；
		op：要进行的操作例如注册事件，可能的取值EPOLL_CTL_ADD 注册、EPOLL_CTL_MOD 修 改、EPOLL_CTL_DEL 删除
		fd：关联的文件描述符,在这里指channel的fd
		events：感兴趣的事件,如OP_READ等
    */
    private native void epollCtl(int epfd, int opcode, int fd, int events);
    //epoll_ctl可以操作epoll_create创建的epoll，如将socket句柄加入到epoll中让其监控，或把epoll正在监控的某个socket句柄移出epoll。原理留到看epoll时才讲.
    private native int epollWait(long pollAddress, int numfds, long timeout,
                                 int epfd) throws IOException;
    
	 void initInterrupt(int fd0, int fd1) {
        outgoingInterruptFD = fd1;
        incomingInterruptFD = fd0;
        epollCtl(epfd, EPOLL_CTL_ADD, fd0, EPOLLIN);
    }
	   /**
     * Add a file descriptor
     */
    //上面讲到的pollWrapper.add(fd)调用的就是这个方法
    void add(int fd) {
        // force the initial update events to 0 as it may be KILLED by a
        // previous registration.
        synchronized (updateLock) {
            assert !registered.get(fd);
            //强制初始化事件为0,也即OP_READ,理由看上面英文解释,我不是很理解
            //不过在这之后还是会调用setInterest()为其设置的
            setUpdateEvents(fd, (byte)0, true);
        }
    }
	 private void setUpdateEvents(int fd, byte events, boolean force) {
        if (fd < MAX_UPDATE_ARRAY_SIZE) {
            if ((eventsLow[fd] != KILLED) || force) {
                //修改eventsLow数组
                eventsLow[fd] = events;
            }
        } else {
            Integer key = Integer.valueOf(fd);
            if (!isEventsHighKilled(key) || force) {
                eventsHigh.put(key, Byte.valueOf(events));
            }
        }
    }
    /**
     * Remove a file descriptor
     */
    //该方法将会在processDeregisterQueue()中调用,目的是在epoll_wait之前删除已关闭的channel的fd(看到doSelect()就懂了)
    void remove(int fd) {
        synchronized (updateLock) {
            // kill pending and future update for this file descriptor
            //将该channel的fd对应的感兴趣时间置为KILLED
            setUpdateEvents(fd, KILLED, false);

            // remove from epoll
            if (registered.get(fd)) {
                //删除epoll中监听的fd
                epollCtl(epfd, EPOLL_CTL_DEL, fd, 0);
                //同时在Java中记录该fd已不存在
                registered.clear(fd);
            }
        }
    }
    //更新某fd的感兴趣的事件,⭐selectionKey.interestOps()方法最终就会来到这里
    void setInterest(int fd, int mask) {
        synchronized (updateLock) {
            // record the file descriptor and events
            int oldCapacity = updateDescriptors.length;
            if (updateCount == oldCapacity) {
                int newCapacity = oldCapacity + INITIAL_PENDING_UPDATE_SIZE;
                int[] newDescriptors = new int[newCapacity];
                System.arraycopy(updateDescriptors, 0, newDescriptors, 0, oldCapacity);
                updateDescriptors = newDescriptors;
            }
            //将修改过的fd记录进updateDescriptors[]
            updateDescriptors[updateCount++] = fd;

            // events are stored as bytes for efficiency reasons
            byte b = (byte)mask;
            assert (b == mask) && (b != KILLED);
            //同时修改eventsLow[]
            setUpdateEvents(fd, b, false);
        }
    }
   
```

该类除了封装操作系统的epoll调用外，最主要的功能还是对`fd→event`关联的管理.

### 四、`channel.close()`

在看最重点的`select()`之前,先看看channel是怎么关闭的

```java
public final void close() throws IOException {
        synchronized (closeLock) {
            if (!open)
                return;
            open = false;
            implCloseChannel();
        }
    }
//AbstractSelectableChannel.java
protected final void implCloseChannel() throws IOException {
        implCloseSelectableChannel();
        synchronized (keyLock) {
            int count = (keys == null) ? 0 : keys.length;
            //获得上面所说的channel记录的SelectionKey,逐个调用其cancel方法
            for (int i = 0; i < count; i++) {
                SelectionKey k = keys[i];
                if (k != null)
                    k.cancel();
            }
        }
    }
//AbstractSelectionKey.java
   public final void cancel() {
        
        synchronized (this) {
            if (valid) {
                valid = false;
                //又转去调用Selector的cancel
                ((AbstractSelector)selector()).cancel(this);
            }
        }
    }
//AbstractSelector.java
    void cancel(SelectionKey k) {                       
        synchronized (cancelledKeys) {
            //最终就是在将要删除的SelectionKey记录到cancelledKeys成员中
            cancelledKeys.add(k);
        }
    }
```

可以看到,`channel.close()`并没有对`epoll`对象操作,而是使用了一个`cancelledKeys`成员记录下这个`SelectionKey`。其实际的对`epoll`对象的rbr更新是在`select()`中。

### 五、`selector.select()`

```java
//SelectorImpl.java
public int select() throws IOException {
        return select(0);
    }
public int select(long timeout)
        throws IOException
    {
        if (timeout < 0)
            throw new IllegalArgumentException("Negative timeout");
        return lockAndDoSelect((timeout == 0) ? -1 : timeout);
    }
  private int lockAndDoSelect(long timeout) throws IOException {
        synchronized (this) {
            if (!isOpen())
                throw new ClosedSelectorException();
            synchronized (publicKeys) {
                synchronized (publicSelectedKeys) {
                    return doSelect(timeout);
                }
            }
        }
    }
```

`select()`的核心`doSelect()`

```java
//EpollSelectorImpl.java 
protected int doSelect(long timeout) throws IOException {
        if (closed)
            throw new ClosedSelectorException();
     //注意看这个方法,该方法中的implDereg(ski)会将已取消的SelectionKeyImpl,在pollWrapper.remove(fd)中真正取消一次(只是将该fd置为EPOLL_CTL_DEL),然后在doSelect()的epoll()中的updateRegistrations()方法将这些新增取消操作调用epoll_ctl更新一遍,也就是添加channel的会添加,取消channel的会删除fd
    	//先更新一下epoll,在此之前调用了channel.cancel()的channel将会在这个方法中 把此改动同步到epoll中
        processDeregisterQueue();
        try {
            begin();
            //调用epoll_wait(),监听注册进epoll的fd
            pollWrapper.poll(timeout);
        } finally {
            end();
        }
    //再删除一遍取消的channel
        processDeregisterQueue();
    	//更新SelectedKeys.将fd(channel)对应的SelectionKey放入成员变量selectedKeys(也是publicSelectedKeys)中
        int numKeysUpdated = updateSelectedKeys();
    	//跳过
        if (pollWrapper.interrupted()) {
            // Clear the wakeup pipe
            pollWrapper.putEventOps(pollWrapper.interruptedIndex(), 0);
            synchronized (interruptLock) {
                pollWrapper.clearInterrupted();
                IOUtil.drain(fd0);
                interruptTriggered = false;
            }
        }
        return numKeysUpdated;
    }
```

#### ①、`processDeregisterQueue()`

```java
//SelectorImpl.java
void processDeregisterQueue() throws IOException {
        //获得已取消的SelectionKey,如上面所说的调用socketChannel.cancel()时,就会将取消的SelectionKey放入AbstractSelector的cancelledKeys成员中
        Set<SelectionKey> cks = cancelledKeys();
        →{
            return cancelledKeys;
        }
        synchronized (cks) {
            if (!cks.isEmpty()) {
                Iterator<SelectionKey> i = cks.iterator();
                while (i.hasNext()) {
                    SelectionKeyImpl ski = (SelectionKeyImpl)i.next();
                    try {
                        //在这里与EPollArrayWrapper打交道
                        implDereg(ski);
                    } catch (SocketException se) {
                        throw new IOException("Error deregistering key", se);
                    } finally {
                        //返回的直接是成员变量引用,这里删除之后AbstractSelector的cancelledKeys成员也就删除了
                        i.remove();
                    }
                }
            }
        }
    }
//SelectorImpl.java
protected void implDereg(SelectionKeyImpl ski) throws IOException {
        assert (ski.getIndex() >= 0);
        SelChImpl ch = ski.channel;
        int fd = ch.getFDVal();
    //删除该fd→SelectionKey映射
        fdToKey.remove(Integer.valueOf(fd));
    //⭐调用epollCtl(EPOLL_CTL_DEL)将该fd(channel)从epoll中删除了
        pollWrapper.remove(fd);
        ski.setIndex(-1);
        keys.remove(ski);
    //从已IO完成的set中删除该selectionKey
        selectedKeys.remove(ski);
        deregister((AbstractSelectionKey)ski);
        SelectableChannel selch = ski.channel();
        if (!selch.isOpen() && !selch.isRegistered())
            ((SelChImpl)selch).kill();
    }
```

#### ②、`pollWrapper.poll()`

又回到了`EPollArrayWrapper`类

```java
//EPollArrayWrapper.java
int poll(long timeout) throws IOException {
        updateRegistrations();
    //监听注册进epoll的fd,返回IO完成的fd个数,并赋值到updated成员中
        updated = epollWait(pollArrayAddress, NUM_EPOLLEVENTS, timeout, epfd);
        for (int i=0; i<updated; i++) {
            if (getDescriptor(i) == incomingInterruptFD) {
                interruptedIndex = i;
                interrupted = true;
                break;
            }
        }
        return updated;
    }
//更新新注册的channel，上面的方法是向epoll删除已取消的channel,而这里就是向epoll更新新注册的channel
private void updateRegistrations() {
        synchronized (updateLock) {
            int j = 0;
            //向epoll更新所有在updateDescriptors的fd
            while (j < updateCount) {
                //将改变过 事件类型的fd取出
                int fd = updateDescriptors[j];
                //从eventsLow[]成员中取出 改变过 事件类型的fd对应的事件类型
                short events = getUpdateEvents(fd);
                boolean isRegistered = registered.get(fd);
                int opcode = 0;

                if (events != KILLED) {
                    //根据registered的存在性判断决定是向epoll中添加fd还是修改fd的events
                    //在介绍EPollArrayWrapper时曾说过,这个registered会与epoll中的fd存在性保持同步,用处就是在这里.如果epoll中已存在fd,则修改其events;而如果不存在,就添加fd→events以保持fd在epoll中的唯一性
                    if (isRegistered) {
                        //⭐注意看这里,要更新的events=0就意味着想从epoll中删除该fd,也即如果要取消在epoll中的clientSocket除了调用clientSocket.cancel()还可以调用selectionkey.interestOps(0);
                        opcode = (events != 0) ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
                    } else {
                        opcode = (events != 0) ? EPOLL_CTL_ADD : 0;
                    }
                    if (opcode != 0) {
                        //向epoll注册该channel
                        epollCtl(epfd, opcode, fd, events);
                        //同步registered成员
                        if (opcode == EPOLL_CTL_ADD) {
                            registered.set(fd);
                        } else if (opcode == EPOLL_CTL_DEL) {
                            registered.clear(fd);
                        }
                    }
                }
                j++;
            }
            updateCount = 0;
        }
    }
```

#### ③、`updateSelectedKeys()`

方法来到这里,就说明epoll已经监听到有channel的网络IO已完成,此时就开始进行`selectedKeys`的更新了

```java
//EpollSelectorImpl.java
private int updateSelectedKeys() {
    	//获取本次epoll返回时IO完成的fd个数
        int entries = pollWrapper.updated;
        int numKeysUpdated = 0;
        for (int i=0; i<entries; i++) {
            //⭐这个方法正是从EpollArrayWrapper的pollArray成员中获取IO完成的fd
            int nextFD = pollWrapper.getDescriptor(i);
            {
                    int offset = SIZE_EPOLLEVENT * i + FD_OFFSET;
        			return pollArray.getInt(offset);
            }
            //从fdToKey映射中得到SelectionKey
            SelectionKeyImpl ski = fdToKey.get(Integer.valueOf(nextFD));
            // ski is null in the case of an interrupt
            if (ski != null) {
                
                //曾在EPollArrayWrapper中提到过,OP_ACCEPT会经过一次转换才会变成真正放入epoll_ctl的events参数(其实OP_READ、OP_WRITE也会经过转换)
            	//这里从pollArray成员中获取的就是注册进epoll的events值
                int rOps = pollWrapper.getEventOps(i);
                //如果selectedKeys中已经存在这个SelectionKey,则说明是代码出现了读取完socketChannel的数据之后没有调用其close()导致的,调用channel.close(),就会在processDeregisterQueue()中将该selectionKey从selectedKeys中移除
                if (selectedKeys.contains(ski)) {
                    if (ski.channel.translateAndSetReadyOps(rOps, ski)) {
                        numKeysUpdated++;
                    }
                } else {
                    //转换epoll的events到channel定义的events
                    ski.channel.translateAndSetReadyOps(rOps, ski);
                    //个人认为这个判断语句和sk.nioReadyOps()==sk.nioInterestOps()等价
                    if ((ski.nioReadyOps() & ski.nioInterestOps()) != 0) {
                        //⭐往成员selectedKeys中添加SelectionKey,在这之后调用selector.selectedKeys()就会返回这个成员
                        selectedKeys.add(ski);
                        numKeysUpdated++;
                    }
                }
            }
        }
        return numKeysUpdated;
    }
```

```java
//该接口方法定义在SelChImpl.java中,有多种不同的实现,且最关心的ServerSocketChannel和SocketChannel两者的实现不同
public boolean translateAndSetReadyOps(int ops, SelectionKeyImpl sk);
//-------------------------------ServerSocketChannel实现----------------------------------
    
public boolean translateAndSetReadyOps(int ops, SelectionKeyImpl sk) {
        return translateReadyOps(ops, 0, sk);
    }
 public boolean translateReadyOps(int ops, int initialOps,
                                     SelectionKeyImpl sk) {
     	//epoll层面的感兴趣事件ops是1
     	//取出Selectionkey层面该channel感兴趣的事件,一般是16
        int intOps = sk.nioInterestOps(); // Do this just once, it synchronizes
        int oldOps = sk.nioReadyOps();
     	//0
        int newOps = initialOps;

        if ((ops & Net.POLLNVAL) != 0) {
            // This should only happen if this channel is pre-closed while a
            // selection operation is in progress
            // ## Throw an error if this channel has not been pre-closed
            return false;
        }

        if ((ops & (Net.POLLERR | Net.POLLHUP)) != 0) {
            newOps = intOps;
            sk.nioReadyOps(newOps);
            return (newOps & ~oldOps) != 0;
        }

        if (((ops & Net.POLLIN) != 0) &&
            ((intOps & SelectionKey.OP_ACCEPT) != 0))
            //一般都会进来这里,而newOps将会等于SelectionKey.OP_ACCEPT
                newOps |= SelectionKey.OP_ACCEPT;

        sk.nioReadyOps(newOps);
     →{
         //设置SelectionKey的readyOps为SelectionKey.OP_ACCEPT
         readyOps = ops;
     }
        return (newOps & ~oldOps) != 0;
    }
//------------------------------------SocketChannel实现-----------------------------------
//和ServerSocketChannel做的工作差不多,都是转换工作,只不过这里的ops一般会等于intOps
public boolean translateReadyOps(int ops, int initialOps,
                                     SelectionKeyImpl sk) {
        int intOps = sk.nioInterestOps(); // Do this just once, it synchronizes
        int oldOps = sk.nioReadyOps();
        int newOps = initialOps;

        if ((ops & Net.POLLNVAL) != 0) {
            // This should only happen if this channel is pre-closed while a
            // selection operation is in progress
            // ## Throw an error if this channel has not been pre-closed
            return false;
        }

        if ((ops & (Net.POLLERR | Net.POLLHUP)) != 0) {
            newOps = intOps;
            sk.nioReadyOps(newOps);
            // No need to poll again in checkConnect,
            // the error will be detected there
            readyToConnect = true;
            return (newOps & ~oldOps) != 0;
        }

        if (((ops & Net.POLLIN) != 0) &&
            ((intOps & SelectionKey.OP_READ) != 0) &&
            (state == ST_CONNECTED))
            newOps |= SelectionKey.OP_READ;

        if (((ops & Net.POLLCONN) != 0) &&
            ((intOps & SelectionKey.OP_CONNECT) != 0) &&
            ((state == ST_UNCONNECTED) || (state == ST_PENDING))) {
            newOps |= SelectionKey.OP_CONNECT;
            readyToConnect = true;
        }

        if (((ops & Net.POLLOUT) != 0) &&
            ((intOps & SelectionKey.OP_WRITE) != 0) &&
            (state == ST_CONNECTED))
            newOps |= SelectionKey.OP_WRITE;

        sk.nioReadyOps(newOps);
        return (newOps & ~oldOps) != 0;
    }
```

`select`小结：

1. 调用`processDeregisterQueue()`,将调用了`channel.close()`的channel所属fd从`epoll`中删除
2. 调用`updateRegistrations()`,将调用了`channel.register(selector,OP_···)`和`selecitionKey.interestOps(i)`从而对channel感兴趣事件做了改动的操作真正应用到(调用`epoll_ctl`)epoll对象中(要注意的一个点是调用`selectionkey.interestOps(0)`会将该channel从epoll对象中删除)
3. `updateSelectedKeys()`将从JVM操作的`AllocatedNativeObject`对象中获取已IO完成的fd,将这些fd对应的`selectionKey`放入`selectedKey`中, 

### 五、`server.configureBlocking()`

其调用栈如下:

```java
//ServerSocketChannelImpl.java
protected void implConfigureBlocking(boolean block) throws IOException {
        IOUtil.configureBlocking(fd, block);
    }
public static native void configureBlocking(FileDescriptor fd,
                                                boolean blocking)
```

由此得知,`configureBlocking()`设置的是`ServerSocket`的阻塞状态。若设置了`server.configureBlocking(true)`,之后调用`serverSocket.accept()`就会阻塞着等待连接,也就是不将该`eventpoll`放入该`Socket`对象的等待队列中,而是用最传统的BIO方式将进程引用放入该`Socket`对象的等待队列中。