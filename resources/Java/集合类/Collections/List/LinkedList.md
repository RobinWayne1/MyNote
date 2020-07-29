# LinkedList和vector

实现了DeQue接口,底层用双向链表实现List逻辑。由于实现了DeQue接口，LinkedList还可以用实现的`getFirst()`和`addLast()`使其当作队列使用

LinkedList与ArrayList的区别：

1. 效率不同

   * 随机访问时（`get()`和`set()`）,ArrayList占有绝对优势，因为底层以数组实现，所以随机访问时间复杂度为O(1);而LinkedList做随机访问时需要遍历整个链表,时间复杂度为O(n)
   * 增加删除时(`add()`和`remove()`,特指对队头队尾的增删),LinkedList占有优势,因为LinkedList类成员变量存放着firstNode和lastNode,所以增删的时间复杂度为O(1);而ArrayList有时候需要扩容,所以效率要比其低

2. 底层实现不同(数组与双向链表)

3. 内存空间占用

4. 双向链表与单链表




ArrayList和Vector的区别

1. Vector与ArrayList一样，也是通过数组实现的，不同的是它支持线程的同步，即某一时刻只有一个线程能够写Vector，避免多线程同时写而引起的不一致性，但实现同步需要很高的花费，因此，访问它比访问ArrayList慢。
2. 如果集合中的元素的数目大于目前集合数组的长度时，vector增长率为目前数组长度的100%,而arraylist增长率为目前数组长度
   的50%.如过在集合中使用数据量比较大的数据，用vector有一定的优势。