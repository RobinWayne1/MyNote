# MVCC实现机制之InnoDB具体实现

首先来讲read view,他是一个数据结构

* 在可重读隔离级别下,==**每个事务开始前创建一个read view结构**==
* 而在读提交隔离级别下,**==事务里的每条sql开始前创建一个read view结构==**

read view的主要成员如下:

> * low_limit_id：表示创建read view时，**当前事务==*活跃读写链表*最大==的事务ID，即最近创建的除自身外最大的事务ID(个人猜测如果最大是自己会加个1)**
>
> * up_limit_id：表示创建read view时，**当前事务==*活跃读写链表*最小==的事务ID。==即(up_limit_id,low_limit_id)，当前事务id处在这个范围之中==**
>
> * **==trx_ids：创建read view时，活跃事务链表里所有事务ID,*在RR隔离级别下,所有在此链表里的事务更改的行都不可读*==**
>

**注意:当进行insert,update,delete时,InnoDB会将当前事务ID写入行记录的隐藏列data_trx_id中**

### 一、在RR隔离级别下，==在事务开始前的快照的读==

当要读取数据时,InnoDB会将行数据的data_trx_id与read view中的成员比较:

* 如果`data_trx_id>=low_limit_id`(low_limit_id不包括自己),则此行数据对这个事务是不可见的**(即某事务在当前事务后创建且修改了行数据提交了,那这行数据对当前事务不可见)**
* 如果`data_trx_id<up_limit_id`,即在当前事务开始前修改此行的事务已经提交,所以该记录可见
* 如果`up_limit_id<data_trx_id<low_limit_id`，则需要查看更改当前行的事务id(data_trx_id)是否存在于trx_id中,==**如果存在,即该事务在当前事务开始前还未提交,该记录不可见**==。如果不存在，即该事务在当前事务开始时已经提交了，可见。(疑问的解答在于此,在此控制同时执行的事务的事务隔离性)

![](E:\Typora\resources\MySQL\2100026-19f3a841076a7140.png)

==注:如果读数据不符合条件会根据db_roll_ptr来到undo log链表对比trx_id(即为被修改版本号),找到版本号最新的且不大于事务版本号的数据返回。==**(即在找到不符合条件的刚INSERT的行时迭代undo log链表也不会找到旧版本数据,所以其他事务是用不到insert undo log的数据的)**

