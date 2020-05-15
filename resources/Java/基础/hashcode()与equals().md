# ` hashcode()`与`equals()`

### 一、`hashcode()`方法的作用

1. hashcode可以作为对象的标志,能够方便如`HashMap`这样的数据结构在查找时确定`key`对象所在的桶位置
2. 如果两个`Object`调用`equals()`返回true,则这两个`Object`调用`hashcode()`一定会返回相同的值
3. 而若两个`Object`调用`hashcode()`返回相同的值,则虚拟机并不保证两个`Object`调用`equals()`会返回`true`,`hashcode`相同只能说明这两个对象将会被放入同一个桶中
4. `equals()`方法要与`hashcode()`一起重写,不能只重写其中某个方法

### 二、`Object`的`equals()`

`Object`的`equals()`方法就只是简单的对比两个对象的位置,也即比较两个引用变量是否指向同一块堆内存

### 三、为何这两个方法要一起重写

首先要明白重写的含义在哪里。**比如需求是对具有相同成员变量值进行去重存储**，此时重写的方法就应该为下面这种样子：

```java
public class ClassTest
{
    private Object member1;
    private Object member2;
    private Object member3;

//···忽略GetSetter
    @Override
    public int hashCode()
    {
        return Objects.hash(member1, member2, member3);
      →→→{//⭐实际上就是将各个成员的hashcode相加
            public static int hashCode(Object a[]) 
            {
        		if (a == null)
            		return 0;

        		int result = 1;

        		for (Object element : a)
                {
                    result = 31 * result + (element == null ? 0 : element.hashCode());
                }
        		return result;
    		}
    	 }
    }

    @Override
    public boolean equals(Object obj)
    {
        if (!(obj instanceof ClassTest))
        {
            return false;
        }
        if (this == obj)
        {
            return true;
        }
        final ClassTest c=(ClassTest)obj;
        if(!member1.equals(c.getMember1()))
        {
            return false;
        }
        if(!member2.equals(c.getMember2()))
        {
            return false;
        }
        if(!member3.equals(c.getMember3()))
        {
            return false;
        }
        return true;
    }
}

```

在这种需求下，如果只重写其中一个方法,此时`HashSet`的去重机制就会出错。

在`HashSet`的`add()`方法中调用的是`HashMap`的`put()`方法,将一个 **键为传入对象 值为常量的`k-v`**放入`HashMap`相应的桶中。

1. `HashMap`首先根据key的hash值（调用`hashcode()`求出）与容量求余找到相应的table下标。
2. 其次再将k-v结点放入桶中。然而，放入桶这个操作是会产生两种结果的：
   1. 发现桶中有结点的key与当前欲插入的key相同，则更新这个key的value。
   2. 桶中没有结点的key与当前欲插入的key相同，则插入当前的key

判断结点的key是否相同使用的是下面这段代码:

```java
if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
```

分析到这里应该就能看出,`HashSet`这个类必须是 要`hashcode()`与`equals()`正确配合才能保证其去重机制。

#### Ⅰ、重写了`hashcode()`而没有重写`equals()`

还是按照之前的 成员变量值相同则去重 为需求，假设此时已经有一个`object1`存放在`HashSet`中,此时用户想将一个成员变量 与`object1`相同 的`object2`加入`HashSet`。

由于重写了`hashcode()`，此时`HashMap`能够根据`object2`的hash值（调用`hashcode()`求出）找到与`object1`相同的桶位置,但是在遍历链表查重调用`key.equals(k)`时,由于没有重写`equals()`,此时就会调用`Object.equals()`使得无论如何都会返回false,这样就出现了错误使得`HashSet`中存在了两个成员变量值相同的对象。

#### Ⅱ、重写了`equals()`而没有重写`hashcode()`

此时由于`object1`与`object2`的hashcode不相同，使得放入的桶也不相同，没有出现哈希碰撞自然也就能成功插入该结点了，同样出现了两个成员变量值相同的对象。