# ConcurrentHashMap 1.7

### 一、成员变量

```java

//用于存放寻找segment的掩码值，key.hash的高位用于选择segment
final int segmentMask;

//移位数,用作计算段下标,具体看put()
final int segmentShift;

//segments数组
final Segment<K,V>[] segments;
```

### 三、构造方法

```java
 public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
       //参数concurrencyLevel用来控制段的数量,其必须小于 MAX_SEGMENTS
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
//---------------------------------------------------------------------------------------
     //concurrencyLevel用于设置map中段的数目
     //找到最接近concurrencyLevel的 2的n次幂ssize,⭐将ssize作为初始存放段的数组的容量
        int sshift = 0;
        int ssize = 1;
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }   
     
        this.segmentShift = 32 - sshift;
     //⭐重点关注这里，剑指offer的解题思路,在上面的循环体中找到第一个比concurrencyLevel大的 2的n次幂后,再减1,其右边的二进制位就全部变成1了,这就是掩码的计算方法.(与操作时并不只是直接与这么简单)
     //关于segmentShift和segmentMask的具体用法再put()过程中会讲
        this.segmentMask = ssize - 1;
//---------------------------------------------------------------------------------------
     // initialCapacity 是设置整个 map 初始tables的大小，
    // 这里根据 initialCapacity 计算 Segment 数组中每个位置可以分到的大小
     if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
      
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
            ++c;
     //cap即代表每个段中的table数目
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;
//---------------------------------------------------------------------------------------
        //创建Segment数组和第一个segement,其他segement将会lazy-init
        Segment<K,V> s0 =
            
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
     //⭐将ssize作为初始存放段的数组的容量
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
     //将第一个segement放入Segement数组中
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }

```

### 四、数据载体

#### Ⅰ、HashEntry

HashEntry就相当于`HashMap`的`Node`,table中发生hash碰撞的数据将以链表的形式存放

```java
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
     /**
      * 使用volatile写入语义设置next域
      */
     final void setNext(HashEntry<K,V> n) {
         UNSAFE.putOrderedObject(this, nextOffset, n);
     }
}
```

#### Ⅱ、Segment

```java
//成员变量
//tryLock()的自旋重试次数
static final int MAX_SCAN_RETRIES =
    Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

//加入分段锁之后,table就属于Segement管理了

//要注意volatile修饰数组则只对数组的引用有效,而对数组内的元素无效
transient volatile HashEntry<K,V>[] table;

//记录段内所有table的结点数目
transient int count;

//⭐不再是迭代器相关,而是用作size()统计结点数目时判断是否有并发情况
transient int modCount;

//table.length'*loadfactor,当count>threshold时扩容
transient int threshold;

//扩容因子
final float loadFactor;
```

**重点注意:`ConcurrentHashMap`中的table现在是受段管理了,而不直接是map管理table,所以构造方法中所有指标参数都是基于段内部的,而不是基于整个map的**

### 五、增删查分析

#### Ⅰ、插入

```java
//ConcurrentHashMap.class
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    //⭐计算hash
    //这个方法百度了一下没人看过,我也着实看不懂
    int hash = hash(key);
    // ⭐ 根据 hash 值找到 Segment 数组中的位置 j
    //    hash 是 32 位，无符号右移 segmentShift(28) 位，剩下高 4 位，
    //    然后和 segmentMask(15) 做一次与操作，也就是说 j 是 hash 值的高 4 位，也就是Segment数组(segements)的下标
    int j = (hash >>> segmentShift) & segmentMask;
    //获取相应下标的segment
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        //对 segment[j] 进行初始化,使用segment[0]的参数
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```

`Segement`作为一个锁管理着数个table,所以对table的具体操作也全都封装在了`Segement`对象中

```java
 final V put(K key, int hash, V value, boolean onlyIfAbsent) {
        HashEntry<K,V> node = tryLock() ? null :
     //当从这个方法中返回时,当前线程就是已经成功获取锁的状态
            scanAndLockForPut(key, hash, value);
        V oldValue;
        try {
            HashEntry<K,V>[] tab = table;
            //求余获得目标桶下标
            int index = (tab.length - 1) & hash;
            //获得目标桶的头结点
            HashEntry<K,V> first = entryAt(tab, index);
            for (HashEntry<K,V> e = first;;) {
                if (e != null) {
                    K k;
                    //如果发现相同key的HashEntry,修改其值并返回
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        oldValue = e.value;
                        if (!onlyIfAbsent) {
                            e.value = value;
                            ++modCount;
                        }
                        break;
                    }
                    //没找到就继续便利
                    e = e.next;
                }
                //当指针e遍历到链表尾或该桶中本来就没有结点
                else {
                    //node.next=first
                    if (node != null)
                        node.setNext(first);
                    else
                        node = new HashEntry<K,V>(hash, key, value, first);
                    
                    //扩容
                    int c = count + 1;
                    if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                        rehash(node);
                    //⭐将新节点作为目标桶的头节点
                    else
                        setEntryAt(tab, index, node);
                    ++modCount;
                    count = c;
                    oldValue = null;
                    break;
                }
            }
        } finally {
            unlock();
        }
        return oldValue;
    }
```

`put()`过程逻辑总结:

1. 通过key计算出hash,并**将hash的高位与段掩码相与求得目标段的数组下标**

2. 自旋获得该段的锁

3. **通过求余得到目标table的下标**,遍历目标table的链表,寻找是否有相同key的结点,若有则替换其value

4. 若没有,新建一个结点,将该结点插入链表头(`HashMap`是插入链表尾)

##### ①scanAndLockForPut()

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    //根据hash值获取相应的桶,和HashMap一样也是使用hash&(tab.length - 1)求余
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    //再一次尝试获取锁,如果失败则开始自旋过程
    while (!tryLock()) {
        HashEntry<K,V> f; 
        //当自旋重试次数<0
        if (retries < 0) {
            //e=null的含义是桶中尚未有结点或此时桶中遍历链表的指针已经到达了尾部
            if (e == null) {
                //且尚未创建一个新节点以供当前put()使用
                if (node == null) 
                    //创建一个新结点
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            //寻找到了目标节点
            else if (key.equals(e.key))
                retries = 0;
            //未找到目标节点,则尝试遍历链表找到目标节点
            else
                e = e.next;
        }
        //当自旋次数到达MAX_SCAN_RETRIES,直接进入AQS的同步队列
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        //当自旋次数是偶数
        else if ((retries & 1) == 0 &&
                 //检查到目标桶的头节点已被其他线程修改,则重新寻找key相同的目标节点
                 //this指Segement
                 (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    //⭐请注意,无论指针e指向哪个结点或while(!tryLock())循环体内出现怎样的竞态条件都好,返回的只是node,而这个node和桶中链表的节点没有任何关系,只是存粹的一个新节点
    return node;
}
```

总的来说,此方法主要的作用是**自旋获取锁**,但在自旋过程中顺便遍历桶中链表寻找目标key结点**并可能实例化返回一个与桶链表无任何联系的新节点**。**==显然,在临界区外对临界资源table中链表的访问是没有意义的,那为什么`scanAndLockForPut()`方法的实现中要遍历链表找到目标节点呢?这是一种优化方式,主要是尽可能使当前链表中的节点进入CPU高速缓存，提高缓存命中率，以便获取锁定后的遍历速度更快。==**

##### ②rehash

⭐当**==段内==**新增结点后**==段内==**结点总数大于**==段内==**threshold(`table.length*loadFactor`),则要==**对段**==进行扩容

```java
private void rehash(HashEntry<K,V> node) {
 
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    //和HashMap一样,扩容后容量翻倍
    int newCapacity = oldCapacity << 1;
    //新阈值
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            //此次求余操作
            int idx = e.hash & sizeMask;
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                // 下面这个 for 循环会找到一个 lastRun 节点，这个节点之后的所有元素的新的求余值都相同,所以这之后的元素都可以放在一起
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                   
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                //直接将lastRun后的节点放入新table
                newTable[lastIdx] = lastRun;
                //将lastRun前面的结点逐个求余拷贝去相应的新table
                //⭐⭐⭐可以看见，这种扩容时复制的算法使得重新映射后(扩容时旧桶内的元素只会复制到两个桶中)两个桶中的链表元素的相对顺序改变,这点不同于1.8的HashMap
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    //这个方法扩容和新增节点绑定在了一起
    //将新节点加入进新table
    int nodeIndex = node.hash & sizeMask; 
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```

**==扩容逻辑:==**

1. 当段内结点总数大于cap(桶个数)*loadFactor时,进行扩容
2. 开始逐个table转移结点
3. 先遍历一次table的链表,找出 lastRun结点,将以lastRun结点为头的链表直接赋值去新table中。**先遍历一遍的作用就在于此,==如果lastRun后面有较多的结点,那么这种直接赋值的转移方法更能够节省内存和时间==**
4. 从旧table的链表头开始逐个**将结点插入到其相应的新table的链表头中**
5. 最后插入新结点

#### Ⅱ、查找

```java
public V get(Object key) {
    Segment<K,V> s; 
    HashEntry<K,V>[] tab;
    int h = hash(key);
    //计算目标段下标
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    //⭐使用volatile操作获取目标段
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        //⭐使用volatile操作获取段中的目标桶
        //遍历链表找到目标结点的value
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

非常简单的查找逻辑:

1. 根据key.hash值volatile操作获取目标table
2. 遍历桶结点找到key相同的目标结点

#### Ⅲ、遍历

和`HashMap`1.8的实现类似,都有三种可遍历对象:`EntrySet`、`KeySet`、`ValueSet`,而这三种对象的迭代器实现都继承自`HashIterator`,只是每个子类迭代器取`HashEntry`里的不同的值而已.

```java
abstract class HashIterator {
    	//用作记录遍历中的段在segments[]中的下标
        int nextSegmentIndex;
    	//用作记录遍历中的桶在table[]中的下标
        int nextTableIndex;
        //记录正在遍历的段
        HashEntry<K,V>[] currentTable;
    	//记录将要遍历的下一个结点
        HashEntry<K, V> nextEntry;
        HashEntry<K, V> lastReturned;

        HashIterator() {
            nextSegmentIndex = segments.length - 1;
            nextTableIndex = -1;
            //初始化
            advance();
        }

        //寻找下一个遍历的结点
        final void advance() {
            for (;;) {
                //nextTableIndex大于等于0说明该段还有table没遍历,取该table的头节点作为下一个遍历的结点
                if (nextTableIndex >= 0) {
                    if ((nextEntry = entryAt(currentTable,
                                             //--,所以一个段内的桶是从下往上遍历
                                             nextTableIndex--)) != null)
                        break;
                }
                //来到此处说明段内的桶已经全部遍历完了,但map中还有段没遍历
                else if (nextSegmentIndex >= 0) {
                    //--,所以map的段是从下往上遍历
                    Segment<K,V> seg = segmentAt(segments, nextSegmentIndex--);
                    if (seg != null && (currentTable = seg.table) != null)
                        nextTableIndex = currentTable.length - 1;
                }
                else
                    break;
            }
        }

        final HashEntry<K,V> nextEntry() {
            HashEntry<K,V> e = nextEntry;
            if (e == null)
                throw new NoSuchElementException();
            lastReturned = e; // cannot assign until after null check
            //如果next为空,则要向上寻找table遍历
            if ((nextEntry = e.next) == null)
                advance();
            return e;
        }

        public final boolean hasNext() { return nextEntry != null; }
        public final boolean hasMoreElements() { return nextEntry != null; }

        public final void remove() {
            if (lastReturned == null)
                throw new IllegalStateException();
            ConcurrentHashMap.this.remove(lastReturned.key);
            lastReturned = null;
        }
    }
```

##### ①遍历的大致流程

![](E:\Typora\resources\Java\数据结构\Map\ConcurrentHashMap1.7遍历.png)

##### ==②ConcurrentModificationException==

不同于`HashMap`,此处的`HashIterator`并没有使用过modCount,也即如果在遍历过程中修改了map,迭代器并不会返回`ConcurrentModificationException`

#### Ⅳ.`clear()`

```java
public void clear() {
    final Segment<K,V>[] segments = this.segments;
    //for循环逐个段清空数据
    for (int j = 0; j < segments.length; ++j) {
        Segment<K,V> s = segmentAt(segments, j);
        if (s != null)
            s.clear();
        →→{
              lock();
            try {
                HashEntry<K,V>[] tab = table;
                for (int i = 0; i < tab.length ; i++)
                    setEntryAt(tab, i, null);
                ++modCount;
                count = 0;
            } finally {
                unlock();
            }
        }
    }
}
```

此方法是用一个for循环遍历段,逐个清空段的数据。由于在清空时只是对所在段上锁，并没有对整个map上锁，所以调用完此方法后很有可能map里是有数据的

#### Ⅴ、==`size()`==

```java
  public int size() {
        final Segment<K,V>[] segments = this.segments;
        int size;
        boolean overflow; // 为true表示size溢出32位
        long sum;         // 所有segment中modCount的总和
        long last = 0L;  
        int retries = -1; // 第一次迭代不计入重试，因此总共会重试3次
        try {
            for (;;) {
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock(); // force creation
                }
                sum = 0L;
                size = 0;
                overflow = false;
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K,V> seg = segmentAt(segments, j);
                    if (seg != null) {
                        sum += seg.modCount;
                        int c = seg.count;
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }
```



==说大体思路==:

1. 首先不加锁循环所有的Segment，求所有`Segement.count`之和，同时计算所有Segment的modcount之和sum，如果sum与last相等，说明迭代期间没有发生其他线程修改ConcurrentHashMap的情况，返回size,如果不相等则重试
2. 当重试次数超过预定义的值（RETRIES_BEFORE_LOCK为2）时，对所有的Segment依次进行加锁**(全部加锁)**，再对`Segement.count`求和。

### 六、弱一致性

可以看到,`get()`和使用迭代器遍历是无锁操作,而都是使用volatile语义保证table和segments的可见性.当查找或遍历过程中其他线程修改了map，这项修改并不会反映并不会反映到查找或遍历返回结果上，所以`ConcurrentHashMap`是弱一致性

> 参考资料
>
> https://my.oschina.net/7001/blog/896587
>
> https://www.javadoop.com/post/hashmap
>
> https://blog.csdn.net/klordy_123/article/details/82933115