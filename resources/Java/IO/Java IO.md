# Java IO

IO这一块涉及非常多操作系统知识，而我既想讲操作系统，也想讲JavaIO框架底层原理，所以将这两样东西混杂串起来讲就比较容易能理解，所以读者看这篇博客会发现很多张貌似一样的图，但实际上我是想让读者分清楚JavaIO和操作系统内核的区别和联系

## ==⭐名词解释==

* **==阻塞和非阻塞==是宏观上（进程间通讯，==通常表现为网络IO的处理上==）的，具体来说就是 ：进程/线程要访问的数据是否就绪，进程/线程是否需要等待**

* **==同步/异步==是微观上（进程内的数据传输，==通常表现为对本地IO的处理上==）的，具体来说就是：访问数据的方式，==同步需要主动读写数据，在*读写数据*的过程中还是会阻塞==；异步只需要I/O操作完成的通知，==*并不主动读写数据*==，由操作系统内核完成数据的读写。**

> 上面两个解释其实就是依据Linux五种IO模型得出的，记住IO模型的名字和区别其实就是记住了上面四种的区别

**==请注意，同步的定义是轮询去查看IO操作是否就绪，这是用户进程去查看的而不是内核查看，需要分辨清非阻塞IO和多路复用IO的区别==**

下面给出五种IO模型的大概解释：

<img src="E:\Typora\resources\Java\IO\五种IO模型大概解释.gif" style="zoom:100%;" />

**==⭐可以看到，非阻塞IO*需要用户进程不断check*；而在多路复用IO中用户进程只调用了一次check就一直阻塞，等到*网络IO完成时操作系统才会返回*==**

## 五种IO模型

### 观看建议

Java的NIO、AIO的主要功能其实都是依赖于操作系统内核的一些系统调用的，所以学IO其实更多的是要学操作系统怎么处理这些IO。

### 一、BIO

#### 网络模型

<img src="E:\Typora\resources\Java\IO\BIO网络模型.png" style="zoom:50%;" />

#### 内核模型

<img src="E:\Typora\resources\Java\IO\LinuxBIO模型.png" style="zoom:50%;" />

BIO即同步阻塞IO，==由**一条线程**来监听客户端的连接建立请求,当连接建立后,服务器端需要创建一个新线程(线程池)处理客户端发送过来的数据,且客户端数据的读取或写入都必须阻塞在这个线程中==，即**⭐服务端需要在这个线程中等待数据从客户端传送到服务器端**，才能开始读操作。（连接的建立只需要调用`socket.connect()`,此时服务器就会创建新线程,而数据真正发送则需要在客户端调用`socket.write()`中，此时服务器调用的`input.readLine()`就会发生阻塞）

缺点:

* 每当有新连接就创建线程,当连接数过多时,不断创建新线程会导致OOM;就算是使用线程池,等待队列也有一个界限,到达这个界限之后就会执行拒绝策略丢弃请求==(最主要关注的点还是有新连接就创建新线程,使得在新线程内有大概率大部分时间都在等待网络IO)==
* 当网络拥塞或客户端迟迟不发送数据,新创建的线程就会一直阻塞在`socket.read()`方法中,白白浪费大量资源

### 二、NIO

#### 网络模型（主要记Reactor模型）

![](E:\Typora\resources\Java\IO\NIO网络模型.png)



#### 混杂Java的内核模型

<img src="E:\Typora\resources\Java\IO\NIO内核模型.jpg" style="zoom:50%;" />

**Java NIO是多路复用IO==（并非是同步非阻塞IO）==，对于客户端的 连接建立 请求，代码逻辑(即为图中的AcceptorHandler)将会将此客户端对应的`SocketChannel`对象注册进Selector中,Selector将会将此`Socket`注册到内核中。当调用`selector.select()`时,Selector底层将会调用内核的`epoll_wait`,此时操作系统内核将不断轮询这些`socket`,一旦有至少一个`Socket`的数据传输到服务端时,`epoll_wait`将会返回,使得业务代码可以开启新线程处理客户端发来的请求.==*⭐⭐⭐⭐⭐⭐然而请注意，`epoll_wait`的返回只是因为数据传输到了服务端内核空间，若Java进程要读取这些数据，还需要将数据从内核空间复制到用户空间才能直接对数据进行处理，即`socketChannel.read(buffer)`其实是需要阻塞等待 数据从内核空间复制到用户空间,之后才能将数据读入`ByteBuffer`中,这是与AIO的一个最重要的区别*==**

**⭐对比BIO的优点**:

* 可以看到,在客户端与服务端建立连接时并没有创建新线程,而是使用多路复用器**这一条线程**轮询这些 连接套接字,等到真正有数据传送到服务端时**才开启新线程**处理请求.**==这样就使得*新开的线程*不需要阻塞等待网络IO,直接就可以处理数据.==**
#### Ⅰ、三大组件
在Java中组成NIO的有三大组件：Buffer、Channel、Selector。

##### 1、Buffer

<img src="E:\Typora\resources\Java\IO\Buffer类结构.png" style="zoom:50%;" />

Buffer类是读取传输数据的唯一途径,此类相当于一个数组，只不过 写和读取的方式有点复杂。

下面来介绍`ByteBuffer`的属性与使用方法

###### ①、属性

`position`:在写模式下,代表下一个写入字节的下标;在读模式中,代表下一个读取的字节的下标

`limit`:在写模式下, 代表的是最多能写入数据的下标，这个时候 limit 等于 capacity(相当于array.length);在读模式下,代表实际数据的最大下标(相当于`ArrayList.size`)

`capacity`:相当于`array.length`,但这个length是用`ByteBuffer.allocate()`分配大小的

<img src="E:\Typora\resources\Java\IO\Buffer数据结构.png" style="zoom:50%;" />

###### ②、使用方法

```java
//ByteBuffer.java
//⭐初始化一个ByteBuffer对象,参数为容量
public static ByteBuffer allocate(int capacity)
//⭐将byte数组转化为ByteBuffer
public static ByteBuffer wrap(byte[] array)  
//从写模式转换为读模式,⭐注意，通常在说 NIO 的读操作的时候，我们说的是从 Channel 中读数据到 Buffer 中，对应的是对 Buffer 的写入操作,之后要读这个buffer就要换成读模式。
public final Buffer flip()
    
    
//记录当前position下标
public final Buffer mark()
//使position重新回到mark()的位置
public final Buffer reset() 
//会重置 position 为 0，通常用于重新从头读写 Buffer
public final Buffer rewind()
//重置 Buffer 的意思，相当于重新实例化了一样   
public final Buffer clear()
```

##### 2、Selector

Selector是多路复用器，由他来轮询`Channel`并返回网络IO完毕的`Channel`,之后的代码逻辑就能通过`Channel`的事件类型来进行不同的逻辑处理,如 用作建立连接的`ServerSocketChannel`和用作数据传输的`SocketChannel`(这块逻辑在DEMO中具体体现)

###### ①、重点方法

```java
//实例化Selector
Selector selector = Selector.open();
//⭐重点:轮询注册进Selector的channel,当发现至少有一个的网络IO完毕时则停止阻塞,返回网络IO完毕的个数
//要注意,这个方法不是一次性的,即select()发现channel有数据并对其进行读取后,第二次再次执行这个方法还是会返回读取过的channel,两次调用唯一的不同点是第二次调用 socketChannel.read() 时,由于已经读过一次,通道已经到达了末尾,此时socketChannel.read()将会返回-1(如果在第二次调用read()前又有新增数据可能就不会返回-1了,看jdk文档),此时如果不想它继续在Selector中就只能够断开此连接
public int select()
//⭐当调用select()返回后,通过此方法来获得IO完毕channel的集合 
public Set<SelectionKey> selectedKeys()    
//唤醒阻塞在select()中的selector
public Selector wakeup()    
```

##### 3、Channel

所有的NIO操作始于Channel，代码逻辑通过往Channel写数据和读数据以完成数据在网络上传输的功能。它与前面介绍的 Buffer 打交道，读操作的时候将 Channel 中的数据填充到 Buffer 中，而写操作时将 Buffer 中的数据写入到 Channel 中。

<img src="E:\Typora\resources\Java\IO\Channel类结构.png" style="zoom:50%;" />

主要看基于TCP的`ServerSocketChannel`和`SocketChannel`.

`ServerSocketChannel`是服务端channel,专门用作监听连接请求;而`SocketChannel`则专门用作客户端与服务器建立连接 和 负责客户端服务器端两者数据的传输

###### ①、重点方法

```java
//ServerSocketChannel.java

//⭐⭐往selector中注册channel,第二个参数为这个channel的类型,用于在代码中判断IO完毕的channel类型以执行不同的逻辑
/**
SelectionKey.OP_READ
对应 00000001，通道中有数据可以进行读取

SelectionKey.OP_WRITE
对应 00000100，可以往通道中写入数据

SelectionKey.OP_CONNECT
对应 00001000，成功建立 TCP 连接

SelectionKey.OP_ACCEPT
对应 00010000，接受 TCP 连接
*/
//规定ServerSocketChannel只能是SelectionKey.OP_ACCEPT类型,而SocketChannel只能是其他三种类型
public final SelectionKey register(Selector sel, int ops)

//⭐这个方法比上面多了一个att参数,这个参数就是attachment.若在register()将该对象传了进Selector,那么
public abstract SelectionKey register(Selector sel, int ops, Object att);

//⭐⭐返回需要建立连接的客户端SocketChannel，连接建立事件的专用方法
public SocketChannel accept()
    
```

```java
//SocketChannel.java

//⭐读取channel里的数据进ByteBuffer中，channel内的数据只供读一遍，若没有数据添加且再掉用此方法第二次则会返回-1
public int read(ByteBuffer var1)
//向特定主机端口建立连接   
public boolean connect(SocketAddress var1)
    
//向对方传输数据时则调用此方法写入数据
public int write(ByteBuffer var1)
```

##### 4、`SelectionKey`

`SelectionKey`是`Selector`和`channel`的桥梁.

```java
//SelectionKey.java

    public static final int OP_READ = 1 << 0;
    public static final int OP_WRITE = 1 << 2;
    public static final int OP_CONNECT = 1 << 3;
    public static final int OP_ACCEPT = 1 << 4;

	//返回此选择键所关联的通道.即使此key已经被取消,仍然会返回.
	public abstract SelectableChannel channel();
	//返回此选择键所关联的选择器,即使此键已经被取消,仍然会返回.
	public abstract Selector selector()
	//请求将此键取消注册.一旦返回成功,那么该键就是无效的(⭐我理解的是在调用select()之前取消,那么这个channel就不会被注册到操作系统中,也就不会被轮询),被添加到selector的cancelledKeys中.cancel操作将key的valid属性置为false,并执行selector.cancel(key)(即将key加入cancelledkey集合)
    public abstract void cancel()
	
    //将
    public abstract SelectionKey interestOps(int ops);

	public abstract int readyOps();

	//向
	 public final Object attach(Object ob) {
        return attachmentUpdater.getAndSet(this, ob);
    }

    /**
     * Retrieves the current attachment.
     *
     * @return  The object currently attached to this key,
     *          or <tt>null</tt> if there is no attachment
     */
    public final Object attachment() {
        return attachment;
    }
```

从以上的方法中就能看出Java NIO的架构,**无论是ServerSocketChannel或SocketChannel都要注册进Selector中,只不过这两者负责的事件类型不同:前者负责建立连接,且在建立连接后要将客户端的SocketChannel注册进Selector;而后者负责传输数据,所以代码中对不同的事件类型要有不同的处理逻辑**

#### Ⅱ、Demo

```JAVA
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

```java
public class NIOClient
{
    public static void main(String[] args) throws IOException
    {
        SocketChannel socketChannel = SocketChannel.open();
        try
        {
            socketChannel.connect(new InetSocketAddress(8080));

            socketChannel.write(ByteBuffer.wrap("link start!".getBytes()));

            ByteBuffer out = ByteBuffer.allocate(1024);
            socketChannel.read(out);
            System.out.println(out.position());
            System.out.println(out.toString());
        }finally
        {
            socketChannel.close();
        }
    }
}
```

### 三、AIO

#### 混杂Java的内核模型（主要记Proactor模型）

![](E:\Typora\resources\Java\IO\AIO流程.jpg)

AIO即**异步非阻塞IO**，服务器端只需要将要监听的`AsynchronousServerSocketChannel`和`CompletionHanlder`注册进AIO框架，==其中`server.accept()`方法会向操作系统操作系统做`io_read`系统调用。==当有客户端将数据传送到该Channel时,操作系统内核就会将这些数据复制到用户空间,然后**通知**Java进程。**(这个通知的含义貌似是将 通知 这个任务放进`AsynchronousChannelGroups`配置的线程池中,然后通知任务内部又会将 `CompletionHandler.complete()`方法的调用 作为一个任务放入线程池。这里我讲不清楚通知是怎么被放到线程池中的，因为这源码写出来就tm没打算过给人看)**AIO框架就会将 回调逻辑的执行 作为任务提交给`AsynchronousChannelGroups`配置的线程池执行。

#### Ⅰ、Demo

```java
public class AIOServer
{
    public static void main(String[] args) throws IOException {

        // 实例化，并监听端口
        AsynchronousServerSocketChannel server =
                AsynchronousServerSocketChannel.open().bind(new InetSocketAddress(8080));

        // 自己定义一个 Attachment 类，用于传递一些信息
        Attachment att = new Attachment();
        att.setServer(server);
		//进行io_read系统调用，可以看看Linux的异步IO模型
        server.accept(att, new CompletionHandler<AsynchronousSocketChannel, Attachment>() {
            //Java进程收到操作系统IO完成的通知后，将用AsynchronousChannelGroups线程池的线程执行该方法
            @Override
            public void completed(AsynchronousSocketChannel client, Attachment att) {
                try {
                    SocketAddress clientAddr = client.getRemoteAddress();
                    System.out.println("收到新的连接：" + clientAddr);

                    // 收到新的连接后，server 应该重新调用 accept 方法等待新的连接进来
                    att.getServer().accept(att, this);

                    Attachment newAtt = new Attachment();
                    newAtt.setServer(server);
                    newAtt.setClient(client);
                    newAtt.setReadMode(true);
                    newAtt.setBuffer(ByteBuffer.allocate(2048));

                    // 这里也可以继续使用匿名实现类，不过代码不好看，所以这里专门定义一个类
                    client.read(newAtt.getBuffer(), newAtt, new ChannelHandler());
                } catch (IOException ex) {
                    ex.printStackTrace();
                }
            }

            @Override
            public void failed(Throwable t, Attachment att) {
                System.out.println("accept failed");
            }
        });
        // 为了防止 main 线程退出
        try {
            Thread.currentThread().join();
        } catch (InterruptedException e) {
        }
    }
}
class ChannelHandler implements CompletionHandler<Integer, Attachment> {

    @Override
    public void completed(Integer result, Attachment att) {
        if (att.isReadMode()) {
            // 读取来自客户端的数据
            ByteBuffer buffer = att.getBuffer();
            buffer.flip();
            byte bytes[] = new byte[buffer.limit()];
            buffer.get(bytes);
            String msg = new String(buffer.array()).toString().trim();
            System.out.println("收到来自客户端的数据: " + msg);

            // 响应客户端请求，返回数据
            buffer.clear();
            buffer.put("Response from server!".getBytes(Charset.forName("UTF-8")));
            att.setReadMode(false);
            buffer.flip();
            // 写数据到客户端也是异步
            att.getClient().write(buffer, att, this);
        } else {
            // 到这里，说明往客户端写数据也结束了，有以下两种选择:
            // 1. 继续等待客户端发送新的数据过来
//            att.setReadMode(true);
//            att.getBuffer().clear();
//            att.getClient().read(att.getBuffer(), att, this);
            // 2. 既然服务端已经返回数据给客户端，断开这次的连接
            try {
                att.getClient().close();
            } catch (IOException e) {
            }
        }
    }

    @Override
    public void failed(Throwable t, Attachment att) {
        System.out.println("连接断开");
    }
}
@Getter
@Setter
class Attachment {
    private AsynchronousServerSocketChannel server;
    private AsynchronousSocketChannel client;
    private boolean isReadMode;
    private ByteBuffer buffer;
}
```

#### ==Ⅱ、AIO和NIO的区别==

* ①AIO回调基于通知，回调是AIO框架内部自动调用；而NIO的数据处理逻辑 则需要用户代码手动调用
* <img src="E:\Typora\resources\Java\IO\LinuxNIO.png" style="zoom:33%;" /><img src="E:\Typora\resources\Java\IO\LinuxAIO.png" style="zoom:33%;" />②**NIO中,只要网络IO完成,内核就通知Java进程数据可读，`selector.select()`此时也就不再阻塞。当用户要读取数据时(`socketChannel.read(buffer)`)，NIO框架内部阻塞进行系统调用使得数据从内核空间复制到用户空间，并将数据读入`ByteBuffer`供用户代码使用。而在AIO中，网络IO完成后，==*内核还负责将数据从内核空间复制到了用户空间，这一步完成之后内核再通知Java进程数据可读,这就使得AIO比NIO少了一次系统调用阻塞.上面两个LinuxIO模型就很清楚的说明了这个区别*==**

### 四、非阻塞IO（无法定义同步还是异步，因为它两不像）

**一定要分清楚同步非阻塞IO并不是多路复用IO！！**

<img src="E:\Typora\resources\Java\IO\同步非阻塞IO.png" style="zoom:50%;" />

用户进程不断询问内核是否有数据已准备好，但是在询问与询问之间可以执行其他代码

### 五、信号驱动式IO（异步阻塞IO，这个名字只是为了方便自己记，这样定义是错误的）

原理比较简单，**这几种的内核模型主要有两个阶段：网络IO和本地内核进程与用户进程的IO**，分清楚了这个两个阶段就很好记了。信号驱动式IO阻塞的原因是：读取数据时会产生系统调用，使得数据从内核拷贝到用户空间，这个时间段会产生阻塞。

<img src="E:\Typora\resources\Java\IO\Linux异步阻塞IO.png" style="zoom:50%;" />

知识点:五种LinuxIO模型,五种模型之间的区别,Java AIO、Java NIO的用法流程，还有Reactor（多路复用IO）和Procator（异步IO）的抽象流程，名词解释

> 参考资料
>
> https://blog.csdn.net/mulinsen77/article/details/89040158
>
> # https://www.cnblogs.com/fatmanhappycode/p/12399074.html
>
> https://www.javadoop.com/post/nio-and-aio
>
> https://www.liangzl.com/get-article-detail-4042.html
>
> https://www.imooc.com/video/19328
>
> https://blog.csdn.net/historyasamirror/article/details/5778378