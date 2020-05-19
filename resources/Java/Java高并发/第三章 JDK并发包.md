# 第三章 JDK并发包

### 一.同步控制

#### 1.Reentrant lock

`lock.lock()`

`lock.unlock()`

一个线程内允许嵌套获得**同一把锁**==(注意:synchronized也可以)==

##### (1)中断响应

`lock.lockInterruptibly()`方法是一个可以对中断进行响应的锁申请动作,即在等待锁的过程中,可以响应中断**(即调用`thread.interrupt()`方法可以中断申请锁)**

##### (2)锁申请等待限时

`lock.tryLock(5,TimeUnit.SECONDS)`表示线程在这个锁请求中最多等待5s,**超过5s没有得到锁则会返回false**

不带参数的方法如果没有得到锁则会立刻返回false==(即在具体源码实现中完全没有用到AQS的阻塞队列,没有进到`acquire()`方法)==

**因此,只要执行足够长的时间,线程总是会得到所有需要的资源,从而正常执行**

##### (3)公平锁

维护一个有序队列,在有线程请求锁但没有请求到时,会进入请求队列等待,当有线程释放锁时,就会按队列顺序派发锁。

==非公平锁性能高于公平锁的一个原因是==：在恢复一个被挂起的线程与该线程真正开始运行之间存在严重的延迟。当持有锁的线程释放锁的同时必须通知操作系统恢复运行阻塞的线程，**==但是如果在恢复过程中有新的已准备好的线程来获取这个锁,如果此时系统将锁资源分配给新线程,那么就完全有可能新的线程已经使用完了锁而刚好到阻塞的线程恢复完成,减少由于线程恢复速度慢而造成的CPU等待时间。==**

==**⭐注：非公平的意义仅仅是在唤醒被阻塞的线程过程中可以插队，而被阻塞的线程的唤醒还是按照AQS阻塞队列先进先出顺序的**==

**这就是最后分析的公平锁与非公平锁源码实现的重要思想.**

#### 2.Reentrant lock搭档:Condition

`condition.await()`会使当前线程等待,同时释放当前锁,当其他线程中使用`condition.singal()`方法,线程会重新获得锁继续执行(==`condition.singal()`方法不会释放锁,要手写代码释放==)

==这里注意和`wait(),notify()`的区别,`wait(),notify()`是对synchronized锁住的对象进行操作,`wait()`进入锁住对象的条件队列 ;而condition是对于ReentrantLock对象,**`await()`*进入condition*的条件队列**,**即一个lock里有notFull和notEmpty两个condition,notEmpty.signal()不会唤醒notFull里条件队列的线程**==

#### 3.允许多个线程同时访问:信号量

* 允许多个线程同时访问

* 指定一个信号量的准入数,即同时能申请多少个许可创建信号量对象,线程释放许可时要调用`release()`方法

#### 4.读写锁

读写分离锁特征: 

* 读和读之间不阻塞
* 读阻塞写,写也会阻塞读
* 写和写阻塞

#### 5.倒计数器

`countDownLatch.countDown()`和`countDownLatch.await()`是两个相对应的方法,前者用作计数,后者用作阻塞线程直到计数完毕(可中断)。用法例子:多个线程的执行体的开头调用`countDownLatch.await()`,等待其他线程中的`countDownLatch.counDown()`使得计数器到达0后,所有调用`countDownLatch.await()`的线程都开始重新执行。==**⭐注:在CountDownLatch对象的state为0时,此时再调用`countDownLatch.await()`则会直接通过不阻塞.**==

==CountDownLatch强调的是一个或多个线程，等待其他一组线程完成操作，再继续执行。==

#### 6.循环栅栏

`public CyclicBarrier(int parties,Runable barrierAction)`

parties即是计数总数,**barrierAction就是当有parties个线程调用`CyclicBarrier.await()`==(有一个线程调用此方法时则完成一次计数 计数+1)==方法后，在释放 调用`CyclicBarrier.await()`的线程 之前所执行的代码。**而之后则会释放阻塞线程让其继续执行。

`CyclicBarrier.await()`方法会抛出两个异常

* InterruptedException:等待过程中线程被中断
* BrokenBarrierException:表示当前CyclicBarrier已经破损,系统没有办法等待所有线程到齐了

==CyclicBarrier强调的是允许**一组线程之间相互等待**，达到一个共同点，再继续执行。==

#### 7.CountDownLatch与CyclicBarrier区别

* CountDownLatch只能使用一次，而CyclicBarrier可以重复使用
* `CountDownLatch.countDown`是减法操作,而`CyclicBarrier.await()`是加法操作
* **==最重要的理解性的区别是两者的*等待对象*不相同==** **(这里还是结合两个类的具体方法进行理解比较好)**
  * **CountDownLatch等待的是其他线程执行完操作,当前线程(多个`await()`线程)才继续执行(`CountDownLatch.countDown()`)**
  * **CyclicBarrier则是线程之间互相等待,等到所有线程都准备完毕才一起开始执行(`CyclicBarrier.await()`)**
  * 在代码实现上两者都可以实现对方的语义,但是这违背了初衷

#### 8.FutureTask

<img src="E:\Typora\resources\Java\Java高并发\FutureTask.png" style="zoom:50%;" />

* 基本原理：线程执行futureTask中的任务，另一个线程通过`futureTask.get()`获取任务执行的返回结果,当任务未执行完成时,`futureTask.get()`将会阻塞直到任务返回。

* 基本用法:由图可见,FutureTask继承了Runnable接口,如果要线程执行任务则只需要新建一个线程。

  ```java
  new Thread(new FutureTask<Object>(new Callable<Object>()
                                    {
                    
                                        public Object call()throws Exception
                                        {
                                            return doThings();
                                        }
                                    }));
  ```

  ==`futureTask.get()`将会返回`callable.call()`方法所返回的对象==
  
* 异常处理:在阐述FutureTask的内部异常处理机制时，要先进行说明：==由于Runnable接口中并没有抛出异常，所以普通的 如继承Thread类或实现Runnable接口方式 并不能直接在创建并使用线程的栈帧中捕获到异常。==而FutureTask解决了这个问题:

```java
 public void run() {
      // ···
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
               //···
            }
        } finally {
            //···
        }
    }
```

用户交给线程的任务是在`Callable.call()`中体现的,而FutureTask将异常提取的工作放在了`run()`方法中,并将提取到的exception赋值到成员变量outcome中(如果没有出现异常则此成员变量就是正常返回结果),当用户调用`FutureTask.get()`之后将异常返回给用户。此举将用户代码与异常捕获解耦，并使 使用线程的栈帧也能获得异常信息。

#### 9.线程阻塞工具类:LockSupport

`LockSupport.park()`方法阻塞当前线程,但是不释放锁.`LockSupport.unpark()`方法取消阻塞

而这和`suspend()与resume()`的区别在于**LockSupport类为每个线程准备了一个许可,如果许可可用,那么`park()`方法会立即返回并且消费这个许可,`unpark()`方法使一个许可变为可用,但是许可不能累加**

==**举例:**==

​	许可本来不可用,

* 如果`park()`在`unpark()`之前,线程没有许可,则`park()`没有消费的许可则进行阻塞,随后`unpark()`使一个许可可用使`park()`方法可以返回

* 如果`unpark()`在`park()`之前,则线程本来就有许可,`park()`会立即返回

`park()`方法支持中断影响,而且不会抛出InterruptedException异常,调用`Thread.interrupt()`方法之后`park()`会默默返回,并且可以通过`Thread.interrupted()`等方法获得中断标记

#### 10.RateLimiter限流

介绍两种限流算法:

漏桶算法:利用一个缓冲区,当有请求进入系统时,无论请求的速率如何,都现在缓冲区保存,然后以固定的流速流出缓冲区进行处理

令牌桶算法:在令牌桶算法中,桶中存放的不再是请求,而是令牌.

* 处理程序只有拿到令牌后,才能对请求进行处理。如果没有令牌,那么处理程序要么丢弃请求,要么等待可用令牌。

* 为了限制流速,该算法在每个单位时间内产生一定量令牌存入桶中。
* 桶的容量有限，如果没有令牌被消耗，只能累计有限单位时间内的令牌数量。

使用`rateLimiter.acquire()`方法时，过剩的流量调用会等待，直到有机会执行

使用`rateLimiter.tryAcquire()`方法时，过剩的流量会丢弃

### 二.线程池

![在这里插入图片描述](http://www.pianshen.com/images/108/265844bed9e9eeb918b14ff2c94d130c.png)

- ==**Executor：一个接口，其定义了一个接收Runnable对象的方法execute，其方法签名为execute(Runnable command)**==

- **ExecutorService：是一个比Executor使用更广泛的子类接口，其提供了生命周期管理的方法，⭐以及可跟踪一个或多个异步任务执行状况返回Future的方法**

  - ```java
    
    public interface ExecutorService extends Executor {
        //使用线程池线程执行任务，注意execute没有返回值
        void execute(Runnable command);
        //⭐不再接受新的任务同时等待已提交的任务完成(虽然interruptIdleWorkers()里有对工作线程的interrupt()但是不会执行)
        void shutdown();
        //⭐不接受新的任务且尝试取消(就是对工作线程调用interrupt(),如果工作线程不响应则一样无法取消)所有运行中的任务,并不再启动队列中未开始执行的任务,返回尚未启动的任务清单
        List<Runnable> shutdownNow();
        boolean isShutdown();
        boolean isTerminated();
        boolean awaitTermination(long timeout, TimeUnit unit)
            throws InterruptedException;
        
        <T> Future<T> submit(Callable<T> task);
        <T> Future<T> submit(Runnable task, T result);
        Future<?> submit(Runnable task);
    
      
        <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
            throws InterruptedException;
        <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                      long timeout, TimeUnit unit)
            throws InterruptedException;
        <T> T invokeAny(Collection<? extends Callable<T>> tasks)
            throws InterruptedException, ExecutionException;   
        <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                        long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException;
    }
    
    ```
    
    ==ExecutorService的生命周期有三种状态:运行、关闭、已终止。ExecutorService在对象创建时处于运行状态，调用`shutdown()`或`shutdownNow()`后处于关闭状态,而在ExecutorService**关闭后提交的任务**将根据拒绝策略采用具体的措施。当这些任务都处理完毕后则进入已终止状态。==

- AbstractExecutorService:ExecutorService执行方法的默认实现

- ScheduledExecutorService：一个可定时调度任务的接口

- ScheduledThreadPoolExecutor:ScheduledExecutorService的实现，一个可定时调度任务的线程池

- ==**ThreadPoolExecutor：表示一个线程池，可以通过调用*Executors的静态工厂方法来创建一个拥有特定功能的线程池并返回一个ExecutorService对象***==

以上成员均在 java.util.concurrent包中, 是 JDK并发包的核心类。其中ThreadpoolExecutor表示一个线程池。 ==Executor**s**类==则扮演着线程池工厂的角色,通过 Executors可以取得一个拥特定功能的线程池。从 UML图中亦可知, ThreadPoolExecutor类实现了 Executor接口, 因此通过这个接口, 任何 Runnable的对象都可以被 ThreadPoolExecutor线程池调度。

#### 1.==Executors工厂类(看准了和Excutor的区别)==

Executor框架提供了各种类型的线程池，主要有以下工厂方法：

##### ①newFixedThreadPool()


创建==固定数目线程==的线程池。该线程池中的线程数量始终不变。当有一个新的任务提交时，线程池中若有空闲线程，则立即执行。**若没有，则新的任务会被暂存在一个任务队列中，待有线程空闲时，便处理在任务队列中的任务。**
```java
public static ExecutorService newFixedThreadPool(int nThreads){
    return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```
##### ②newSingleThreadExecutor()

该方法返回一个==只有一个线程==的线程池。若多于一个任务被提交到该线程池，任务会被保存在一个任务队列中，待线程空闲，按先入先出顺序执行队列中的任务。

```java
public static ExecutorService newSingleThreadExecutor(){ 
     return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));}
```
##### ③newCachedThreadPool()

该方法返回一个可==根据实际情况调整线程数量==的线程池。线程池的线程数量不确定，但若有空闲线程可以复用，则会优先使用可复用的线程。如果现有线程没有可用的，则创建一个新线程并添加到池中。终止并从线程池中移除哪些已经60秒钟未被使用的线程。

```java

public static ExecutorService newCachedThreadPool(){
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());}
```
##### ④newSingleThreadScheduledExecutor()

该方法返回一个ScheduledExecutorService对象，线程池==大小为1==。ScheduledExecutorService接口在ExecutorService接口之上**扩展了在给定时间执行某任务的功能，如在某个固定的延时之后执行，或者周期性执行某个任务。**

```java
public static ScheduledExecutorService newSingleThreadScheduledExecutor(){};
```
##### ⑤newScheduledThreadPool()

该方法返回一个ScheduledExecutorService对象，可以指定线程数量。

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize){};
```

#### 2.核心线程池内部实现

Executors工厂类中不同的工厂方法创建出完全不同功能的线程池,但是其内部都使用了ThreadPoolExecutor类

```java
public ThreadPoolExecutor(int corePoolSize,//线程池中的线程数量
                          int maximumPoolSize,//线程池中的最大线程数量
                          long keepAliveTime,//线程池空闲线程存活时间
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

* corePoolSize:线程池中的线程数量
* maximumPoolSize:线程池中的最大线程数量
* keepAliveTime:线程池线程超过corePoolSize时,多余的空闲线程存活时间**(只对于创建newCachedThreadPool()时有用,因为其他两个工厂方法线程数量不会超过corePoolSize)**
* unit:keepAliveTime的单位
* ==**workQueue:任务队列,被提交但尚未被执行的任务**==
* threadFactory:线程工厂,用于个性化创建线程,如要处理线程池中线程抛出的非受查异常,则可以为Thread构造一个UncaughtExceptionHandler以处理这些非受查异常
* ==handler:拒绝策略.当任务太多来不及处理时如何拒绝任务==

##### ==⭐(1)线程池工作方式==

1. 如果运行的线程少于 corePoolSize，**则 Executor 始终首选添加新的线程（无论是否有空闲线程都会创建新线程）**，而不进行排队。（如果当前运行的线程小于corePoolSize，则任务根本不会存放到queue中)
2. 如果运行的线程等于或多于 corePoolSize，==则 Executor 始终首选将请求加入队列==，**而不添加新的线程**。
3. 如果无法将请求加入队列（队列已满），则创建新的线程，除非创建此线程超出 maximumPoolSize，如果超过，在这种情况下，新的任务将被拒绝。
4. 线程池线程超过corePoolSize时,多余的线程的空闲时间超过了keepAliveTime，则将线程回收

##### (2)workQueue (不要把任务队列和工作线程数混为一谈,任务队列只是一个条件队列)

根据功能分类。有以下几种BlockingQueue 
①直接提交队列，功能由SynchronousQueue提供： 
==使用SynchronousQueue提交的任务不会被真实的保存，而总是将新任务提交给线程执行(**即SynchronousQueue的容量为0,这样就比较好理解Executor的工作方式了**)，**即任何一次插入操作的元素都要等待相对的删除/读取操作，否则进行插入操作的线程*就要一直等待*，反之亦然。**如果线程池中的线程数已经达到corePoolSize还未到达maximumPoolSize，则尝试创建新的线程**(因为该队列容量是0**);如果线程数量已经达到maximumPoolSize，则执行拒绝策略。==因此使用SynchronousQueue队列，通常需要设置很大的maximumPoolSize值。 

**==注:直接提交队列和普通的阻塞队列都有一个条件队列,但是此条件队列非彼条件队列.普通的阻塞队列是生产者将数据放入队列中,而消费者从队列中获取数据;而直接提交队列中的条件队列存储的是生产者线程,等待的是生产者线程而不是数据,你无法窥见这个队列里有什么信息(例如数据),而阻塞队列可以查看这些信息.就好比学生写意见放入意见信箱,信箱里的信件等校长来取,这是阻塞队列;而学生手里拿着信件,等校长有空了,直接交给他,这就是直接提交队列==**

②有界的任务队列，功能由ArrayBlockingQueue提供： 
使用ArrayBlockingQueue时，若有新的任务需要执行，如果线程池的实际线程数小于corePoolSize，则会有限创建新的线程(即不会加入任务队列)，若大于corePoolSize，则会将新任务加入条件队列。**==若任务队列已满，无法加入，则会在总线程数不大于maximumPoolSize的前提下，创建新的线程执行任务。若大于maximumPoolSize，则执行拒接策略。*可见,有界队列仅在当任务队列装满时,才有可能将线程数提升到corePoolSize以上*==** 
③无界的任务队列，功能由对LinkedBlockingQueue提供： 
**与有界队列相比，除非系统资源耗尽，否则无界的任务队列==不会存在任务进入条件队列失败的情况。==**当有新的任务到来，系统的线程数小于corePoolSize时，线程池会创建新的线程执行任务，**但当线程数达到corePoolSize后，又有新的任务加入，==则任务会进入队列等待,不会继续增加线程数。==**所以使用LinkedBlockingQueue的ThreadPoolExecutor的maximumPoolSize并没有作用。若任务的创建速度和处理速度差异很大，无界队列就会快速增长，直到耗尽系统内存。 
④优先任务队列：功能由PriorityBlockingQueue提供： 
它时一个特殊的无界队列，前面的ArrayBlockingQueue和LinkedBlockingQueue都是按照先进先出处理任务的。而PriorityBlockingQueue可以根据任务自身的优先级顺序先后执行。在确保系统性能的同时，也能有很好的质量保证。

##### (3)拒绝策略

==拒绝策略发生条件：==

* 有界队列被填满且线程池线程数到达了maximumPoolSize时
* 有任务被提交到已关闭的ExecutorService时

jdk内置拒绝策略如下:

* AbortPolicy(中止策略):该策略直接抛出RejectExecutionException异常,阻止系统正常工作
* CallerRunsPolicy(调用者运行策略):只要线程池未关闭,**该策略直接在调用者线程中,运行当前被丢弃的任务.**所以任务提交性能的线程性能会急剧下降
* DiscardOldestPolicy(抛弃最旧策略):该策略**丢弃最老的一个请求,也就是即将被执行的一个任务**,并尝试再次提交当前任务
* DiscardPolicy(抛弃策略):默默丢弃无法处理的任务

#### 3.扩展线程池

ThreadPoolExecutor是一个可以扩展的线程池,它提供了beforeExecute(),afterExecute()和terminated()三个接口用来对线程池进行控制,前两个方法将会在执行任务的线程中调用,可以在创建线程池时重写这几个方法.

#### 4.分而治之:Fork/Join框架

将一个大任务fork成多个子任务,交给多个线程分别执行,然后调用join()方法等待所有子任务执行完毕之后主线程才开始继续执行.

在实际使用中,JDK给出一个ForkJoinPoll线程池来对线程进行调度

由于线程池的优化,提交的任务和线程数量并不是一对一的关系,**在绝大多数情况下,一个物理线程实际上是需要处理多个逻辑任务的.因此每个线程必然需要拥有一个双端任务队列.**所以有可能出现这样一种情况:线程A已经把自己的任务都执行完了,线程B还有一堆任务等着处理.**此时线程A就会帮助线程B,从线程B的双端队列的队尾中拿一个任务过来处理,尽可能达到平衡**，==这就是工作密取模式。==

#### 5.线程池对异步调用的支持

```java
	//参数是Callable类的对象,调用后返回FutureTask从而可以获取task的结果
	<T> Future<T> submit(Callable<T> task);
	//参数是Runnable对象,调用后返回Future,直到任务完成后才会返回参数给定的result。(没看错就是给了result然后调用future.get()也返回result)
	//其内部实现则是 通过Executors的继承了Callable的RunnableAdapter内部类,将Runnable包装成具有返回值result的Callable对象(即call()方法直接调用task.run(),然后返回result),然后将这个Callable放进FutureTask里返回给用户
	//建议看源码比较好理解
    <T> Future<T> submit(Runnable task, T result);
	//同上,只不过future.get()返回null(要注意,这里和上面方法绝对不能传一个FutureTask进去,因为结果为null是已经固定了的(newTaskFor(task,null)),所以futureTask的结果并不能返回让用户得到)
    Future<?> submit(Runnable task);
```

这是ExecutorService接口的方法,由AbstractExecutorService实现了这些方法。相较于普通的`Executor.execute()`,与上面FutureTask讲到的一样,都增加了异常返回；同时也增加了个性化的接口调用。当然,`Executor.execute(futureTask)`不算作异步调用,因为他都没有返回值。。

#### 6.CompletionService:Executor与Future的结合

### 三.JDK并发容器

* ConcurrentHashMap:高效的,线程安全的并发HashMap

* CopyOnWriteArrayList:适用于读多写少的场合,具有非常好的性能,远远优于Vector

* ConcurrentLinkedQueue:高效并发队列

* BlockingQueue

* ConcurrentSkipListMap

这里只介绍ConcurrentHashMap,CopyOnWriteArrayList,BlockingQueue,跳跃表的实现和普通跳跃表差不多,只不过插入节点和改变节点值的时候使用了CAS操作,ConcurrentLinkedQueue则完全搞不懂

#### 1.高效读取:不变模式下的CopyOnWriteArrayList(写时复制技术可以和Linux的`fork`联想记忆一下)

 很多应用场景中,读操作远远大于写操作。由于读操作根本不会修改原有的数据，因此对于每次读取都进行加锁其实是一种资源浪费，所以我们应该允许多个线程同时访问List的内部数据。所以CopyOnWriteArrayList只有写入和写入之间需要同步等待。

* **CopyOnWriteArrayList的实现思想：==当这个array需要修改时，并不修改原有的内容，而是对原有的数据进行一次拷贝(利用`Arrays.copyOf`浅拷贝数组元素)，将修改的内容写入副本中。写完之后，再用修改完的副本替换原来的成员变量数组，这样就可以保证写操作不会影响读了。而在读取时，则直接返回(使用)成员变量数组。==**写入操作时会使用重入锁,这个锁是用于控制写-写的情况.**总之:在将数组使用方法```Arrays.copyOf()```后将新的元素加入副本数组,然后替换老数组,修改完成.**

* **而且，这个ArrayList的底层实现数组是这样修饰的:```private volatile transient Object[] array```,所以在修改完之后,读取线程可以立即"察觉"到这个引用覆盖,实现了可见性.**
* ⭐**有一个问题，一个线程正在进行读取操作时，另一个线程对容器进行`setArray()`--即将修改后的数组赋值给成员变量,这是否线程安全?答案是肯定的。要知道,COWAL里的`getArray()`方法会返回原数组的那块堆内存给调用者,而所有的读操作都将使用这个方法访问底层数组而不是直接使用成员变量(直接使用成员变量,例如在进行迭代操作时,迭代的两个元素就有可能不是同一个数组),这样就能保证如果`getArray()`在`setArray()`之前,`setArray()`产生的结果不会影响`getArray()`的调用者对数组的使用。**

通过JMH的测试,在性能方面,get()的性能与ConcurrentLinkedQueue差不多,size()的性能是后者的10几倍,set()性能也比后者要好,所以大多数场景中CopyOnWriteArrayList要优于ConcurrentLinkedQueue

#### 2.数据共享通道:BlockingQueue

有两种JDK内置BlockingQueue:ArrayBlockingQueue和LinkedBlockingQueue.

ArrayBlockingQueue适合有界队列,LinkedBlockingQueue适合无界队列.

##### ①、ArrayBlockingQueue

队列通过维持一个重入锁和两个condition:**notEmpty和notFull分别控制队列为空时的take操作和队列为满时的put操作**

在put()方法中,如果队列元素已满,那么方法将会进入`notEmpty.await()`中,等待有线程将元素put进队列时调用`notEmpty.signal()`将等待线程唤醒;元素为空时同上机制

##### ②、LinkedBlockingQueue

LBQ却使用了锁分离的思想，将`put`和`take`操作分成了两把锁,在添加数据时争抢`putLock`,而在读取数据时争抢`tackLock`

##### ③**PriorityQueue**

默认是一个小顶堆

##### ④为什么ArrayBlockingQueue不用锁分离思想 

LinkedBlockingQueue添加元素时有一个构造节点的时间，为了尽量减少这部分时间占比，使用一个读锁一个写锁可以实现并发存取的优化。而ArrayBlockingQueue在添加元素时不需要开辟空间等等（创建时指定数组大小），所以反而是加锁解锁时间占比较大，如果继续使用这种读写锁分离的设计，增加的时间损耗大于并发读写带来的收益。



### 四.AbstractQueuedSynchronizer

#### Ⅰ、AQS结构

##### 1、基本属性

```java
// 头结点，你直接把它当做 当前持有锁的线程 可能是最好理解的
private transient volatile Node head;

// 阻塞的尾节点，每个新的节点进来，都插入到最后，也就形成了一个链表
private transient volatile Node tail;

// 这个是最重要的，代表当前锁的状态，0代表没有被占用，大于 0 代表有线程持有当前锁
// 这个值可以大于 1，是因为锁可以重入，每次重入都加上 1
private volatile int state;

// 代表当前持有独占锁的线程，举个最重要的使用例子，因为锁可以重入
// reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有了锁
// if (currentThread == getExclusiveOwnerThread()) {state++}
private transient Thread exclusiveOwnerThread; //继承自AbstractOwnableSynchronizer
```

##### 2、CLH阻塞队列（阻塞队列）

```java
//AQS.java
static final class Node {
      // 标识节点当前在共享模式下
    static final Node SHARED = new Node();
    // 标识节点当前在独占模式下
    static final Node EXCLUSIVE = null;
    
    
         //节点从阻塞队列中取消。有三种情况会让节点进入此状态：1、tryAcquire()报错2、tryLock()超时3、tryLock()被中断.注意取消后节点还在阻塞队列中。
        static final int CANCELLED =  1;
       //标识当前节点的后继节点的线程处于等待状态，如果当前节点释放同步状态会通知后继节点，使得后继节点的线程能够运行；
        static final int SIGNAL    = -1;
        //当前节点在条件队列(非阻塞队列)中。
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;
    
    
    
    
    volatile int waitStatus;
	//前驱
    volatile Node prev;
	//后继
    volatile Node next;
	//该阻塞结点属于那条线程
    volatile Thread thread;
	
    /**
         * Link to next node waiting on condition, or the special
         * value SHARED.  Because condition queues are accessed only
         * when holding in exclusive mode, we just need a simple
         * linked queue to hold nodes while they are waiting on
         * conditions. They are then transferred to the queue to
         * re-acquire. And because conditions can only be exclusive,
         * we save a field by using special value to indicate shared
         * mode.
         */
    //由官方文档可以看出,该属性在独占模式下是ConditionObject单向队列的next指针
    //而在共享模式下,该指针则会用来保存SHARED这个Node,以表示当前AQS运行在共享模式下
    Node nextWaiter;
    //返回结点的前驱
     final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
}
```

#### Ⅱ、AQS独占模式

接下来以ReentrantLock实现来讲解AQS独占模式。

首先说明:AQS的阻塞队列是一个带头节点的双端队列(即头节点为空,如何保持头节点为空请看`setHead()`方法)

##### 1.acquire(`lock()`)

```java
//ReentrantLock.java
//非公平锁
//首先要明确一点,ReentrantLock对AQS中state的定义是一个线程中重入锁的次数。
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        //如果此时对state的CAS操作成功,⭐就不用加入aqs的双端条件队列，当state=0时则说明此时锁并没有被占用
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        //如果锁此时正在被当前线程或其他线程占用，则进入AQS继续执行
        else
            acquire(1);
    }
	//tryAcquire()的任务则是要根据语义而对state进行acquire,并返回是否成功
    protected final boolean tryAcquire(int acquires) {
      
        return nonfairTryAcquire(acquires);
       --→ {
               final Thread current = Thread.currentThread();
            int c = getState();
           //当state=0时则说明此时锁并没有被占用
            if (c == 0) {
                //利用CAS操作更改state为所重入的次数
                if (compareAndSetState(0, acquires)) {
                    //设置独占线程为当前线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
           //当state！=0则说明此锁正在被占用，此处判断语句用来判断是否是当前线程占用着，是则可以重入申请
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
           //锁被其他线程占用
            return false;
        }
    }
}
```
==`tryAquire()`要实现的是根据定义对state进行CAS修改以满足语义和**⭐独占模式(state属于被占用状态时不能修改state并返回false)**并返回修改结果,如ReentrantLock的定义是根据锁没有被占用或者锁正在被当前线程占用等情况修改state;==

```java
//AbstartQueuedSynchornizer.java
public final void acquire(int arg) {
    //这里要说一个困惑了很久的问题...例，if(··||··)在||前的语句判断为真的情况下,||后的语句不会执行;而if(··&&··)在&&前判断为假的情况下，&&后的语句也不会执行。在下面的解释代码中同样有这个坑。
    //重复(lock())对锁未被占用的情况和增加对当前线程重入锁的处理
    if (!tryAcquire(arg) &&
        //⭐重点方法
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
###### ①addWaiter()将节点加入阻塞队列尾

```java
//AbstartQueuedSynchornizer.java
//此方法的作用是将当前线程包装成节点从而加入阻塞队列
private Node addWaiter(Node mode) {
    //⭐此时的结点的waitStatus是0,即初始化状态
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    //队尾不为空说明队列中已有结点,即已经被初始化了
    if (pred != null) {
        node.prev = pred;
        //第一次尝试加入阻塞队列尾部,即将Node.tail字段设置为当前node
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //来到这里就要初始化阻塞队列了
    enq(node);
 --→ {//不断循环直到成功入尾
        for (;;) {
            Node t = tail;
            //队列还未初始化,要先将其初始化
            if (t == null) { // Must initialize
                //⭐注意这里,先创建了一个空的头结点放进阻塞队列,而且是还没有return的!!!重点注意!所以要到下一个循环时再开始设置当且结点加入队尾
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                //然后死循环不断尝试将当前结点加入队尾
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
    return node;
}
```

###### ②acquireQueued()线程挂起与获取锁

```java
//⭐AbstartQueuedSynchornizer.java

final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            
            //如果前驱为头节点(空节点)并且tryAcquire操作成功了
            if (p == head && tryAcquire(arg)) {
                //在setHead()之前队列的状态是head为空节点,而head.next则等于node
                setHead(node);
                --→ {
                    //而在setHead()中与方法外的下一步则彻底让当前节点变为空节点,因为方法执行到这里当前节点的线程就已经获得锁了
       				 head = node;
       				 node.thread = null;
       				 node.prev = null;
   				 }
                //顺便方便了GC回收当前节点
                //此时p.next结点还有prev指针指向当前获取锁成功的结点,所以该头结点其实还在队列中
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //迷惑判断语句
            //检测是否需要阻塞，如果不需要阻塞，则后半句不会执行
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
               {  --→{		//挂起当前线程
        				LockSupport.park(this);
                   		//在此状态下，有两种途径可以唤醒该线程：1）被unpark()；2）被interrupt()。
                   		//检测是否挂起时被中断,原因是LockSupport支持中断且不抛出				  							InterruptedException；
                   		//没有中断则会继续自旋(返回false)
        				return Thread.interrupted();
    				}
               
                interrupted = true;
               }
            //继续自旋
        }
    } finally {
        if (failed)
            //只有出现了异常才会来到这里取消,如实现类tryAcquire()出现异常
            cancelAcquire(node);
    }
}
```

```java
//Node的waitStatus
/*⭐int CANCELLED = 1//节点从阻塞队列中取消。有三种情况会让节点进入此状态：1、tryAcquire()报错2、tryLock()超时3、tryLock()被中断.注意取消后节点还在阻塞队列中。
* int SIGNAL = -1//后继节点的线程处于等待状态，如果当前节点释放同步状态会通知后继节点，使得后继节点的线* 程能够运行；
* int CONDITION = -2//当前节点在条件队列(非阻塞队列)中。
* int PROPAGATE = -3//表示下一次共享时同步状态获取将会无条件传播下去
* int INITIAL = 0;//初始状态
*/
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node)
{
    int ws = pred.waitStatus;
    //如果前驱节点的waiteStatus为signal
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        //有前驱节点(线程)被cancel了,需要向前迭代找到第一个没有被cancel的节点,将其作为前驱,并返回false让外层继续自旋
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
      //如果前驱节点的waiteStatus=0或propagate,则将其CAS为signal,并返回false
        //⭐这条分支会在共享模式下起作用
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    //上述两种返回false而不直接修改完返回true的原因,个人猜测是运用自旋锁的思想,让线程空转一会等待,不直接挂起
    return false;
}
```

aquire总结：先从AQS实现类(ReentrantLock)调用面向用户的接口(`lock.lock()`)(ReentrantLock在进入AQS前会先进行一次获取锁操作→记录独占线程和重入次数，如果成功则直接获得锁),然后调用`aquire()`进入到AQS里,然后再调用实现类的`tryAquire()`方法。要注意，`tryAcquire()`只会进行一次。在上述==两次(`lock.lock()`有一次)==尝试对state进行修改失败后，就会来到接下来进入双端带头阻塞队列的操作中----` acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`。` addWaiter(Node.EXCLUSIVE)`负责对阻塞队列的初始化以及将当前线程包装为Node加入阻塞队列队尾。==然后来到`acquireQueued()`方法,由于阻塞队列头节点为空节点,所以当`node.prev=head`为true时,则证明目前可能有一个线程在占用锁(也可能没有线程在占用)并且下一个得到锁的 且在阻塞队列（非公平锁情况的条件） 的线程将会是当前线程,所以此时当前线程将尝试调用`tryAquire()`修改状态获取lock，如果获取成功，则将线程所在的当前节点变为头节点(空节点)==~~(关于阻塞队列操作的线程安全性,入队的操作有`enq()`中无限循环`compareAndSetTail(t, node)`保证,出队即`setHead()`则有`tryAcquire()`保证，且head和tail、waitStatus等等关于队列与node的状态都使用了volatile修饰，保证了CAS操作可见性和有序性)~~。如果获取失败，则根据前驱节点的waitStatus选择自旋或阻塞。

知识点:类与方法的继承实现关系、方法要实现什么功能、`acquireQuue()`中 可以尝试获取锁的前提条件、`acquireQuue()`的阻塞时机和自旋思想、中断处理、具体流程

##### 2.公平锁的实现

首先，公平与非公平就是有插队途径与无插队途径。

==下面,我将"被唤醒的head.next到tryAcquire()的执行时间段和唤醒时间"统称为"唤醒"==

```java

static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

 
    final void lock() {
       //非公平锁的第一次插队，此时如果刚好占有独占锁的线程释放了锁，且 当前线程 执行到此处时,阻塞队列的head.next还未被唤醒,此时就会被当前线程插了队
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
       
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
      
        return nonfairTryAcquire(acquires);
       --→ {
               final Thread current = Thread.currentThread();
            int c = getState();
          //非公平锁的第二次插队,原理同上
            if (c == 0) {
              
                if (compareAndSetState(0, acquires)) {
                  
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
          
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
           
            return false;
        }
    }
}
```

```java
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;
		//⭐公平锁的实现就在于此：比Nonfair类少了判断语句，即少了 一个跳过进入阻塞队列直接修改state从而获得锁的方法
        final void lock() {
            acquire(1);
        }

        //但由于acquire()的实现还是会有第二次尝试跳过阻塞队列获得锁的方法(由于tryAquire()在判断语句的前半部分)
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //判断在阻塞队列中是否有线程等待在当前线程之前
                //⭐接lss注释:所以在此比Nonfair类的tryAcquire()实现多了一个判断方法,同样减少了一个可不通过阻塞队列而插队的途径,使得	允许申请锁的新线程	要不就是阻塞队列只有头节点(最后一个占用锁的线程已经释放锁了)或未初始化(还没有线程因为独占问题被阻塞过)的情况,要不就是当前节点为head.next,即已经进入了阻塞队列排队等到的锁,杜绝了node在唤醒过程中插队的现象
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```

```java
 public final boolean hasQueuedPredecessors() 
{
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
     //为true则代表阻塞队列不为空,不为空意思是有两个或两个以上节点(包括头(空)节点)，还有楼下的正在初始化的特殊情况；为false则代表阻塞队列还未初始化或刚好初始化完成(null或node)
    return h != t &&
        //⭐这里有点复杂。先说明一个前提,只有在h!=t为真时才会执行本条判断语句。由于在enq()初始化中,先进行CAS设置head然后再tail=head,在这两步的间隙中,head!=null而tail=null,且当前只有头节点一个节点,所以此时这条语句就能判断出来阻塞队列在初始化的过程中,判断为true(有线程在对阻塞队列初始化就说明是他先来的,即他刚好遇到了上一个线程独占锁的状态,而我们没遇到,所以是他先来的)；若阻塞队列中本来就有两个节点，则判断为false
        ((s = h.next) == null 
         //若是在初始化过程中不会执行到这里；如果本来就有两个节点，那要具体判断了
         || s.thread != Thread.currentThread());
}
```

==至此可以看出,**公平锁的实现实际上是杜绝 在阻塞队列中的 被唤醒的head.next线程在继续进行自旋过程中,进行到`tryAquire()` 之前的那段时间内(可以一并类比成唤醒过程),新线程不通过阻塞队列直接修改state从而得到锁的情况.(与最上面讲的非公平锁思想对照来看就会发现很好记)**==

知识点：宏观上两者原理的区别、代码实现上两者的区别(如何阻止插队)、阻塞队列在初始化过程中时，`hasQueuedPredecessors()`的判断细节

##### 3.release(`unlock()`)

```java
//ReentrantLock.java
public void unlock() {
    sync.release(1);
  --→{//AQS.java
        if (tryRelease(arg)) {
            Node h = head;
            //得到阻塞队列head.next并将其唤醒
            //解释下后半判断语句。由于shouldParkAfterFailedAcquire()的设计,head,也就是当前正在释放锁线程的以前的结点的waitStatus只有在-1的时候,head.next才 可能 会进入阻塞(即head.next刚刚来到acquireQueued()),否则head.next则一直是自旋状态
            //ls解释错误,如果是这样的话为什么不直接改成h.waitStatus==-1呢?忽略掉ls注释吧
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
}
```

```java
//ReentrantLock.java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    //锁是线程独占的,只有获取锁的线程才能释放锁
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    //如果当前线程的锁全部释放完(没有重入情况了)
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    
    Node s = node.next;
    //如果head.next已经被取消,则要从队尾一直向前找到最后一个未被取消的节点
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

release总结：首先先调用`tryRelease()`尝试修改exclusiveThread和state,此时其他线程就已经可以通过`tryAcquire()`争夺到锁了。但是我们还需要唤醒在阻塞队列阻塞的线程，随后则找到阻塞队列中第一个没有被取消的线程,将其唤醒,让其尝试执行`tryAcquire()`争夺锁。

知识点：执行流程

#### Ⅲ、AQS共享模式

==`tryAquireShared()`要实现的是**⭐共享模式(state无论在什么状态下都能修改成功并返回当前state数目)**,如Semaphore的定义则是根据请求的资源数修改state,如果`state<0`才将线程加入阻塞队列。==

接下来以`ContDownLatch`讲解AQS共享模式:

##### 1、acquireSharedInterruptibly(`await()`)

等待执行的线程使用的方法`await()`,可以看到和`ReentrantLock`类似,都用一个内部类`Sync`来实现AQS的`tryAcquireShared`和`tryReleaseShared`方法

```java
//CountDownLatch.java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
//AQS.java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //老套路,⭐调用实现类的tryAcquireShared()判断是否进行阻塞
    if (tryAcquireShared(arg) < 0)
      →→→
    	{
        //查看state的值是否已到达0,如果已到达0则说明所有线程都已到达,当前线程可以继续执行
          return (getState() == 0) ? 1 : -1;
    	}
        //来到这里就意味着调用countDown()的线程数还不够,要可能要进行阻塞操作
        doAcquireSharedInterruptibly(arg);
}
```

###### ①doAcquireSharedInterruptibly()

```java
//来到这里就意味着调用countDown()的线程数还不够,await()要可能要进行阻塞操作
//看到这里就可以先看countDown()方法了,⭐目前只需要知道await()会将线程包装作结点加入阻塞队列
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    //向阻塞队列添加结点,同时向条件队列添加SHARED结点以表示该节点的共享模式
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();

            if (p == head) {
                // 同上，只要 state 不等于 0，那么这个方法返回 -1
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            //⭐请看ReentrantLock笔记对该方法的解释,在这里只需要知道的一个重点就是,这个方法会将当前线程结点的前驱的waitStatus从初始状态0设置为Signal,后面releaseShared()要用到
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

##### 2、releaseShared()

```java
public void countDown() {
    sync.releaseShared(1);
}
 public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

###### ①tryReleaseShared()

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        //state=0意味着已经countDown完毕,再countDown也没有作用了,因为CountDownLatch只能使用一次
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

就是用CAS操作将`state-1`

###### ②doReleaseShared()

```java
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            //再插入线程结点时,shouldParkAfterFailedAcquire(p, node)已经将前驱(即头结点)的waitStatus设置为了Signal
            if (ws == Node.SIGNAL) {
                //将结点状态从SIGNAL CAS设置为0
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                //请看ReentrantLock的注释,该方法会将当前结点的后继结点唤醒
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

紧接着头结点的后继结点将会被唤醒,看看唤醒后的结点的操作

##### 3、唤醒操作

```java

private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
           
            if (p == head) {
                // 同上，只要 state 不等于 0，那么这个方法返回 -1 
                int r = tryAcquireShared(arg);
                //2、此时state=0了，进入下面的分支
                if (r >= 0) {
                    //3、⭐⭐最重点的实现，把当前结点设置为头结点，并将唤醒其后继结点，没看错，这是在doAcquireShared()方法中调用doReleaseShared()唤醒后继节点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
           //1、被唤醒
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

```java
private void setHeadAndPropagate(Node node, long propagate) {
    Node h = head; // Record old head for check below
    //将头结点设置为当前结点
    setHead(node);
	//下面的方法的意思是:唤醒后继结点,而后继节点被唤醒后,又会执行到此方法,又会将后继结点唤醒,最终所有阻塞队列中的线程都会被唤醒
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

###### ①再谈doReleaseShared()

```java
private void doReleaseShared() {
    
    for (;;) {
        Node h = head;
        
        // 1. h == null: 说明阻塞队列为空
        // 2. h == tail: 说明阻塞队列不为空,且头结点可能是刚刚在enq()中刚初始化的头节点(即enq()还没有执行第二个循环).
        //或者是普通线程节点，但是此普通线程节点既然是头节点了，那么代表已经被唤醒了(⭐头结点只有是空节点时才是阻塞状态,若是普通线程结点,则说明此时唤醒已经在传播)，阻塞队列没有其他节点了
        // 所以这两种情况不需要进行唤醒后继节点
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            //再插入线程结点时,shouldParkAfterFailedAcquire(p, node)已经将前驱(即头结点)的waitStatus设置为了Signal
            if (ws == Node.SIGNAL) {
                //将结点状态从SIGNAL CAS设置为0
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                //请看ReentrantLock的注释,该方法会将当前结点的后继结点唤醒
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        //for循环的退出逻辑
        if (h == head)                   // loop if head changed
            break;
    }
}
```

for循环退出逻辑:

1. h == head：说明头节点还没有被刚刚用 unparkSuccessor 唤醒的线程（这里可以理解为 t4）占有，此时 break 退出循环。

2. h != head：头节点被刚刚唤醒的线程（这里可以理解为 t4）占有，那么这里重新进入下一轮循环，唤醒下一个节点（这里是 t4 ）。我们知道，等到 t4 被唤醒后，其实是会主动唤醒 t5、t6、t7...，那为什么这里要进行下一个循环来唤醒 t5 呢？我觉得是出于吞吐量的考虑。

###### 4、小结

整体思路:

1. 多个线程调用`await()`,来到`doAcquireInterruptibly()`,若`countDown()`线程数还没有到达目标个数,该方法会将全部调用`await()`的线程包装成结点放入一个头结点为空节点的阻塞队列
2. 当多个线程调用`countDown()`使得state=0,此时方法调用就进入了`doReleaseShared()`该方法将唤醒空头结点的后继结点,并将后继结点设置为头结点
3. 当后继节点被唤醒后,在阻塞工作线程的`doAcquireInterruptibly()`方法体内,调用`setHeadAndPropagate(node, r)`进而调用`doReleaseShared`将 ⭐`有线程的头结点的后继节点`唤醒。随着各个结点都变为头结点，也都将头结点的后继节点唤醒并将后继节点变为头结点，从而所有的节点都能够被唤醒。

#### Ⅳ、Condition

一个ReentrantLock可以有多个Condition，每个ConditionObject都有自己的一个条件队列，而阻塞队列则是每个ReentrantLock对象有一个。而条件队列与阻塞队列节点都用Node类来表示，**方便节点从条件队列到阻塞队列的移动**。==注：一个node不可能同时在两个队列上==

##### 1、ConditionObject结构

```java
public class ConditionObject implements Condition, java.io.Serializable {
//条件队列的头节点
private transient Node firstWaiter;
//条件队列的尾节点
private transient Node lastWaiter;
}
```

`ConditionObject`是`Condition`的实现类,它实现了其`await()`、`signal()`等方法

##### 2、await

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //添加结点到队尾，并做了一些初始化操作(无头的单向链表)
    Node node = addConditionWaiter();
    
    //释放当前线程占用的lock,并唤醒阻塞队列的下一个节点(调用release()方法)
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //如果当前结点在阻塞队列中,则说明在执行完fullyRelease()释放锁后,其他线程已经执行完并且已调用condition.signal()了,此时就不需要阻塞
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        //由于park()响应中断,下面判断语句方法就是 在因为中断而取消挂起后,确保当前线程一定进入阻塞队列的操作
      //⭐具体解释看下面代码块 
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //来到这里就说明已经有线程调用condition.signal()了,signal()方法会将结点从条件队列移动到阻塞队列,此时acquireQueued(node, savedState)就在阻塞队列中尝试获取锁,如果获取不了则选择阻塞
    //后面半句的意思是:如果在acquireQueued()过程中tm产生了中断,且上面park()的时候不属于第一种情况,则在最后对线程设置中断标志
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

###### ①`addConditionWaiter()`向条件队列添加节点

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 如果队列中的最后一个结点的状态是被取消的（即Node，cancled,如果在线程没有获取锁时就调用condition.await()方法,fullRelease()方法则会将该结点标记为取消状态）
    if (t != null && t.waitStatus != Node.CONDITION) {
        //(接ls)则从头到尾遍历条件队列删除所有为取消状态的结点
        unlinkCancelledWaiters();
       →→→{
           //就是单纯的遍历队列删除结点
                  Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
       }
        t = lastWaiter;
    }
    //默认新建结点为CONDITION状态
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    //如果队列为空,则先初始化firstWaiter队列头指针,后初始化lastWaiter队列尾指针
    if (t == null)
        firstWaiter = node;
    else
        //若不为空,则与当前队尾结点相连,并将新结点作为队尾
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

###### ②fullyRelease()释放Condition所属的ReentrantLock

```java
final long fullyRelease(Node node) {
    boolean failed = true;
    try {
        long savedState = getState();
        //调用AQS.release()方法释放锁,⭐只有有重入情况或者线程未持有锁才会返回false
        if (release(savedState)) {
         →→→ {
                if (tryRelease(arg)) 
             	{
           			Node h = head;
            		if (h != null && h.waitStatus != 0)
                	unparkSuccessor(h);
            		return true;
       		 	}
        	  return false;
            }
            
            
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        //如果是上面两种情况,则将结点标记为取消,等待后续的加入结点操作检测出
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

###### ③isOnSyncQueue(node)判断当前结点是否已在阻塞队列

```java
   final boolean isOnSyncQueue(Node node) {
    // 移动过去的时候，node 的 waitStatus 会置为 0，这个之后在说 signal 方法的时候会说到
    // 如果 waitStatus 还是 Node.CONDITION，也就是 -2，那肯定就是还在条件队列中
    // 如果 node 的前驱 prev 指向还是 null，说明肯定没有在 阻塞队列(prev是阻塞队列链表中使用的)
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
    // 下面这个方法从阻塞队列的队尾开始从后往前遍历找，如果找到相等的，说明在阻塞队列，否则就是不在阻塞队列

    // 可以通过判断 node.prev() != null 来推断出 node 在阻塞队列吗？答案是：不能。
    // 这个可以看上篇 AQS 的入队方法，首先设置的是 node.prev 指向 tail，
    // 然后是 CAS 操作将自己设置为新的 tail，可是这次的 CAS 是可能失败的。
        return findNodeFromTail(node);
    }
```

###### ④checkInterruptWhileWaiting()检查中断状态,分为在`signal()`调用前中断和`signal()`调用后中断

```java
/*三种情况:
REINTERRUPT： 代表 await 返回的时候，需要重新设置中断状态
THROW_IE： 代表 await 返回的时候，需要抛出 InterruptedException 异常
0 ：说明在 await 期间，没有发生中断
*/
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
// 只有线程处于中断状态，才会调用此方法
// 如果需要的话，将这个已经取消等待的节点转移到阻塞队列
// 返回 true：如果此线程在 signal 之前被取消，
final boolean transferAfterCancelledWait(Node node) {
    // 用 CAS 将节点状态设置为 0 
    // 如果这步 CAS 成功，说明是⭐ signal 方法之前发生的中断，因为如果 signal 先发生的话，signal 中会将 waitStatus 设置为 0
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        // 将节点放入阻塞队列
        // ⭐ ⭐ ⭐ 这里我们看到，即使中断了，依然会转移到阻塞队列
        enq(node);
        return true;
    }

    // 到这里是因为 CAS 失败，肯定是因为⭐ signal 方法已经将 waitStatus 设置为了 0
    // signal 方法会将节点转移到阻塞队列，但是可能还没完成，这边自旋等待其完成
    // 当然，这种事情还是比较少的吧：signal 调用之后，没完成转移之前，发生了中断
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

==所以即使发生了中断，结点依然会转移到阻塞队列。==

```java
 
final boolean transferAfterCancelledWait(Node node) {
    //第一种情况：如果在signal()发生之前产生的中断(即执行transferForSignal()第一条判断语句之前),则将节点加入到阻塞队列
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
   //第二种情况:在signal()发生之后(有可能还没执行到转移的阶段,由于signal()会保证将节点加入阻塞队列,所以这里等到加入后才return false)
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

所以在`await()`过程中中断的机制则是,如果在`signal()`之前被中断,则`await()`方法抛出异常**(但线程此时仍然是获取了锁的)**;如果在`signal()`之后被中断,则只是重新对线程设置中断标志,并让线程继续执行.

await总结:当线程调用`await()`方法后,首先`await()`会将当前线程包装成node加入condition条件队列队尾,然后释放线程持有的锁并从阻塞队列中唤醒一个线程从而让这个线程尝试争夺锁，然后就调用`LockSupport.park()`将当前线程挂起。当有其他线程调用`signal()`后,该条线程可能在阻塞队列中等待被唤醒,也可能在阻塞队列中直接被唤醒了(具体看`signal()`)。(这里不讲中断机制,中断还需看上面理解)由于此时node已经进入了阻塞队列便可以跳出让其挂起的while循环，随后则调用`acquireQueue()`尝试获取独占锁以继续进行,获取成功后便返回。

知识点：==中断机制==、执行顺序、唤醒时机

##### 3、signal

```java
public final void signal() {
    //检测是否是当前线程持有锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

```java
private void doSignal(Node first) {
    do {
        //将当前first移除条件队列,并将first.next变为头节点
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
        //将first转移到阻塞队列
    } while (!transferForSignal(first) &&
             //如果前面语句中的方法转移失败,且条件队列还有线程在,则继续尝试,否则没有线程让其signal,返回
             (first = firstWaiter) != null);
}
```

###### ①transferForSignal()将结点转移到阻塞队列

```java
final boolean transferForSignal(Node node) {
    //⭐将要转移去阻塞队列的node的waitStatus从CONDITION设置成0,如果失败则代表node的waitStatus目前是CANCEL,已经被取消了,即有重入情况或者线程未持有锁的await()操作,该结点还在条件队列但是已经被取消了
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false; 
	//将node加入阻塞队列队尾
    Node p = enq(node);
    //前驱节点waitStatus
    int ws = p.waitStatus;
    //如果前驱节点已取消或CAS ⭐前驱节点 waitStatus操作失败,则直接唤醒node的线程尝试获取锁
    //个人猜测这里直接唤醒node的线程的原因是让其调用shouldParkAfterFailedAcquire()方法，从而将node结点前的已经CANCEL的结点全部删除(这样猜测的原因是acquireQueue()的判断语句p==head暂时是不会成立的)
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    //如果上述条件都不满足,则加入阻塞队列,与其他因为独占而阻塞的线程一起重新争抢锁 ⭐(轮到此node时,将其从await(){LockSupport.park()}中唤醒)
    return true;
}
```

signal总结:线程调用`signal()`,首先`signal()`会移除当前condition条件队列的队头==(按调用`await()`的顺序将结点转移到阻塞队列,且**如果条件队列中没有结点,则直接返回**)==,并尝试将其放入阻塞队列队尾中。期间需要判断这个结点是否被取消，如果判断出被取消了则要取出下一个队头直到当前结点没有被取消。然后调用CAS设置state(从CONDITION到0)与调用`enq()`方法将结点加入阻塞队列。根据条件唤醒线程后,方法结束。

知识点：执行顺序、唤醒时机、唤醒地点