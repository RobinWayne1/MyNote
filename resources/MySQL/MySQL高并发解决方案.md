# MySQL高并发解决方案（大表优化问的是一个东西）

1. 代码中sql语句优化

2. 数据库字段优化，索引优化

3. 加缓存，redis/memcache等

4. 主从，读写分离

5. 分区表

6. 垂直拆分，解耦模块

7. 水平切分

### 一、代码中sql语句优化

将索引.md的各种优化都说出来就完事了。

### 二、数据库字段优化，索引优化

可以讲一下自定义自定义Hash索引的方案和建索引原则

### 三、加缓存

这个就没必要怎么吹了

### 四、主从复制

#### Ⅰ、原理

![](E:\Typora\MyNote\resources\MySQL\主从复制原理图.png)

1. 主库在每次进行事务提交之前，会将事务信息写入binlog中，之后才提交事务（详情请看binlog组提交）。binlog记录DML的顺序是以事务的提交顺序。
2. 从库会开启一个I/O线程，I/O线程建立一个与主库的客户端连接，然后会在主库中创建一个二进制转储线程。这二进制转储线程会读取主库的binlog发送给从库的I/O线程，从库I/O线程则会将这些日志写入到中继日志中。若二进制转储线程赶上了主库的事务提交进度，则会进行睡眠，若有新事务提交则又会被唤醒。
3. 从库的SQL线程会读取中继日志的信息，并在从库执行，从而实现从库数据的更新。

#### Ⅱ、常见问题

##### 1、数据库宕机可能造成从库数据丢失

使用mysql semi-sync（半同步复制）。半同步复制会确保事务提交后binlog至少传输到一个从库（也就是在从库收到事务后才真正提交事务），但是不保证从库运行完这个事务。

使用半同步复制将会降低主库事务提交的性能，响应时间会变长。

##### 2、从库只有一个sql Thread，主库写压力大时会造成从库延时较大

5.6中新增了并行复制机制，规避了一个sql Thread的问题。虽然原理关乎组提交，但还是看不懂我就不看了。

### 五、分区表

#### Ⅰ、分区表的原理

分区表其实就是将一个数据表的文件分散存储在不同的物理文件中，而对于每个分区物理文件，InnoDB都会为其创建与原表相同的索引结构。

对于分区表的DML操作，**MySQL首先会先根据查询中指定的分区所依赖的列的条件确定相应的分区（前提是DML要有这个条件），之后再调用对应的存储引擎接口访问该分区的数据。这个技巧就可以用来过滤大部分数据，相当于粗粒度的索引。**

#### Ⅱ、分区表类型

##### 1、range分区

基于属于一个给定连续区间的列值，把多行分配给分区。

```sql
CREATE TABLE users (  
       id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,  
       usersname VARCHAR(30) NOT NULL DEFAULT '',  
       email VARCHAR(30) NOT NULL DEFAULT ''  
)  engine=innodb
PARTITION BY RANGE (id) (  
       PARTITION p0 VALUES LESS THAN (3000000)  
       DATA DIRECTORY = '/data0/data',
  
       PARTITION p1 VALUES LESS THAN (6000000)  
       DATA DIRECTORY = '/data1/data' ,   
  
       PARTITION p2 VALUES LESS THAN (9000000)  
       DATA DIRECTORY = '/data2/data'  ,  
  
       PARTITION p3 VALUES LESS THAN MAXVALUE     
       DATA DIRECTORY = '/data3/data',   
);
```

##### 2、list分区

类似于按RANGE分区，区别在于LIST分区是基于列值匹配一个离散值集合中的某个值来进行选择。

```sql
CREATE TABLE user (  
     id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,  
     name VARCHAR(30) NOT NULL DEFAULT '' ,
     user_type   int not null
)  
PARTITION BY LIST (user_type ) (  
     PARTITION p0 VALUES IN (0,4,8,12) , 
     PARTITION p1 VALUES IN (1,5,9,13) ,  
     PARTITION p2 VALUES IN (2,6,10,14),  
     PARTITION p3 VALUES IN (3,7,11,15)   
);
```

##### 3、hash分区

这种模式允许DBA通过对表的一个或多个列的Hash Key进行计算，最后通过这个Hash码不同数值对应的数据区域进行分区。其中HASH分区只能针对整数进行HASH，对于非整形的字段只能通过表达式将其转换成整数。

```sql
CREATE TABLE user (  
     id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,  
     username VARCHAR(30) NOT NULL DEFAULT '',  
     email VARCHAR(30) NOT NULL DEFAULT ''  
)  
PARTITION BY HASH (id) PARTITIONS 4 (  
     PARTITION p0 ,  
     PARTITION p1,  
     PARTITION p2,
     PARTITION p3  
);
```

##### 4、key分区

与hash分区类似，不过KEY分区支持除text和BLOB之外的所有数据类型的分区，而hash只支持整数。

#### Ⅲ、分区表的优点和缺点

##### 1、优点

1. 分区表的数据可以分布在不同的物理设备上，从而更加高效的使用多个物理设备
2. 分区表的数据更容易维护，如可以对一个独立分区进行优化、修复、检查或者是删除
3. 一些查询可以得到极大的优化，这主要是借助于满足一个给定WHERE语句的数据可以只保存在一个或多个分区内，这样在查找时就不用查找其他剩余的分区。
4. 数据减少**有可能**使表的B+数索引层数减少，从而减少磁盘IO产生
5. 相较于分库分表，服务器端不需要太大改动

##### 2、缺点

1. 如果索引列和分区列不匹配，那么在DML查询该索引列时就会扫描所有分区。由于每个分区表都是独立的文件，所以单个表文件可以2~3次IO完成的查询在分区表下可能就要进行十几次的IO（就是因为不匹配，所以要扫描所有分区以保证查询准确）
2. 分区越多，MySQL对于过滤某查询属于哪个分区的成本就会越高，因为MySQL需要扫描所有分区来得知这个答案
3. 如果分区字段中有主键或者唯一索引列，那么所有主键列和唯一索引列都必须包含进来

### 六、垂直拆分

垂直拆分的常见方式是根据表中字段的活跃度进行拆分，将不经常查询的字段放到另外一张表中。具体的方法是两张表都以ID作为冗余主键，但是有不同的字段。

优点：

1. 可以使得行数据变小，一个数据块(Block)就能存放更多的数据，在查询时就会减少I/O次数(每次查询时读取的Block 就少)

缺点：

1. 需要管理冗余列
2. 分表之后表连接的操作就会增多，这就需要根据业务权衡
3. 事务的处理变得复杂了 

### 七、水平拆分

水平拆分是通过某种策略将数据分片来存储，分库内分表和分库两部分，每片数据会分散到不同的MySQL表或库，达到分布式的效果，能够支持非常大的数据量。前面的表分区本质上也是一种特殊的库内分表。

库内分表，仅仅是单纯的解决了单一表数据过大的问题，**由于没有把表的数据分布到不同的机器上，因此对于减轻MySQL服务器的压力来说，并没有太大的作用，大家还是竞争同一个物理机上的IO、CPU、网络，这个就要通过分库来解决。**

优点：

1. 不存在单库大数据和高并发的性能瓶颈
2. 应用端改造相比垂直拆分较少
3. 提高了系统的稳定性和负载能力

缺点：

1. 分片的事务一致性难以解决
2. 跨结点连接性能极差