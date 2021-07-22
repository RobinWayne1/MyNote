# MappedByteBuffer

## 一、基本使用

```java
public class MappedByteBufferTest {
    public static void main(String[]args) throws Exception{
        byte[]arr = new byte[20];
        MappedByteBuffer mappedByteBuffer =
                //设置可读写模式
                new RandomAccessFile(new File("e://test.txt"),"rw")
                 //获取FileChannel实例
                .getChannel()
                        //内存映射文件
                        .map(FileChannel.MapMode.READ_WRITE,0,20);
        for(int i = 0;mappedByteBuffer.remaining() > 0;i++){
            // 获取一个字节，调用该方法将移动position指针
            arr[i] = mappedByteBuffer.get();
        }
        System.out.println(Arrays.toString(arr));
        //修改共享库内存映射区域的数据
        mappedByteBuffer.putChar(14,'艹');
    }
}
```

Demo如上所示，再解释下mmap的入参调用细节。

```javajava
 MappedByteBuffer map(MapMode mode, long position, long size)
```

* `mode`：代表工作模式，如果设置`MapMode.READ_WRITE`那就支持可读写,但是前提条件是`RandomAccessFile`也设置了`rw`模式
* `position`：读取文件内容的起始位置
* `size`：读取文件内容的范围限制

## 二、基本概念

MMU：内存管理单元，也就是地址变换机构，负责管理页表以及虚拟地址到物理地址的转换。

虚拟内存：即非连续存储管理，即让程序认为其所使用的内存是连续的，而实际上在物理内存中分配成了多个物理内存碎片。

虚拟地址：虚拟地址指的就是在OS底层是非连续存储管理（虚拟内存）条件下，程序寻找内存时所使用的地址。如分页存储管理中的逻辑地址，由页号+页内偏移量表示。

> 这里仅按我当前阶段理解范围内来解释，因为我所学的OS只是和Linux内核应该有些差异。

## 三、mmap原理

<img src="E:\Typora\MyNote\resources\Java\IO\linux进程内存空间与内存映射文件.jpg" style="zoom:25%;" />

<img src="E:\Typora\MyNote\resources\Java\IO\linux进程内存空间与内存映射文件2.png" style="zoom:85%;" />

linux进程**==虚拟内存（注意是虚拟不是物理,再进一步说这里某些区域物理上并不是一个进程独占的）==**的空间分配如上图所示，**其中text数据段、堆、栈等都是独立的虚拟内存区域，而还有一大部分的虚拟内存区域没有被分配存放的数据类别，这部分区域就是内存映射文件中所使用到的虚拟内存区域**。

### Ⅰ、读原理

1. 在程序调用**mmap函数**时，(没有主语)首先会寻找一段连续的虚拟内存区域以专门用来存放所需要映射的文件，**将这段虚拟内存区域的起始地址、结束地址、`vm_ops`(表示针对这段区域可以使用的系统调用函数)等信息存放到`vm_area_struct`虚拟区域结构中,然后将这个结构插入进程的虚拟地址区域链表(链表中存放的是进程虚拟区域的信息)**(这块内存分配逻辑建议看原文)。**==这块虚拟内存区域所对应的物理内存页其实是内核程序与用户程序共享的，只不过内核程序与用户程序都有自己的不同的虚拟地址对应这一块物理内存页，在两者看来他们都在操作着不同的内存区域，而物理上其实是同一内存区域。换句话说,这块区域即是PageCache,也是用户空间的虚拟内存==**

   ![](E:\Typora\MyNote\resources\Java\IO\用户空间映射PageCache.jpg)

2. 内核进程通过一系列步骤找到所需要映射的磁盘物理地址。然后建立进程页表，请求页表项中记录页框对应的磁盘地址，此时映射建立完成。

3. 读取内存数据，查看请求页表发现无法找到页号具体对应页框时触发缺页异常，内核从而通过请求页表中记录的文件所在磁盘地址寻页或选择内存一页换出或分配其 内存新页。

### Ⅱ、写原理

Linux读写文件都要经过PageCache，所以这就是mmap要设计成用户空间虚拟内存映射PageCache的用意。而通过mmap映射的文件在被修改的时候就要从映射区域刷回磁盘，这个同步操作有4个时机：

* 调用专供mmap使用的同步函数`msync`,类似`fsync`阻塞直到buffer刷盘成功
* 用 `munmap` 函数对文件进行解除映射关系时
* 进程退出时
* 系统关机时

## 四、`DirectByteBuffer`与`MappedByteBuffer`与`HeapByteBuffer`

先看看Linux进程地址空间与JVM的关系,看完就会觉得很清晰。

<img src="E:\Typora\MyNote\resources\Java\IO\Linux用户空间与JVM的关系.jpg" style="zoom:50%;" />

<img src="E:\Typora\MyNote\resources\Java\IO\Linux内核空间与JVM的关系.jpg" style="zoom:50%;" />

* `HeapByteBuffer`:读取数据时从页缓冲拷贝到Linux运行时堆(Java堆外),然后再将数据拷贝到Java堆内。使用`ByteBuffer.allocate()`获得
* `DirectByteBuffer`:读取数据时从页缓冲拷贝到Linux运行时堆(Java堆外),然后在Java堆内new一个`DirectByteBuffer`对象直接引用堆外数据。使用`ByteBuffer.allocateDirect()`获得
* `MappedByteBuffer`:在Linux进程虚拟内存中的未使用部分分配一块区域映射页缓冲,在真正访问文件数据时产生缺页异常以进行数据读取。使用`FileChannel.map()`获取。

## 五、使用场景与局限性

### Ⅰ、使用场景

1. 在零拷贝中曾讲过，大文件无法受到页缓冲的好处，因为页缓冲最重要的作用其实是缓冲热点数据，CPU会优先读取页缓冲然后再去找磁盘。而大文件读取一般是不会重复读取最近曾读取过的数据的，此时内核缓冲区就失去了缓冲的作用。所以大文件的读取就非常适合mmap，因为舍弃了页缓冲的缓存作用并且减少了一层拷贝。

2. 通过上面的三大`ByteBuffer`区别可以看出,mmap几乎不受JVM的控制(没有虚拟机参数限制大小,因为数据存放的地点甚至不在Linux堆)。然而实际上使用`MappedByteBuffer`最多可以读取2G大的文件，因为`MappedByteBuffer.size`是long型，最多可表示2G大小，再大就溢出了。

   还有一个注意的地方就是，这块区域是虚拟内存，由于虚拟存储器请求调页机制的存在，除去size和页号表示大小限制其实是可以读取无限大的文件的。这个特点相比于其他两种`ByteBuffer`就是又一大亮点。

### Ⅱ、局限性

1. 资源的释放比较麻烦，`FileChannel`提供了`map()`方法却没向外提供`unmap()`,导致难以释放内存。可以通过下面的代码释放

   ```java
   // 在关闭资源时执行以下代码释放内存
   Method m = FileChannelImpl.class.getDeclaredMethod("unmap", MappedByteBuffer.class);
   m.setAccessible(true);
   m.invoke(FileChannelImpl.class, buffer);
   ```

> 参考资料
>
> https://www.jianshu.com/p/f90866dcbffc
>
> https://stackoverflow.com/questions/46190787/does-mmap-directly-access-the-page-cache-or-a-copy-of-the-page-cache
>
> https://www.cnblogs.com/huxiao-tee/p/4660352.html#_label5
>
> https://blog.csdn.net/sinat_42483341/article/details/106780536
>
> https://blog.csdn.net/zhouguoqionghai/article/details/112771864
>
> https://blog.csdn.net/linxdcn/article/details/72903422?spm=1001.2014.3001.5501
>
> https://www.cnblogs.com/huxiao-tee/p/4657851.html#_label5
>
> ⭐https://zhuanlan.zhihu.com/p/366964820 
>
> https://zhuanlan.zhihu.com/p/104994838
>
> https://cloud.tencent.com/developer/article/1057666
>
> 《深入了解Linux内核》