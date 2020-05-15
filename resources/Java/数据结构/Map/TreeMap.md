# TreeMap

特点:

1. TreeMap基于红黑树实现,与HashMap的树化不同的是,TreeMap没有内置的类似HashMap用作树化的`tieBreakOrder()`比较工具**（TreeMap强调key有序，但HashMap只有树化后key才有序）**，==所以放入的key值必须实现Comparable接口或在构造器中指定Comparator以便在访问红黑树时进行比较==

2. 由于红黑树必须依据key进行比较，所以key不能为null；而HashMap的key可以为null

3. 由于红黑树为了保持平衡需要在插入/删除红黑树时进行旋转，所以插入/删除效率没有HashMap高

   