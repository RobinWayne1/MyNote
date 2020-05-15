# ConcurrentHashMap

## 一、成员变量

```java
/*
 *1、当值为-1时，代表table正在初始化或扩容.用于并发初始化或扩容table时 线程争抢初始化成功 的标志.
 *2、当值为-n时，代表有n个线程正在激活扩容的线程
 *3、当table还为null时，值有可能为正数(若在构造函数中指定initCapacity,则sizeCtl的值为的最小2次幂)或0，代表initTable()中table的初始容量
 *4、当table刚初始化完,值为0.75*table容量,即代表threshold
*/
private transient volatile int sizeCtl;

//标识扩容转移时下一个协助转移的线程 转移 的桶数组起点(起点的值比终点边界的大,即从后往前算)
private transient volatile int transferIndex;
//在转移过程中，存放扩容后的目标桶数组，目的是让下一个协助转移的线程继续往该桶数组转移结点
private transient volatile Node<K,V>[] nextTable;
// 表示该桶正在数据转移
static final int MOVED     = -1; 
 // 表示该桶已经转换成树
static final int TREEBIN   = -2;
```

## 二、插入

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //键值都不允许null
    if (key == null || value == null) throw new NullPointerException();
    //计算hash值
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //table为null则要进行初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //使用与HashMap同样的方式求余
        //⭐⭐⭐⭐⭐tabAt()方法底层使用Unsafe.getObjectVolatile()获取volatile数组中的元素。由此引出一个重要知识点:volatile数组并不保证元素的可见性,只保证数组引用的可见性.查Blog看到有对CopyOnWriteArrayList和ConcurrentHashMap做法不同的疑问,这里我想说的是,COWAL是修改副本数组后将其覆盖到原数组引用上,而这种做法是满足volatile语义的
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //⭐当桶中没有结点,则用cas操作为桶新建一个当前kv结点
            //返回false则说明出现并发,则继续循环;返回true就可以拉到最后面了
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //此时桶数组位置i正在扩容.⭐可以看到,如果发现扩容还未轮到当前插入的位置i,那还是能够成功插入的.
        else if ((fh = f.hash) == MOVED)
            // 帮助数据迁移,在位置i扩容时位置i的put()操作显然是无法进行的,因为会把位置i的头结点锁住,所以此时这条线程不如去帮忙数据迁移而不是去到下面的else分支阻塞
            tab = helpTransfer(tab, f);
        //来到这里说明桶中已有结点,则要开始遍历桶中的结点了
        else {
            V oldVal = null;
            synchronized (f) {
                //检查现在桶中的头结点还是不是f,如果已经被并发修改了头结点,此时就要重新循环从而争抢新头结点的monitor
                if (tabAt(tab, i) == f) {
                    //fh>=0说明这个桶的结构是链表而不是红黑树
                    if (fh >= 0) {
                        //用于统计结点个数,方便最后检测是否树化
                        binCount = 1;
                        //老套路,遍历链表查找是否有相同key,有则根据onlyIfAbsent替换其value
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            //当遍历到链表尾也没有发现相同key,为这个k-v创建新结点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //如果桶结构是红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        //调用红黑树的插入方法
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                //如果链表长度大于8,树化(树化这一块的参数和hashmap一样)
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    //返回替换的旧值
                    return oldVal;
                break;
            }
        }
    }
    //增加结点个数
    //⭐在这里会判断现在的桶数组的结点数是否已经大于阈值,如是则要扩容
    addCount(1L, binCount);
    return null;
}
```

### 1、hash

```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

采用了HashTable与HashMap的做法的结合，其中HASH_BITS为32位1，用来转换负数

### 2、第一次put时初始化table（`initTable()`）

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        //有其他线程正在初始化table
        if ((sc = sizeCtl) < 0)
            //让出CPU,将线程状态由运行改为就绪,若重新抢占CPU后其他线程还没有初始化完table,将会继续自旋
            Thread.yield(); // lost initialization race; just spin
        //当前线程竞争成功,用CAS操作更改SizeCtl标志为-1
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    //使用sizeCtl的值作为初始容量或以16作为初始容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    //创建桶数组
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    //修改sizeCtl为0.75*cap,即threshold
                    sc = n - (n >>> 2);
                }
            } finally {
                //修改回原值,所以try才需要if语句判断是否已经初始化完成,如果初始化完成就说明不再需要初始化,则要将sizeCtl改回原来的值
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

总的来说，`initTable()`的过程就是使用CAS修改`sizeCtl`的操作,保证初始化table的线程只有一个,并且在初始化完后以`sizeCtl`存放扩容阈值

### 3、树化操作（`treeifyBin()`）

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        // MIN_TREEIFY_CAPACITY 为 64
        // 所以，如果数组长度小于 64 的时候，其实也就是 32 或者 16 或者更小的时候，会进行数组扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            // 扩容
            tryPresize(n << 1);
        // b 是头结点
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 对头结点加锁
            synchronized (b) {

                if (tabAt(tab, index) == b) {
                    // 下面就是遍历链表，建立一颗红黑树
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    // 将红黑树设置到数组相应位置中
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

### 4、扩容

#### ①、调节多线程进行数据迁移（` tryPresize();`）

```java
private final void tryPresize(int size) {
       
             /* MAXIMUM_CAPACITY = 1 << 30
             * 如果给定的size大小大于等于数组容量的一半，则直接使用最大容量，
             * 否则使用tableSizeFor算出来
             */
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        //如果数组table还没有被初始化，则初始化一个大小为sizeCtrl和刚刚算出来的c中较大的一个大小的数组
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            int rs = resizeStamp(n);
            //sc<0代表有线程正在扩容Table(即标识转移的线程个数)，则当前线程帮助扩容
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //2、⭐每有一个线程协助转移,则会将sizeCtl的值+1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    //nextTable指的是转移的目标数组，此时该数组已经被第一个扩容的线程初始化好了，所以当前线程继续以该桶数组为目标转移结点
                    transfer(tab, nt);
            }
            //没有其他线程在扩容,则由当前线程承担 初始化扩容 这一任务
            //1、扩容时第一个开始扩容的线程总会来到这里
            else if (U.compareAndSwapInt(this, SIZECTaL, sc,
                           //⭐将sizeCtl设置成一个比较大的负数,该数用来标志转移的线程数(用法要看transfer())
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```

#### ②、结点迁移（`transfer`）

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //计算每个线程每次迁移任务应该负责多少桶数组
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
//-------------------------------初始化新桶数组---------------------------------------------
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
//-----------------------限定当前线程转移的边界或处理当前线程下一个负责转移的桶数组下标位置---------
    int nextn = nextTab.length;
    //⭐放在桶数组中位置i的头部，标识桶数组位置i已经有线程正在处理了
    //⭐该结点的hash值就是MOVED
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    //当advance=true,则说明当前旧桶数组的位置i结点转移已经完毕
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    //i表示扩容的数组下标起点,bound标识扩容的数组下标终点
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //当位置i结点转移已完毕
        while (advance) {
            int nextIndex, nextBound;
            //--i,代表当前线程下一个负责转移的位置,如果i已经<bound,则说明这条线程的迁移工作已经完成了
            if (--i >= bound || finishing)
                //让当前线程继续转移工作
                advance = false;
             // 将 transferIndex 值赋给 nextIndex,一开始transferIndex值等于桶数组长度
            // 这里 transferIndex 一旦小于等于 0，说明原数组的所有位置都有相应的线程去处理了
            else if ((nextIndex = transferIndex) <= 0) {
                //所有位置都有线程处理,不需要当前线程做转移工作了
                i = -1;
                advance = false;
            }
            //⭐当一条线程第一次来做迁移工作时 或 已经做完了一次迁移工作后,则会来到这里,先设置好迁移的起点和终点。已经做完一次的线程如果发现transferIndex还没有小于0，则意味着该线程还要继续做迁移工作
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                    //这个三元判断的意思是:如果当前起点的桶数组下标大于步长(默认每个CPU处理16个桶)
                    //则将transferIndex值CAS为起点减去步长的值,该值将作为下一个 协助扩容线程 的起点
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                //transferIndex的值是下一个 协助扩容线程的 起点,那也就是当前扩容线程的终点 bound
                bound = nextBound;
                //当前扩容线程的起点
                i = nextIndex - 1;
                advance = false;
            }
        }
//-------------------------判断整个map的迁移过程是否结束--------------------------------------
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                // 重新计算 sizeCtl：n 是原数组长度，所以 sizeCtl 得出的值将是新数组长度的 0.75 倍
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 之前我们说过，sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2,是一个负数
            // 然后，每有一个线程参与迁移就会将 sizeCtl 加 1，
            // 这里使用 CAS 操作对 sizeCtl 进行减 1，代表做完了属于自己的任务
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //这里判断的是此时的sizeCtl是否等于原sizeCtl，若等于则说明已经没有线程在扩容了(⭐如果只有一条线程扩容,这条线程则会一直死循环负责整个桶数组的数据迁移直到迁移完成[看上面else if分支],所以该线程在未迁移完成前绝不会来到这个分支将sizeCtl-1.所以一旦sizeCtl等于原值,则说明桶数组已迁移完毕)，设置finishing为true代表扩容已结束，在下一次循环就会来到上面的finish判断分支
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                //结束了
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
//---------------------------------------------------------------------------------------
        //数组中把null的元素设置为ForwardingNode节点(hash值为MOVED[-1])
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        //如果头结点hash为MOVED，说明该位置已经有被处理过了
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
      
//-----------------------------------------数据迁移----------------------------------------  
        else {
            //正式开始数据迁移
            synchronized (f) {
                //检查头结点是否还是那个结点，若不是则说明在此过程中有操作已经将该桶数组位置的头结点改变了，重新循环
                if (tabAt(tab, i) == f) {
                    //lowNode
                    //highNode
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                       //⭐不要看和CHP1.7都用到了lastRun,实际上这两个的实现是不太一样的
                        //注意看这个runBit和下面的b,都是用hash&oldCap,这实际上是1.8HashMap的将旧桶数组链表分为高低两条链表的思路;而CHP1.7的思路是遍历链表用hash&newCap-1找到最前的一个结点,该结点后的结点都映射到同一个新桶中,将这个结点直接放入新桶,然后再往前逐个用hash&newCap-1进行重新映射结点
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        //找到最前的一个lastRun结点,该结点后的结点都会映射到同一个新桶
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        //若runBit=0,则说明lastRun后的结点属于 低链表结点
                        if (runBit == 0) {
                            
                            ln = lastRun;
                            
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        //⭐下面这个循环就开始将lastRun结点前的结点分离成高链表和地链表
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                //将结点插入链表头
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //设置低链表 属于 的 新桶数组位置
                        setTabAt(nextTab, i, ln);
                        //设置高链表 属于 的 新桶数组位置
                        setTabAt(nextTab, i + n, hn);
                        //将旧桶数组位置i的头结点设置为fwd表示该桶已转移完成
                        setTabAt(tab, i, fwd);
                        //设为true表示当前位置已转移完毕,下个循环将转移下一个位置
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 如果一分为二后，节点数少于 6，那么将红黑树转换回链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

#### ③、扩容小结

##### Ⅰ、扩容触发机制

当`putVal()`方法的最后 发现插入了元素后,链表的长度大于8但是桶数组容量小于64  或在`addCount()`方法中发现结点个数已经大于阈值时,就开始扩容。

##### Ⅱ、扩容流程（此处以树化中的扩容为例子）

1. **初始化过程的争抢**:方法执行到`tryPresize()`时,若发现`sizeCtl`仍是正数(代表阈值),那就说明此时并没有线程在扩容,这条线程就会尝试CAS `sizeCtl` 成一个较大的负数以表示正在扩容的线程数。如果修改成功，那么这条线程则要负责初始化新桶数组；若修改失败则说明初始化工作被抢，那么就不断循环检查`nextTable`直到不为空,则以该桶数组为新桶数组协助数据迁移
2. **新桶数组初始化**：方法执行到`transfer()`,若当前线程是进行新桶初始化的线程,则先新建一个新桶数组,容量为旧桶的2倍。
3. **限定转移的边界**：在初始化之后则开始限定转移的边界。每条线程都负责一部分旧数组下标的结点迁移(通过CPU个数算出)，我将其称作一个迁移任务n。`transferIndex`成员就是用来记录每条线程负责迁移的数组下标起点,若  compare `transferIndex` and swap`transferIndex-n`成功,此时就说明由`transferIndex` 到`transferIndex-n`的数组下标的迁移工作属于当前线程了,可以开始数据迁移了.
4. **数据迁移**:若桶的结构是链表,则迁移的过程是`HashMap1.8`与`ConcurrentHashMap1.7`的结合。依旧是分成高位链表和低位链表。首先先遍历一遍该位置的链表，找到`lastRun`结点(该结点后的结点都属于高位链表或低位链表,即都会重新映射到同一个新桶中),将该结点直接赋值给高位链表或低位链表.随后则再遍历一次链表,将高位结点加入高位链表,将低位结点加入低位链表,随后将这两条链表赋值给新的桶数组。至此完成一个旧数组下标的数据迁移。
5. **判断是否该线程的迁移任务已完成**：回到限定转移边界的循环体中，根据当前需要转移的数组下标`--i`和边界`bound`判断这条线程的转移任务已完成，若未完成`(i>=bound)`则继续转移`tab[i]`的结点;若已完成`(i<bound)`,则要判断整个map的数据迁移任务是否已完成
6. **判断整个map的数据迁移任务是否已完成**:此时就是用`sizeCtl`代表的 正在迁移线程数 来判断是否已结束(当前该值是否等于原值)。若未结束，则重新来到第3步确定转移的边界；若已经结束了，则返回。

### 5、插入小结

1. 根据hash求余算出桶的位置。若发现桶数组为空，则先进行初始化。
2. 若发现桶为空，则用CAS直接插入新结点，该结点为头结点
3. 若发现桶此时正在进行数据迁移，则帮助其一起数据迁移
4. 来到这里就要遍历桶链表或桶红黑树了。获取桶中头结点的`monitor`,随后遍历该桶,若找到相同key则替换;若没有找到,则在链表尾添加结点;红黑树类似.
5. 添加完结点后,判断该桶是否需要进行树化.
6. 最后判断整个桶数组的结点个数是否已经大于阈值`sizeCtl`,若是则要扩容

## 三、查找

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        //求出目标桶的头结点
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            //如果头结点就是目标节点，返回桶的头结点
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        //hash<0说明此桶在扩容或者该桶的结构是红黑树,则调用ForwordNode.find()或TreeBin.Node.find()方法得到结点value
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        //来到这里说明桶结构是链表,遍历查找即可
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

### 1、正在扩容的桶的查找(`find()`)

```java
 Node<K,V> find(int h, Object k) {
        
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            //遍历新桶
            for (;;) {
                int eh; K ek;
                //遍历发现桶中的某结点是目标结点,则返回
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                    //⭐没看懂为什么新桶数组也会出现ForwardingNode
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        //红黑树结点,调用它的find()方法
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```

### 2、查找小结

1. 计算 hash 值

2. 根据 hash 值找到数组对应位置: (n - 1) & h

3. 根据该位置处结点性质进行相应查找

   1. 如果该位置为 null，那么直接返回 null 就可以了

   2. 如果该位置处的头结点刚好就是我们需要的，返回该头结点的值即可

   3. 如果该位置节点的 hash 值小于 0，说明正在扩容，或者是红黑树，后面我们再介绍 find 方法

   4. 如果以上 3 条都不满足，那就是链表，进行遍历比对即可

## 四、ConcurrentHashMap的同步机制

### 1、读操作

　　首先是读操作，从源码中可以看出来，在get操作中，根本没有使用同步机制，也没有使用unsafe方法，所以读操作是支持并发操作的。

### 2、写操作

　　分析这个之前，先看看什么情况下会引起数组的扩容，扩容是通过transfer方法来进行的。而调用transfer方法的只有`trePresize`、`helpTransfer`和`addCount`三个方法。

这三个方法又是分别在什么情况下进行调用的呢？

* tryPresize是在treeIfybin和putAll方法中调用，treeIfybin主要是在put添加元素完之后，判断该数组节点相关元素是不是已经超过8个的时候，如果超过则会调用这个方法来扩容数组或者把链表转为树。

* helpTransfer是在当一个线程要对table中元素进行操作的时候，如果检测到节点的HASH值为MOVED的时候，就会调用helpTransfer方法，在helpTransfer中再调用transfer方法来帮助完成数组的扩容

* addCount是在当对数组进行操作，使得数组中存储的元素个数发生了变化的时候会调用的方法。　

**所以引起数组扩容的情况如下**：

* 只有在往map中添加元素的时候，在某一个节点的数目已经超过了8个，同时数组的长度又小于64的时候，才会触发数组的扩容。

* 当数组中元素达到了sizeCtl的数量的时候，则会调用transfer方法来进行扩容　　

**那么在扩容的时候，可以不可以对数组进行读写操作呢？**

　　事实上是可以的。当在进行数组扩容的时候，如果当前节点还没有被处理（也就是说还没有设置为fwd节点），那就可以进行设置操作。如果该节点已经被处理了，则当前线程也会加入到扩容的操作中去。

**==那么，多个线程又是如何同步处理的呢？==**

在ConcurrentHashMap中，同步处理主要是通过Synchronized和unsafe两种方式来完成的。

* 在修改sizeCtl，使用的都是unsafe的方法，来达到并发安全的目的

* 当需要在某个桶位置设置结点的时候，则会通过Synchronized的同步机制来锁定该位置的头结点。

* 在数组扩容的时候，则通过处理的步长和fwd节点来达到并发安全的目的，通过设置hash值为MOVED

* 当把某个位置的节点复制到扩张后的table的时候，也通过Synchronized的同步机制来保证线程安全

 

> 参考资料
>
> https://www.cnblogs.com/zerotomax/p/8687425.html
>
> https://javadoop.com/post/hashmap

