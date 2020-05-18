# HashMap

### 一、静态常量

```java
//默认的table数组的长度,若构造HashMap时没有指定容量,则会在第一次put()时进行resize(),使用此常量将table初始化为长度16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

//table容量最大值，值为(Integer.MAX_VALUE-1)/2
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * The load factor used when none specified in constructor.
 */
//默认的load_faactor
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//当进行桶中链表在队尾的插入操作后,判断是否要进行树化的其中一个阈值
static final int TREEIFY_THRESHOLD = 8;

//在扩容时对桶中树结点拆分成高低链表后,利用这个阈值判断这两个链表中的树结点是否有必要链化
//在删除桶中树结点后,判断是否需要对桶进行链化的阈值
//不选择<8时链化是为了给HahshMap一个缓冲的阶段
static final int UNTREEIFY_THRESHOLD = 6;

//当进行桶中链表在队尾的插入操作后,判断是否要进行树化的另一个阈值,table.length大于64才允许树化;小于则选择扩容
static final int MIN_TREEIFY_CAPACITY = 64;
```

### 二、成员变量

```java
//存放桶的数组
transient Node<K,V>[] table;

//提供一个可以遍历所有节点的接口
transient Set<Map.Entry<K,V>> entrySet;

//table中结点个数
transient int size;

//Iterator相关,其中对HashMap的修改操作会更改modCount.若正在用Iterator遍历HashMap的Set时修改了HashMap,迭代器的next()方法将会抛出ConcurrenModificationException
transient int modCount;

/**
 * The next size value at which to resize (capacity * load factor).
 *
 * @serial
 */
//当table中含有的结点超过threshold,则需要进行resize
//threshold并不一定就由capacity * load factor决定,也有可能等于旧threshold*2
int threshold;

//控制扩容阈值的参数,loadFactor越大,桶内碰撞率越高;loadFactror越小,浪费table空间越多
final float loadFactor;
```

### 三、构造方法

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    //threshold的第一次赋值存放的是初始容量,而tableSizeFor()则是找最接近initialCapacity的 2的n次幂
    this.threshold = tableSizeFor(initialCapacity);
}
```

`HashMap`并不在构造函数中初始化,而是将这项工作交给第一次调用的`put()`方法

### 四、查找

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    //根据对key的hash值取余的结果定位数组中桶的位置，判断桶中是否有结点，有才继续执行
    if ((tab = table) != null && (n = tab.length) > 0 &&
       //⭐这种位运算求余的方法只能用在除数是2的n次幂时才能使用，即x%(2^n)=x&((2^n)-1)=((2^n)-1)&x
        //⭐这种位运算的速度较之普通的求余较快,所以这也是为什么table.length永远是2的n次幂;其次在扩容时需要对桶中的结点分组,若table容量不是2的n次幂,那这个算法不可能实现
        (first = tab[(n - 1) & hash]) != null) {
        //判断桶中的首结点的key是否为目标key，若是则此节点为目标结点
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //如果首结点不是目标结点，则要遍历红黑树或链表进行查找
        if ((e = first.next) != null) {
            //若桶中结点的结构是红黑树
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //若桶中结点的结构是链表
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

#### ⭐1、hash

下面来看hash值的计算方法:

```java
static final int hash(Object key) {
        int h;
    	//调用由Object实现的方法得到hashcode,并且将该hashcode异或 hashcode无符号右移16位的数字(即hashcode高16位), 以当作HashMap的哈希值,这个异或的操作则称为扰动函数
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

![扰动函数](E:\Typora\resources\Java\数据结构\Map\扰动函数.png) 其中Object实现的`hashcode()`保证:

1. 两个相同的对象hashcode一定相同
2. hashcode相同的两个对象不一定相同

==`hash()`需要扰动函数的原因:==**要知道,key的hash值最终是要和table.length-1做 "&运算" 求余的。当HashMap刚初始化时table容量较小，如16-1的二进制位为`0000 0111`,此时和如8位hashcode `1001 0111`做与运算,可以看到,hashcode前面5位无论怎样具有特异性,做完与运算始终会等于0,这就相当于只有hashcode后面4位参与了映射,大大增加了桶碰撞几率(即对低16位相同高16位不同的hashcode不公平)。所以要增加扰动函数，使得高16位也发挥它的作用，让分布更加的随机。**

### 五、遍历

HashMap有了三种可遍历的对象：`EntrySet`、`ValueSet`、`KeySet`,三种对象分别由有各自的迭代器实现,但都继承自`HashIterator`(即本质上都是对table进行遍历,只不过在各自实现中取出各自所需的数据)

```java
abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot

    HashIterator() {
        expectedModCount = modCount;
        Node<K,V>[] t = table;
        current = next = null;
        index = 0;
        if (t != null && size > 0) { // advance to first entry
            //当table含有的node数量不为0时,找到第一个node,将其赋值给next
            do {} while (index < t.length && (next = t[index++]) == null);
        }
    }

    public final boolean hasNext() {
        return next != null;
    }

    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
		//⭐快速失败
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        //⭐TreeNode继承自Node，所以TreeNode保留了它作为Node时的next指针。所以对于迭代器来说，对红黑树的遍历其实也是对链表的遍历
       
        //这一步的作用为找到下一个不为null的node
        //先寻找当前桶的当前节点.next是否有结点,若无则来到下一个桶继续寻找结点
        if ((next = (current = e).next) == null && (t = table) != null) {
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        //返回的是原next
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        removeNode(hash(key), key, null, false, false);
        expectedModCount = modCount;
    }
}
```

#### 1、遍历大致流程![](E:\Typora\resources\Java\数据结构\Map\遍历.png)

### 六、插入

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //若table未初始化，则调用resize()对table进行扩容初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //若key的hash相对应的桶中还未有结点,则在该桶中新建一个结点作为当前结点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    //若桶中已有结点了,则要根据具体的结构进行结点链接
    else {
        Node<K,V> e; K k;
        //如果头节点hash与当前hash相同,且头节点key与当前key也相同,则说明此次是更新操作
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //接下来就要开始遍历链表或红黑树,查找数据结构内是否有与当前key相同的node,若有则对其进行更新操作(onlyIfAbsent=false的情况下);若无则直接将新节点加入进数据结构中
        
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
       //遍历链表
        else {
            for (int binCount = 0; ; ++binCount) {
                //当遍历到链表尾都未找到相同key,则新建节点
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //若当前桶中结点>=8,且table长度大于等于64,则进行树化操作
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //找到key相同的结点则跳出循环,让后面操作修改此结点的value(onlyIfAbsent=false的情况下)
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //若在链表或红黑树结点中找到了相同key
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            //⭐注意看这个判断语句,若原结点值未null,不管onlyIfAbsent参数如何设置,如果此时结点值为null则一定会对结点值进行更新
            if (!onlyIfAbsent || oldValue == null)
                //更新结点值
                e.value = value;
            //LinkedHashMap实现的方法,将结点加入到LinkedHashMap的双端链表尾
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //⭐重点注意:结点更新操作并不会修改modCount值
    ++modCount;
    //⭐在进行了结点的 插入 操作之后,才是真正意义上的扩容
    if (++size > threshold)
        resize();
     //LinkedHashMap实现的方法
    afterNodeInsertion(evict);
    return null;
}
```

#### 1、插入大致流程（注意树化和扩容的目标和顺序）

1. 若table还未初始化,则先调用`resize()`进行table初始化
2. 遍历桶中的红黑树或链表,若发现有相同key的结点,则对此节点进行更新操作;若没有发现,==则新建Node加入链表尾并判断是否要对当前桶做树化操作==
3. 判断table中的结点含量是否大于threshold,是则需要进行扩容

#### 2、扩容

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //此判断语句为真则说明table已经初始化
    if (oldCap > 0) {
        //若旧容量本已超过容量最大值则不再进行扩容(最大容量:1 << 30=(int最大表示范围-1)/2,所以以这个为最大容量则是为了控制容量是2的n次幂且不超过int最大表示范围)
        if (oldCap >= MAXIMUM_CAPACITY) {
            //不进行扩容,仅增大阈值防止其扩容
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //⭐阈值和table长度翻倍
        //⭐注意:只要翻倍后的容量比最大容量小且旧容量大于等于16时,新阈值都是翻倍而不是使用newCap*load_factor。所以可以看出,正常情况下大多数扩容都没有使用到load_factor.(⭐如果load_factor是0.75,其实翻倍和newCap*load_factor得出的结果是一样的..)
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            //有可能会溢出导致归0
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
       //⭐此时是oldCap=0&&olcThr>0的情况,即是使用两参构造器构造HashMap的,此时则以用tableSizeFor(initialCapacity)得到的[大于或等于 initialCapacity 的最小2的n次幂](即初始threshold)作为初始化table的.⭐所以在双参构造器中对threshold的特殊操作(与官方注释不符)是为了在table初始化时将threshold作为initialCapacity来使用,仅仅起到暂时存放的功能而已
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        //使用默认构造器初始化的HashMap,按照HashMap默认参数进行初始化
        newCap = DEFAULT_INITIAL_CAPACITY;
        //此时的threshold则等于load_factor*table.length
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //处理newThr溢出归零或双参构造初始化的threshold的赋值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    	 	//java不支持泛型数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //如果原table有结点,则要重新将这些结点放入新table的桶中
    if (oldTab != null) {
        //遍历table数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 重新映射时，需要对红黑树进行拆分,大致原理与链表分组相同
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    
                    //⭐链表结点重新映射桶的算法
                    //遍历桶中的链表,将链表中的结点根据相应条件分成low和high两组
                    do {
                        next = e.next;
                        
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //⭐直接将low链表与high链表放入相应的新table的桶中.分组后的链表中(指low或high链表)结点的相对位置并不会改变
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

##### ==①⭐扩容流程==

1. 判断table是否已经初始化,若已经初始化且扩容后容量未超过最大容量,则将容量翻倍;==其次,若旧容量大于16,则也将扩容阈值`threshold`翻倍==
2. table未初始化,若HashMap是双参构造器构造的,则根据参数指定的`initialCapacity`确定容量,其次根据参数指定的`load_factor*initialCapacity`确定扩容阈值;若根据无参构造器构造,则依据默认`default_load_factor*default_initialCapacity`确认扩容阈值,依据`default_initialCapacity`确认容量
3. 重新映射桶算法的执行

##### ==②⭐链表结点重新映射桶的算法==

注：下面的图只是分析为什么可以直接用`hash&oldCap`将结点分成两条链表,而不是真正的重新映射过程

![](E:\Typora\resources\Java\数据结构\Map\重新映射.png)

**由于扩容是扩容为olcCap的两倍，所以扩容后求余时的与运算会比扩容前求余的与运算多算hash一位。所以只是多了一位二进制位(0或1)要与运算的，并不需要把已经 与运算 过的低位再 与运算 一次,因为已经 与运算 过的低位的十进制值已经在旧table的桶索引中体现了出来.所以这里只需要将`hash&oldCap`得到需要多算的那一位二进制位是1或是0(1则放入high链表,0则放入low链表),若是0,则0+旧table中的桶索引(已经运算过的低位)=新桶中的索引,即不变;若是1,则oldCap+旧table中的的索引=新桶中的索引**

##### ③红黑树结点重新映射桶的算法----拆分

```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    //要进行resize的桶的头节点
    TreeNode<K,V> b = this;
    // 同样是将红黑树节点分成两组
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        //与链表重新映射的做法差不多
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }
	
    if (loHead != null) {
        //当low链表树结点个数小于6，则需要链化
        if (lc <= UNTREEIFY_THRESHOLD)
            tab[index] = loHead.untreeify(map);
        else {
            //分组后的树结点个数仍然大于8
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    //与low链表的操作大致相同
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```

**因为TreeNode保留了`next`指针,所以同样是用分组的方式将旧table的桶中结拆分成高低两部分。与链表分组算法的不同之处在于，TreeNode分完组之后，要根据 链化阈值 判断高低两部分链表是进行链化还是重新树化**

**==这里注意一个问题：若是在旧table中不是同一个桶的结点，`resize()`后也肯定不会在同一个桶中==**

### 七、树化、链化

#### 1、树化

树化操作只会在`putVal()`时触发

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //⭐当table.length>64时才允许对桶树化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    //下面的do··while循环将桶里面的Node转化为TreeNode,保留了Node的next指针并增加了一个prev指针,使得桶的结构是红黑树同时也是个双端链表
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
       
        if ((tab[index] = hd) != null)
            //开始构造红黑树
            //红黑树太难,且类似二叉搜索树,就不贴代码直接写流程了
            hd.treeify(tab);
    }
}
```

##### ①树化流程

1. 若此时table的容量未到64，则选择对table扩容而不是对桶树化
2. 将Node转换为TreeNode，并对prev和next指针做好引用
3. 将TreeNode双端链表构造成红黑树
   1. 根据结点的hash值作为比较手段
   2. 若hash值相同，且key对象实现了Comparable，则使用key对象的`compareTo()`作为比较手段
   3. 使用HashMap内置的`tieBreakOrder()`方法进行比较

#### 2、链化

链化操作会在删除树结点和拆分时触发。由于TreeNode保留着链表结构，所以`untreefiy()`直接将TreeNode替换成Node,链化操作就完成了

### 八、快速失败与安全失败

#### 1、快速失败

如在`HashMap`利用其`Iterator`遍历的过程中,如果有线程调用`map.put()`或`map.remove()`使得`HashMap`的结构改变(值改变不影响,上面源码分析时已经说过了),此时`Iterator.next()`方法就会抛出`ConcurrentModificationException`异常。这就是快速失败。

#### 2、安全失败

在java.util.concurrent下的类，都是线程安全的类，他们在迭代的过程中，如果有线程进行结构的改变，不会报异常，而是正常遍历，这就是安全失败。如1.7和1.8的`ConcurrentHashMap`在迭代时就不会检查`modCount`的值,而是正常遍历;又如`CopyOnWriteArrayList`,任何的写操作都会先将`list`做一个浅拷贝,然后对这个拷贝做修改操作,最后才将这个拷贝赋值回`volatile`修饰的array中,而其遍历操作就是对该`list`的快照。所以遍历操作和修改操作互不影响，就算结构改变也会正常遍历。而这同时也造成了其弱一致性。