# Multiple Version Concurrency Control



### 一、MVCC概念

多版本并发控制即MVCC，通过让 读操作读取行的历史版本而不是最新版本，从而实现不加锁的读（因为没有事务会修改旧的行记录，所以对 **不可变**行记录的读取就不用加锁），大大提高InnoDB并发性能。

### 二、InnoDB实现

InnoDB的MVCC主要依赖两部分数据记录事务信息：行记录隐藏列和`read view`结构。

#### Ⅰ、行记录隐藏列

<img src="E:\Typora\resources\Mysql\行隐藏列.png" style="zoom:48%;" />

 	 

* `trx_id`:表示最近一次对行进行增删改操作的事务ID
* `db_roll_ptr`:该列存放着回滚段的指针,**InnoDB通过该指针寻找行的历史版本**。容易看出，行历史版本在回滚段中以链表形式存放。
* ⭐`delete_flag`:另外,无论是`Compact`行记录格式又或者是`Redundant`行记录格式,其行记录头都有一个`delete_flag`的信息记录本行是否已经被删除。所以一个 事务① 对某行删除**(实际上就只是对行记录头的`delete_flag`置true)**之后，若此时一个在事务①之前开始执行的 事务② 要查询这行记录，InnoDB就能够找到该行记录从而根据其`db_roll_ptr`找到行历史版本返回给事务②。

#### Ⅱ、`read view`结构

下图删减了部分成员：

```c
struct read_view_t{  
  
    trx_id_t    low_limit_id;  
                /*!< The read should not see any transaction  
                with trx id >= this value. In other words,  
                this is the "high water mark". */  
    trx_id_t    up_limit_id;  
                /*!< The read should see all trx ids which  
                are strictly smaller (<) than this value.  
                In other words,  
                this is the "low water mark". */  
  
    trx_id_t*   trx_ids;/*!< Additional trx ids which the read should  
                not see: typically, these are the read-write  
                active transactions at the time when the read  
                is serialized, except the reading transaction  
                itself; the trx ids in this array are in a  
                descending order. These trx_ids should be  
                between the "low" and "high" water marks,  
                that is, up_limit_id and low_limit_id. */  
    trx_id_t    creator_trx_id;  
                /*!< trx id of creating transaction, or  
                0 used in purge */  
}; 
```

##### ①、`read view`定义

`read view`就是一个信息载体,**InnoDB用`read view`来判断某事务`SELECT`执行时哪些行记录版本对改`SELECT`可见,哪些行记录版本不可见。**

##### ②、生命周期

* 在可重读事务隔离级别下，在每次事务开始执行前生成一个`read view`，该`read view`的作用范围是整个事务中的所有`SELECT`。
* 在读提交事务隔离级别下，每次事务中的`SELECT`执行前生成一个`read view`，而该`read view`的作用范围是事务内的单条`SELECT`。**所以`Read Commited`违背事务的隔离性的原因就在于此,`read view`的作用范围是单条`SELECT`就意味着 某条`SELECT`能看到 在它之前执行完毕的增删改操作 的记录,也就是说一个事务内的`SELECT`可以看到另一个事务中的修改。**

##### ③、主要成员解析

* `low_limit_id`和`up_limit_id`:这两个成员记录的是 当前事务/sql 开始执行前的一刻 活跃(正在执行)的事务ID的**界限**。（==由于给事务分配的ID是递增的,其大小决定了事务开启的先后顺序,所以有界限这个概念）==`low_limit_id`记录的是活跃的事务ID的最大值（不包括最新开始执行的自己这个事务），`up_limit_id`记录的是活跃的事务ID的最小值。
* `trx_ids`:而`trx_ids`记录的是**所有**当前事务/sql 开始执行前的一刻正在活跃的事务ID(注意`low_limit_id`和`up_limit_id`只记录了最大和最小两个事务ID,那两个成员只是为了InnoDB方便判断是否可见,看下面就懂了)

### ==三、可重读隔离级别的可见性判断流程==

1. 客户端发来`beain`命令,InnoDB生成`read view`,开启事务
2. 当有`SELECT`语句对某行读取时,通过B+树索引或全表扫描找到该行记录,得到该行记录的隐藏列`trx_id`,将其与本事务的`read view`进行比较
   1. 若`trx_id`比`up_limit_id`小,则说明**要查询的行的最后一次修改操作在本事务开启之前**,所以该行版本对本事务可见
   2. 若`trx_id`比`low_limit_id`大,则说明行最后一次修改操作在本事务开启之后,所以该行版本对本事务不可见。**此时就要根据行的`db_roll_ptr`遍历行版本链表，直到找到一个合适的版本（最迟的且对本事务可见的版本）返回**
   3. 若`trx_id`在其两者之间，此时就要寻找`trx_ids`，看最后一次修改此行的事务是否在活跃链表中，如果不在，则说明该事务在本事务开启之前结束了，可见；若在，则要遍历行版本链表找合适版本。

> 参考资料
>
> https://www.codercto.com/a/88775.html
>
> https://blog.51cto.com/yanzongshuai/2103632