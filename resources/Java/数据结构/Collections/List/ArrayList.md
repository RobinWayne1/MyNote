# ArrayList

```java
//默认容量,使用无参构造器创建对象时使用
private static final int DEFAULT_CAPACITY = 10;

//当构造器中指定容量为0时,则会将此数组赋值给elementData
private static final Object[] EMPTY_ELEMENTDATA = {};

//使用无参构造器创建对象时,将此数组赋值给elemenData(注意:这和上面那个成员变量不是同一个实例,即使用=比较为false,因为{}相当于new Object[];JVM仍然是创建了两个数组实例)
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

//存放元素的数组
transient Object[] elementData; // non-private to simplify nested class access

//数组中含有的元素的量,非elementData.length
private int size;
```

```java
public boolean add(E e) {
    	    //size为ArrayList中含有的元素个数，并不是底层数组的容量 
    		/*calculateCapacity()的作用为当第一次add元素且构造器没有指定容量时，返回DEFAULT_CAPACITY(10)(注此时elementData.length=0),此举将使得ensureExplicitCapacity()将数组扩容至DEFAULT_CAPACITY(10)
    		 */
        	ensureExplicitCapacity(calculateCapacity(elementData, size+1));
    	→→→{
            	//事关迭代器的快速失败,将在Iterator详解中介绍
           		modCount++;       
        		if (minCapacity - elementData.length > 0)
            		//扩容
                    grow(minCapacity);
           }
    
        elementData[size++] = e;
        return true;
    }
```

```java
//参数minCapacity这样叫的原因应该是扩容后数组的长度至少为minCapacity
private void grow(int minCapacity) {
    
    int oldCapacity = elementData.length;
    //新容量=旧容量+旧容量/2,即1.5倍旧容量
    //>>为有符号右移,即若正数右移则左边补0,负数左边补1(计算机表示方式是补码,正数补码与原码相同,负数补码为原码取反加1)
    //>>>为无符号右移,即无论正负数,左边都补0
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //若新容量仍然比minCapacity要小,则直接把minCapaciy当作新容量(调用addAll()时就可能1.5倍扩容不足的情况)
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    //将数组的容量限制在Integer.MaxValue-8
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    //进行数组拷贝
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

```java
public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
    	//比较Class对象的地址是否相同,父类和子类并不是同一个Class对象
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    	//
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
//意义是从src数组第srcPos位元素后的length个元素复制到dest数组中第destPos位后
public static void arraycopy(Object src,
                                   int srcPos,
                                   Object dest,
                                   int destPos,
                                   int length)
```

总结:`add()`方法首先确定好添加元素后的ArrayList含有的元素量,之后对比元素量和容量查看是否要扩容。若要进行扩容，则扩容后的新容量为旧容量的1.5倍，==同时还要处理1.5倍容量仍然不足和最大容量限制两个问题==。之后则使用`System.arraycopy()`方法移动数组元素,再将元素放入数组中.

```java
//注:这里笔记写的ArrayList是让你将其想象成list而不是数组,list没有限制长度
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);
	//即在ArrayList中,要删除的元素后还有几个元素(⭐注意这里用的是size,不是eleData.length,所以还未使用到的数组空间不需要复制)
    int numMoved = size - index - 1;
    //若删除的元素不是ArrayList中的最后一位
    if (numMoved > 0)
        //进行数组复制
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```