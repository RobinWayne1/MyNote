# 浅析BitSet原理

都知道布隆过滤器，布隆过滤器就是通过操作bit数组来标识某个key是否存在的。而对于Java来说，操作bit数组的api就是`BitSet`。

## 一、成员变量

```java
//long型数组,long数值的每个二进制位就用于代表一个bit。该bit为1就代表true,0就代表false。由于long型用8个字节表示，所以一个long型能代表64个bit
private long[] words;
//相当于ArrayList的size，表示words数组真正使用的长度。也就是说words是会预先扩容的
private transient int wordsInUse = 0;
//用于将调用方指定的bitset索引位 转换成word数组索引位,简单来说就是查找表示这个bitset位的long数值放在数组的哪个下标中
private final static int ADDRESS_BITS_PER_WORD = 6;
```

## 二、构造函数

```java
public BitSet(int nbits) {
    // nbits can't be negative; size 0 is OK
    if (nbits < 0)
        throw new NegativeArraySizeException("nbits < 0: " + nbits);

    initWords(nbits);
    sizeIsSticky = true;
}
private void initWords(int nbits) {
    words = new long[wordIndex(nbits-1) + 1];
}

```

上面的调用栈其实就是在初始化`words`数组,比较重要的逻辑在下面这个方法的逻辑。

```java
private static int wordIndex(int bitIndex) {
    return bitIndex >> ADDRESS_BITS_PER_WORD;
}
```

该方法主要就是找出bitset索引位对应的word数组索引位。`bitIndex >> ADDRESS_BITS_PER_WORD`就相当于`bitIndex / 64`,如`65/64`就得出了65这个bitset索引位应该放在`word[1]`中。

## 三、`set()`

```java
public void set(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);
	//获取bitIndex对应的word数组索引位
    int wordIndex = wordIndex(bitIndex);
    //扩容逻辑
    expandTo(wordIndex);
	
    words[wordIndex] |= (1L << bitIndex); // Restores invariants

    checkInvariants();
}
```

比较关键的就是变更bitIndex的这一行代码。其实就是将1左移相应的索引位数，然后用或将long数值的二进制位赋值成1。

> 有一个细节。有没有发现源码没有对 bitIndex与64取模就直接左移了？按照正常逻辑当出现`1<<65`这种场景时不就会移除了?那是因为java的左移操作会对bitIndex自动取模的,也就是`1<<65`会自动变成`1<<1`

源码简单说完了,下面开始讲业务。

http://service-center.vip.vip.com/documentCenter/detail/osp-sales-sink.api.vip.com/com.vip.somp.salessink.backend.service.SalesVisibleService?type=method&name=batchGetMerchandiseArea

这个接口所返回的`areaIds`里的6个long其实就是通过`BitSet`转出来的,所以这个返回值直接调用`BitSet.valueOf()`去用`BitSet`处理最方便。但是我在编码时先使用了位运算去处理，所以将规则记录在这里以方便之后想要对禁售区域做位运算的开发。

由于是`BitSet`转出来的long数组,所以他的规则必定是与`BitSet`底层一致的。通过上面的`set()`源码可以看见,在计算word数组索引位时,随着bitset索引位的增加word数组索引位也就从索引0相应的增加。但是在修改long数值内的二进制位时，则是从该long数值的最右的二进制位作为第0位索引的（因为使用的是左移操作）。所以369个城市在long数组的表示就是：[63\~0,127\~64,255~128,······]。

> 参考资料
>
> https://www.cnblogs.com/lqminn/archive/2012/08/30/2664122.html