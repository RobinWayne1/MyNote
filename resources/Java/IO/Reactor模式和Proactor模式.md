# Reactor和Proactor

###  Reactor

<img src="E:\Typora\MyNote\resources\Java\IO\Recator.png" style="zoom:67%;" />

读操作为例**(要注意在NIO中事件分离器应该是我们自己写的判断事件类型的代码,而`Selector`只是做了一个轮询返回的工作)**

1. 当IO事件数据传输完毕（通过轮询检测）的时候，事件分离器调用相应的事件处理器

2. 事件处理器首先执行实际的读取操作（内核→缓冲区），然后根据读取到的内容进行进一步的处理

### Proactor（图从右往左看）

<img src="E:\Typora\MyNote\resources\Java\IO\Proactor.png" style="zoom:67%;" />

读操作为例

1. IO事件发生，操作系统负责事件的IO处理，处理过程包括等待数据传送到内核且将内核的数据复制到用户空间（缓冲区）

2. 操作系统完成某事件IO后，通知事件分离器（这个事件分离器就像是我在JavaAIO中讲的调用`CompletionHandler`的通知）

3. 事件分离器捕获到读取完成事件后，激活应用程序注册的事件处理器(调用`CompletionHandler`)，事件处理器直接从缓存区读取数据，而不需要进行实际的读取操作。
   



> 参考资料
>
> https://blog.csdn.net/caiwenfeng_for_23/article/details/8458299