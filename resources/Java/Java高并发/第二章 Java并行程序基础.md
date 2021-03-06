# 第二章 Java并行程序基础

## 一、线程基本操作 

### ==1、线程的创建和使用==

<img src="E:\Typora\resources\Java\Java高并发\Thread.png" style="zoom:50%;" />

由上图可知,Thread实现了Runnable,Runnable只有一个接口方法就是`run()`

```java
@Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
```

其中的target指的是成员变量`private Runnable target;`。而这个成员变量是在构造函数中调用`init()`方法根据传参进行初始化的,所以就产生了两种创建线程的方法

* 一种则是创建 直接继承Thread类 的MyThread类 的对象,并将`run()`方法覆盖,使用时就可直接调用父类`Thread.start()`执行`MyThread.run()`执行线程
* 一种则是实现Runnable接口,然后通过Thread的构造器将实现Runnable接口的类对象赋值到成员变量target中,随后调用`Thread.start()`执行`Thread.run()`执行`MyRunnable.run`从而执行线程
* 还有一种是实现Callable接口通过FutureTask包装器来创建Thread线程。**Callable必须和FutureTask绑定使用，所以问Callable和Runnable的区别其实就是问FutureTask和Runnable的区别。从继承关系，返回值，调用的方法方面来讲。**

Java不支持多重继承而可以支持多重实现,所以第二种方法会比较解耦。

### 2、线程中断

```java
public void Thread.interrupt()//中断线程

public boolean Thread.isInterrupted()//判断是否被中断
    
public static boolean Thread.interrupted()//判断是否被中断，并清除当前中断状态 
```

**注：interrupt()方法只是将线程置中断标记位(非线程状态)，并不会真正中断线程，如果想要中断线程需在run()方法头进行状态判断从而手动中断**

注：`Thread.sleep()`方法(使线程状态进入TIMED_WAITTING状态)会由于中断而抛出异常，==此时，它会清除中断标记**（注意是中断标记而不是线程状态）**==，那么在下一次循环开始时，就无法捕获这个中断，所以在异常处理时要再次设置中断标记位

### 3、异常处理

对于使用new Thread(··)构造线程的有两种方法处理异常

* 对于受查异常,只能在`run()`内自行使用try-catch块捕获处理
* 对于非受查异常,需要为Thread构造一个UncaughtExceptionHandler以处理非受查异常

### 4、条件队列

#### Ⅰ、基本使用方法及基本原理

`object.wait()`和`object.notify()`方法必须包含在对应的synchronized语句中，两个方法调用之前都要先获得object监视器==**(同一个对象的锁)**==才能执行，**wait()方法执行后会自己释放锁，而notify()不会自己释放锁，要执行完所在`Synchronized`块之后才会释放**

当一个对象在当前线程中调用`wait()`方法后，当前线程将会进入这个==**锁住的对象(不同的锁对象有各自的等待队列)**==的等待队列。这个等待队列可能会有多个线程，而当这个对象的`notify()`方法调用，对象就会从这个等待队列中**随机选择**==(这一点和Condition实现不同)==一个线程，将其==**唤醒(⭐*注:当线程被唤醒后,它在重新请求锁时不具有任何特殊的优先级,而要与其他尝试进入临界区的线程一起正常地在锁上进行竞争,这点和Condition则是一致的*)。**==

#### Ⅱ、由基本原理引申的使用方法

##### （1）等待

###### ①条件谓词

要想正确地使用条件队列,关键是找出对象在哪个条件谓词上等待。**==条件谓词是使某个操作成为状态依赖操作的前提条件。==**如阻塞队列的`put()`,其中队列的非满就是他的条件谓词。

###### ②过早唤醒

由于其唤醒机制(不是唤醒的线程就是下一个获得锁的线程),所以有可能轮到唤醒的线程获得锁的时候条件谓词又为false了。

如生产者消费者模式中，生产者将任务放入阻塞队列后唤醒某条消费者线程，而当生产者释放锁后，另外一条消费者线程获得锁从而将这个任务消费了。此时队列为空，但被唤醒的在同步队列中的线程并不知道这一点，这条线程一旦获得锁后就会向下继续执行获取到空任务从而报错。

所以基于上面的分析，在唤醒之后仍要需要有一个检查机制判断当前状态是否符合条件谓词要求。即

```java
while(!条件谓词)
{
    object.wait();
}
//执行到这里是才会是正确的状态
```

##### （2）通知

###### ①通知时机

确保在`wait()`的条件谓词变为真时发出通知

###### ②`notify()`和`notifyAll()`

由于是每个对象锁拥有着一个条件队列而不是每个条件谓词拥有一个条件队列,所以完全有可能在一个`Synchronized`块中需要使用两个以上的条件谓词(多个条件谓词放在不同的while循环里的情况)。**所以若在这种情况下使用`notify()`将是不安全的,因为这有可能唤醒了一个等待不同条件的线程，而真正等待这个条件的线程却没有收到通知，此时则应该使用`notifyAll()`**

##### （3）`wait/notify`与`Condition`的区别

最大的区别就是，`wait/notify`可以有多个条件谓词，但是因为这些条件谓词而阻塞的线程只存放在一个条件队列上。

而`Condition`同样可以定义多个条件谓词，并且每个条件谓词都可以拥有一个属于自己的条件队列，这样就不会出现`wait/notify`那种调用一次`object.notifyAll()`就把因为各种条件谓词而阻塞的线程都全部唤醒

#### 5、等待线程结束和谦让

`thread.join()`方法让当前线程阻塞直到调用方法的线程对象执行完毕.

==注:`thread.join()`方法的本质是 调用`thread.wait()`方法(`join()`方法外面是有),当`thread`线程执行完毕后退出前调用`thread.notifyAll()`通知所有等待线程继续执行.**所以不要随意调用Thread对象的`join()`,因为有可能会唤醒其他真正因为条件谓词而等待`thread`对象的线程**==

#### 6、停止线程

停止线程的执行方式有三种：

1. 使用已遗弃的`Thread.stop()`。

   该方法的原理就是在栈顶中且是任意代码位置抛出一个`ThreadDeath`错误。要知道，如果`synchronized`块中出现异常而没有异常处理器捕获,是会直接传递到上一个调用栈的。而一旦代码退出了`synchronized`块则锁就会自动释放，所以`ThreadDeath`错误的重要缺陷就是会使得`synchronized`不正常释放。

2. 使用`Thread.interrupt()`中断线程，且线程代码中要检测中断标志位。

   该方法自然比上一个好。但是如果出现代码中有使用到`InterruptedException`的地方,某些方法就会把中断标志位清空如`interrupted()`方法。

3. 自定义`volatile`标志作为停止标志,代码中检测停止标志并根据需要停止线程。

   使用自定义的变量就不会被重置了。

#### 7、⭐守护线程（未完待续）

守护线程会执行到目标线程退出时退出.

#### 8、隐蔽的错误

Integer对象如果被当作锁,那千万不可以改变对象的值,例如如果对`Integer i=0;`使用了`i++`,那么就会新建一个Integer对象并将新值1赋值给它,对象改变,锁失效

#### 9、==synchronized==

关键字synchronized可以有多种用法

* 指定加锁对象:对给定对象加锁,在进入同步代码前要获得给定对象的锁.
* 直接作用于实例方法:相当于对==**当前实例**==加锁,进入同步代码前要获得当前实例的锁
* 直接作用于静态方法:相当于对==**当前类**==加锁,进入同步代码前要获得当前类的锁

#### 10、线程的状态

<img src="E:\Typora\MyNote\resources\Java\Java高并发\Java线程生命周期.png" style="zoom:100%;" />

* NEW：在新建一个Thread对象后就来到NEW状态
* RUNNABLE:该状态可以被看成一个复合状态。它包括两个子状态: READY 和RUNNING。前者表示处于该状态的线程可以被线程调度器(Scheduler)进行调度而使之处于RUNNING状态。后者表示处于该状态的线程正在运行，即相应线程对象的run方法所对应的指令正在由处理器执行。执行`Thread.yield()`的线程，其状态可能会由RUNNING转换为READY。处于READY子状态的线程也被称为活跃线程。
* BLOCKED:一个线程发-一个阻塞式I/O ( Blocking I/O) 操作后,**或者申请一个由其他线程持有的独占资源(比如锁)时，相应的线程会处于该状态**。处于BLOCKED状态的线程并不会占用处理器资源。当阻塞式I/O操作完成后，或者线程获得了其申请的资源，该线程的状态又可以转换为RUNNABLE。
* WAITING:一个线程执行了某些特定方法之后就会处于这种等待其他线程执行另外一些特定操作的状态。能够使其执行线程变更为WAITING状态的方法包括:`Object.wait()`、`Thread.join()`和`LockSupport.park(Object)`。能够使相应线程从WAITING变更为RUNNABLE的相应方法包括: `Object.notify)/notifyAll()`和 `LockSupport.upark(Object)`。
* TIMED_WAITING:该状态和WAITING类似，**差别在于处于该状态的线程并非无限制地等待其他线程执行特定操作，而是处于带有时间限制的等待状态，如`Thread.sleep()`。**当其他线程没有在指定时间内执行该线程所期望的特定操作时，该线程的状态自动转换为RUNNABLE。
* TERMINATED:已经执行结束的线程处于该状态。由于一个线程实例只能够被启动一次，因此一个线程也只可能有一次处于该状态。`Thread.run()`正常返回或者由于抛出异常而提前终止都会导致相应线程处于该状态。

一个线程在其整个生命周期中，只可能有一次处于NEW状态和TERMINATED状态。

#### 11、线程间的通信方式

1. wait/notify
2. join
3. 信号量、CountDownLatch、CyclicBarrier
4. volatile