# `select`、`poll`和`epoll`的原理

注意点：

1. BIO是`recv()`时进程加入到sock对象等待队列中,而NIO是`select()`时进程就加入到了sock对象等待队列.

2. 要清楚sock对象代表一个文件,也就是fd,清楚他和进程的关系就记得程序的代码是怎么写的了
3. select耦合的原理可以这样记:看epoll_wait和epoll_ctl和epoll_create的作用和解决了什么就行了

###  一、BIO原理

我们先从只监听**一个socket**开始讲起：

假设有一个程序编写着下面代码:

```java
//创建socket
int s = socket(AF_INET, SOCK_STREAM, 0);   
//绑定端口
bind(s, ...)
//监听
listen(s, ...)
//接受客户端连接,阻塞直到有客户端连接
int c = accept(s, ...)
//接收客户端数据，没有数据就先阻塞在这里
recv(c, ...);
//将数据打印出来
printf(...)

```

#### Ⅰ、内核的`recv()`原理

1. 首先,CPU开始执行该程序的代码,该程序所属进程会被放到一个工作队列的头结点表示该进程目前是运行态

   <img src="E:\Typora\resources\Java\IO\recv原理①.jpg" style="zoom:Infinity%;" />

2. ==当进程 A 执行到创建 socket 的语句时==，操作系统会**在内核**创建一个由文件系统管理的 socket 对象。这个 socket 对象包含了发送缓冲区、接收缓冲区（这就是内核空间的缓冲区）与等待队列等成员。**等待队列是个非常重要的结构，它指向所有需要等待该 socket 事件的进程。**

   <img src="E:\Typora\resources\Java\IO\recv原理②.jpg" style="zoom:Infinity%;" />



3. 当程序执行到 `recv()` 时，操作系统会将进程 A 从工作队列移动到该 socket 的等待队列中,此时进程A将阻塞。

   <img src="E:\Typora\resources\Java\IO\recv原理③.jpg" style="zoom:Infinity%;" />

4. ⭐当客户端将数据发送到服务器端时,网卡这个IO设备将会直接将数据写到指定地址的内存中(直接存储器访问方式),然后对CPU产生中断。CPU检测出中断后根据中断信号类型转去执行相应的中断处理程序。而这个中断处理程序则会将网卡传输的数据==移动到监听 数据指定端口的 内核Socket对象输入缓冲区==中。之后则唤醒等待队列中的进程。

   <img src="E:\Typora\resources\Java\IO\recv原理④.jpg" style="zoom:Infinity%;" />



5. 进程被唤醒时则会重新加入到工作队列中，==并且中断处理程序会从Socket对象的等待队列中移除该进程。==之后用户进程就可以将数据从内核sock对象缓冲区移动到用户空间进行读取了。

以上是内核接收数据全过程，这里我们可能会思考两个问题：

- 其一，操作系统如何知道网络数据对应于哪个 socket？
- 其二，**如何同时监视多个 socket 的数据？**

第一个问题：因为一个 socket 对应着一个端口号，而网络数据包中包含了 ip 和端口的信息，内核可以通过端口号找到对应的 socket。当然，为了提高处理速度，操作系统会维护端口号到 socket 的索引结构，以快速读取。

第二个问题是多路复用的重中之重，也正是本文后半部分的重点。

### 二、NIO原理——`select`

`select`监听多个socket的思想:**调用`select`时向内核传入一个监听的 socket 列表fds,将当前进程加入这些socket的等待队列，直到有一个 socket 收到数据，则唤醒进程。**

`select`的用法如下:

```java
int s = socket(AF_INET, SOCK_STREAM, 0);  
bind(s, ...);
listen(s, ...);
int fds[] =  //存放需要监听的socket的fd;
while(1){
    //⭐阻塞时,将当前进程加入这些socket的等待队列
    int n = select(..., fds, ...)
     //⭐中断处理程序将该进程从这些socket的等待队列中移除,并唤醒
     //⭐select返回时,需要遍历fds查看是哪个socket对象有数据
    for(int i=0; i < fds.count; i++){
        if(FD_ISSET(fds[i], ...)){
            //fds[i]的数据处理
        }
    }}
```

#### Ⅰ、内核的`select`原理

1. 当进程 A 执行到创建 socket 的语句时，操作系统会**在内核**创建一个由文件系统管理的 socket 对象。在`select`的代码实现中一般就会创建多个sock。

2. ==当进程A执行到`select()`时,方法将fds传进内核,操作系统会将进程的引用加入进这些fds对应sock对象的等待队列中==

   <img src="E:\Typora\resources\Java\IO\select①.jpg" style="zoom:Infinity%;" />

3. 当网卡接收到数据并放入内存的指定位置后,向CPU发出中断信号。此时CPU执行中断处理程序，中断处理程序将数据移动到相应sock的输入缓冲区中，并且唤醒进程A

   <img src="E:\Typora\resources\Java\IO\select②.jpg" style="zoom:Infinity%;" />

4. ==唤醒进程时，会将进程从所有的等待队列中移除，加入到工作队列里面==

   <img src="E:\Typora\resources\Java\IO\select③.jpg" style="zoom:Infinity%;" />

5. ==进程A被唤醒后,它知道至少有一个 socket 接收了数据。此时就需要进程A遍历调用`FD_ISSET(fd)`检测是哪个sock有数据,并处理这些数据==

简单的方法往往有缺点，主要是：

1.  **每次调用 select 都需要将进程加入到所有监视 socket 的等待队列，每次唤醒都需要从每个队列中移除。这里涉及了两次遍历，而且每次都要将整个 fds 列表传递给内核，有一定的开销。正是因为遍历操作开销大，出于效率的考量，才会规定 select 的最大监视数量，默认只能监视 1024 个 socket。**

2. **进程被唤醒后，程序并不知道哪些 socket 收到数据，还需要遍历一次。**

这个两个问题在epoll中得以解决

### 三、NIO原理——`poll`

`poll`相对于`select`并没有做多少改动,只不过是取消了每个进程的`select` 的最大监视数量,并没有解决上述两个问题

### 四、NIO原理——`epoll`

**select 低效的原因之一是将“维护等待队列”和“阻塞进程”两个步骤合二为一**,这就导致每次调用 select**强制**将进程放入fds指定的sock对象等待队列,而每次被唤醒都要将进程从所有sock对象的等待队列中移除。⭐由于两者的耦合，=="阻塞进程"这一操作使得“维护等待队列”操作产生了许多不必要的开销。==

然而大多数应用场景中，需要监视的 socket 相对固定，并不需要每次都修改。**epoll 将这两个操作分开，先用 epoll_ctl 维护等待队列，再调用 epoll_wait 阻塞进程。这就让用户可以选择 在不需要 增加或删除监听的sock情况下,==直接调用epoll_wait 阻塞而不是像select那样每次都要先将进程放入等待队列(具体实现后面讲)==**

**![](E:\Typora\resources\Java\IO\select和epoll.png)**

如下的代码中，先用 `epoll_create `创建一个 epoll **对象** `epfd`，再通过` epoll_ctl` 将需要监视的 socket 添加到 `epfd `中，最后调用` epoll_wait `等待数据:

```java
int s = socket(AF_INET, SOCK_STREAM, 0);   
bind(s, ...)
listen(s, ...)
//创建epoll对象
int epfd = epoll_create(...);
epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中

while(1){
    int n = epoll_wait(...)
    for(/*接收到数据的socket*/){
        //处理数据
    }
}
```

·····

#### Ⅰ、内核的`epoll`原理

##### ①、创建`epoll`对象

当进程执行到`epoll_create()`时,会在内核中创建一个`eventpoll`对象,该`eventpoll`对象包含的最重要的两个成员是`rdlist` 和 `rbr`、`wq`。

1. `rdlist` 

   <img src="E:\Typora\resources\Java\IO\eventpoll结构.jpg" style="zoom:Infinity%;" />

   ==`rdlist` 指的是就绪队列，是一个双向链表结构，当网卡接收完数据后，中断处理程序就会将收到数据的socket引用（实际是将socket引用包装成了`epitem`对象,`rdlist`存放的是`epitem`）添加到`rdlist` 中。还有一个细节就是，程序可能随时调用 epoll_ctl 添加监视 socket，也可能随时删除。当删除时，若该 socket 已经存放在就绪列表中，它也应该被移除。==**`rdlist`的出现解决了`select`的问题2。**

2. `rbr(red black root)`

   既然 epoll 将“维护监视队列”和“进程阻塞”分离，也意味着需要有个数据结构来保存监视的 socket，至少要方便地添加和移除**(实际上监听 这个信息 是记录在sock对象的等待队列中的,他引用着一个`eventpoll`,但是调用`epoll_ctl(OP_DEL,fd)`的时候如何能直接就得知删除哪个sock对象呢?因为sock对象的等待队列引用的`eventpoll`,`eventpoll`并不知道哪个sock对象引用着它,所以才需要rbr创建一个`eventpoll`到sock对象的引用[看不懂先看后面])**，还要便于搜索，以避免重复添加。红黑树是一种自平衡二叉查找树，搜索、插入和删除时间复杂度都是O(log(N))，效率较好，epoll 使用了红黑树作为索引结构

3. `wq`

   `wq`即等待队列,现在进程在阻塞时并不会将引用加入进所有sock等待队列而是直接加入`eventpoll`等待队列,这就使得进程可以直接阻塞。

<img src="E:\Typora\resources\Java\IO\epoll①.jpg" style="zoom:Infinity%;" />

==由此可见分离思想的实现就是基于`wq`结构和sock等待队列,`wq`负责阻塞进程,而sock等待队列的引用`eventpoll`则负责维护监听队列,`rbr`和`wq`的出现解决了`select`的问题1。(看不懂可以先看后面的监听维护,这里只是先做个总结)==

##### ②、维护监视列表

创建 epoll 对象后，可以用 `epoll_ctl` 添加或删除所要监听的 socket。以添加 socket 为例，如下图，如果通过 epoll_ctl 添加 sock1、sock2 和 sock3 的监视，内核会**将 `eventpoll `添加到这三个 socket 的等待队列中,同时会往`rbr`添加sock对象引用。**

<img src="E:\Typora\resources\Java\IO\epoll②.jpg" style="zoom:Infinity%;" />

##### ③、阻塞进程

假设计算机中正在运行进程 A 和进程 B，在某时刻进程 A 运行到了 `epoll_wait` 语句。如下图所示，**内核会将进程 A 放入` eventpoll `的等待队列中，阻塞进程。**

![](E:\Typora\resources\Java\IO\epoll④.jpg)

##### ④、接收数据

当 socket 收到数据后，**中断程序会给 eventpoll 的`rdlist`添加 socket 引用**。如下图展示的是 sock2 和 sock3 收到数据后，中断程序让` rdlist `引用这两个 socket。

<img src="E:\Typora\resources\Java\IO\epoll③.jpg" style="zoom:Infinity%;" />

##### ⑤、唤醒线程

当进程被唤醒后,由于`rdlist`的存在,进程就可以直接知道哪些`socket`中网络IO完毕了。

<img src="E:\Typora\resources\Java\IO\epoll⑤.jpg" style="zoom:Infinity%;" />

#### Ⅱ、`epoll`的两种触发方式

epoll有EPOLLLT和EPOLLET两种触发模式，LT是默认的模式，ET是“高速”模式。 

**LT（水平触发）模式**下，只要这个文件描述符还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作；

**ET（边缘触发）模式**下，在它检测到有 I/O 事件时，通过 epoll_wait 调用会得到有事件通知的文件描述符，对于每一个被通知的文件描述符，如可读，则必须将该文件描述符一直读到空，让 errno 返回 EAGAIN 为止，否则下次的 epoll_wait 不会返回余下的数据，会丢掉事件。如果ET模式不是非阻塞的，那这个一直读或一直写势必会在最后一次阻塞。

![](E:\Typora\resources\Java\IO\LT与ET.png)

采用**ET**这种边缘触发模式的话，当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你

 

> 参考资料
>
> https://www.cnblogs.com/fatmanhappycode/p/12362423.html
>
> https://my.oschina.net/editorial-story/blog/3052308#comments