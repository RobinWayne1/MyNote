# JVM参数与调优

| JVM参数                     | 说明                                                         |
| --------------------------- | ------------------------------------------------------------ |
| -Xmx                        | 设置堆的最大内存值，等同于-XX:MaxHeapSize。记法，Memory maximum |
| -Xms                        | 设置JVM启动时，堆的初始化大小，记法，Memory start-up         |
| ⭐-Xmn                       | 设置堆中新生代初始化及最大大小。记法，Memory New             |
| -Xss                        | 设置线程大小，等同于-XX:ThreadStackSize。记法，Stack Size。  |
| -XX:PermGen                 | 设置永久代内存的初始化大小（Java8中开始废弃永久代）          |
| -XX:MaxPermGen              | 设置永久代的最大值                                           |
| -XX:SurvivorRatio           | 设置Eden区和Survivor区的空间比例：Eden/S0 = Eden/S1 默认为8  |
| -XX:NewRatio                | 设置年老代和年轻代的比例大小，默认值为2                      |
| ⭐-XX:PretenureSizeThreshold | 设置大对象进入老年代的对象占用阈值。注意tenure是维持的意思。 |
| ⭐-XX:MaxTenuringThreshold   | 晋升到老年代的对象年龄,默认15                                |
| -XX:HandlerPromotionFailure | 是否允许分配担保失败                                         |
| +HeapDumpOnOutOfMemoryError | 当发生OOM时自动dump出堆信息。dump堆内存信息后，需要对dump出的文件进行分析，从而找到OOM的原因。 |

### 一、简单GC调优

#### Ⅰ、目的

1. 将转移到老年代的对象数量降低到最小，以减少FullGC出现的可能性
2. 减少GC的执行时间

#### Ⅱ、方案

1. 将新对象预留在新生代，由于Full GC的成本远高于Minor GC ，因此尽量将对象分配在新生代是明智的做法，**实际项目中根据GC日志分析新生代空间是否分配合理，适当通过-Xmn 命令来调节新生代大小，最大限度降低新对象进入老年代的情况。**

2. 大对象进入老年代，虽然大部分情况下，将对象分配在新生代是合理。但是对于大对象这种做法却是不太好的，大对象如果首次在新生代分配可能会出现空间不足导致很多年龄不够的小对象被分配到老年代，破坏了新生代的结构，可能出现频繁的Full GC。因此，对于大对象，可以设置直接进入老年代；
   -XX:PretenureSizeThreshold 可以设置直接进入老年代的对象大小。

3. 合理设置进入老年代对象的年龄，-XX:MaxTenuringThreshold设置对象进入老年代的年龄大小，减少老年代的内存占用，降低发生Full GC的频率。

4. 设置合理的堆大小，堆大小设置有两个参数：-Xms 初始化堆大小，Xmx最大堆大小。
   

### 二、JVM监控

#### Ⅰ、jps

Java版的ps命令，查看java进程及其相关的信息，如果你想找到一个java进程的pid，那可以用jps命令替代linux中的ps命令了，简单而方便。

命令格式：

**`jps [options] [hostid]`**

#### Ⅱ、jstat

jstat命令是使用频率比较高的命令，主要用来查看JVM运行时的实时状态信息，包括内存状态、垃圾回收等。**如`jstat -gc 9844 500 20`就会print出年轻代老年代总量和使用量,MajorGC和FullGC次数和耗时信息等。**

命令格式：

**jstat [option] LVMID [interval] [count]**

==其中LVMID是进程id，interval是打印间隔时间（毫秒），count是打印次数（默认一直打印）==

option参数解释：

- -class class loader的行为统计
- -compiler HotSpt JIT编译器行为统计
- -gc 垃圾回收堆的行为统计
- -gccapacity 各个垃圾回收代容量(young,old,perm)和他们相应的空间统计
- -gcutil 垃圾回收统计概述
- -gccause 垃圾收集统计概述（同-gcutil），附加最近两次垃圾回收事件的原因
- -gcnew 新生代行为统计
- -gcnewcapacity 新生代与其相应的内存空间的统计
- -gcold 年老代和永生代行为统计
- -gcoldcapacity 年老代行为统计
- -gcpermcapacity 永生代行为统计
- -printcompilation HotSpot编译方法统计

#### Ⅲ、jmap与jhat

jmap是用来生**成堆dump文件**和查看堆相关的各类信息的命令，例如查看finalize执行队列，heap的详细信息和使用情况。

命令格式：

**jmap [option]  (连接正在执行的进程)**

之后则可以通过**jhat**查看dump文件。

#### Ⅳ、jstack

jstack是用来查看JVM线程快照的命令，线程快照是当前JVM线程正在执行的方法堆栈集合。**使用jstack命令可以定位线程出现长时间卡顿的原因，例如死锁，死循环等。**jstack还可以查看程序崩溃时生成的core文件中的stack信息。

命令格式：

**jstack [-l]  (连接运行中的进程pid)**

#### Ⅴ、jconsole

jconsole就是jstat和jstack等命令的图形化界面，更加的方便

#### Ⅵ、jinfo

jinfo是用来查看JVM参数和动态修改部分JVM参数的命令

命令格式：

**jinfo [option] **

options参数解释：

- -flag <name> 打印指定名称的参数
- -flag [+|-]<name> 打开或关闭参数
- -flag <name>=<value> 设置参数
- -flags 打印所有参数
- -sysprops 打印系统配置
- <no option> 打印上面两个选项