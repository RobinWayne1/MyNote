# HashTable

特点：

1. HashTable的默认容量为11，其容量可以随意设置；HashMap的默认容量为16，其容量必须为2的n次幂
2. HashTable的hashcode计算是直接调用key的`hashcode()`（使用时还会&32位1从而将可能为负数的hashcode转换成正数）,而HashMap的hashcode计算则是`key.hashcode()^key.hashcode()>>>16`
3. HashTable的求余是直接使用%,HashMap则是使用性能较高的&运算
4. HashTable的key和value都不允许null值,而`HashMap1.8`和`ConcurrentHashMap1.7`则允许,==⭐所以HashMap要判断key是否存在只能用`containsKey()`而不能使用`get()`判断==
5.  HashTable的扩容后容量是是`2*old+1`,而HashMap为`2*old`
6. HashTable线程安全,HashMap线程不安全

