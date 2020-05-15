# Blog 1 数据库缓存的最终一致性方案

### 一、存在问题

存储的数据随着时间可能会发生变化，而缓存中数据就会不一致。具体能容忍的不一致时间，需要具体业务具体分析，但是通常的业务，都需要做到最终一致。

### 二、解决方案

**方案一**

通过key的过期时间，mysql更新时，redis不更新。这种方式实现简单，但不一致的时间会很长。如果读请求非常频繁，且过期时间比较长，则会产生很多长期的脏数据。

优点：

- 开发成本低，易于实现；
- 管理成本低，出问题的概率会比较小。

不足

- 完全依赖过期时间，时间太短容易缓存频繁失效，太长容易有长时间更新延迟（不一致）

**方案二**

在方案一的基础上扩展，通过key的过期时间兜底，并且，在更新mysql时，同时更新redis。

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHq5twbP9AtGufqpAUDqVcgZyNKSOCYkwh8BeLET1DrICyciaFAfTLVGLo1gria5r9MYTnnzQEvxs9A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

优点

- 相对方案一，更新延迟更小。

不足

- 如果更新mysql成功，更新redis却失败，就退化到了方案一；
- ==在高并发场景，业务server需要和mysql,redis同时进行连接。这样是损耗双倍的连接资源，容易造成连接数过多的问题。==

**方案三**

针对方案二的同步写redis进行优化，增加消息队列，将redis更新操作交给kafka，由消息队列保证可靠性，再搭建一个消费服务，来异步更新redis。

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHq5twbP9AtGufqpAUDqVcgmJJQItqVIOm0SQGm575voqVXjRKlkEPq29S6uhWRZXCIL4COA22XnA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

优点

- 消息队列可以用一个句柄，很多消息队列客户端还支持本地缓存发送，==有效解决了方案二连接数过多的问题==；
- 使用消息队列，实现了逻辑上的解耦；
- 消息队列本身具有可靠性，通过手动提交等手段，可以至少一次消费到redis。

不足

- ==**依旧解决不了时序性问题，如果多台业务服务器分别处理针对同一行数据的两条请求，举个栗子，a = 1；a = 5;，如果mysql中是第一条先执行，而进入kafka的顺序是第二条先执行，那么数据就会产生不一致。**==
- 引入了消息队列，同时要增加服务消费消息，成本较高。

**方案四**(???)

通过订阅binlog来更新redis，把我们搭建的消费服务，作为mysql的一个slave，订阅binlog，解析出更新内容，再更新到redis。

![img](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPHq5twbP9AtGufqpAUDqVcgicFArwKyVia2ISCLC3kficfY5lQLTH4Yxo1BIIHjTYACwUszj1XFEicGKQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

优点

- 在mysql压力不大情况下，延迟较低；
- 和业务完全解耦；
- 解决了时序性问题。

缺点

- 要单独搭建一个同步服务，并且引入binlog同步机制，成本较大。