SATB，snapshot-at-the-beginning，是维持并发GC的正确性的一个手段。**G1 GC的并发理论基础就是SATB，而**

**CMS则是“incremental update”**。

SATB抽象的说就是在一次GC开始的时候是活的对象就被认为是活的，此时的对象图形成一个逻辑“快照”（snapshot）；然后在GC过程中新分配的对象都当作是活的。其它不可到达的对象就是死的了。 

很容易知道哪些对象是一次GC开始之后新分配的：每个region记录着两个top-at-mark-start（TAMS）指针，分别为prevTAMS和nextTAMS。在TAMS以上的对象就是新分配的，因而被视为隐式marked。 

但是在并发GC里，collector一边动mutator也一边动，如果collector并发mark的过程中mutator覆盖了某些引用字段的值而collector还没mark到那里，那collector不就得不到完整的snapshot了么？ 

为了解决这个问题就有了SATB write barrier。G1 GC具体使用的是“湯浅”（Yuasa）式的SATB write barrier的变种。

Write barrier是对“对引用类型字段赋值”这个动作的环切，也就是说赋值的前后都在barrier覆盖的范畴内。在赋值前的部分的write barrier叫做pre-write barrier，在赋值后的则叫做post-write barrier。 

在HotSpot VM里，在引入G1 GC之前，其它GC都只用了post-write barrier，所以它在源码里没有特别的前后缀；而G1 GC特有的pre-write barrier则在源码里有_pre的后缀，可以留意一下。 

C代码 

```C
void oop_field_store(oop* field, oop value) {  

pre_write_barrier(field);  

*field = value; // the actual store  

post_write_barrier(field, value);  

}  
```

Pre/post-write barrier跟SATB有啥关系呢？ 

前面提到SATB要维持“在GC开始时活的对象”的状态这个逻辑snapshot。除了从root出发把整个对象图mark下来之外，其实只需要用pre-write barrier把每次引用关系变化时旧的引用值记下来就好了。这样，等concurrent marker到达某个对象时，这个对象的所有引用类型字段的变化全都有记录在案，就不会漏掉任何在snapshot里活的对象。当然，很可能有对象在snapshot中是活的，但随着并发GC的进行它可能本来已经死了，但SATB还是会让它活过这次GC。 

所以在G1 GC里，整个write barrier+oop_field_store是这样的： 

C代码 

```C
void oop_field_store(oop* field, oop new_value) {  

pre_write_barrier(field);             // pre-write barrier: for maintaining SATB invariant  

*field = new_value;                   // the actual store  

post_write_barrier(field, new_value); // post-write barrier: for tracking cross-region reference  

}  
```

按照湯浅式SATB barrier的设计，pre-write barrier里面的抽象逻辑应当如下： 

C++代码  

```C++
void pre_write_barrier(oop* field) {  

if ($gc_phase == GC_CONCURRENT_MARK) { // SATB invariant only maintained during concurrent marking  

   oop old_value = *field;  

    if (old_value != null && !is_marked(old_value)) {  

      mark_object(old_value);  

      $mark_stack->push(old_value); // scan all of old_value's fields later  

    }  

}  

}  
```

在每次引用关系发生变化时，旧的引用所指向的对象就会被mark上，其子孙也会被递归mark上，这样就不会漏mark任何对象，snapshot的完整性也就得到了保证。 

但实际去看G1的论文和代码，会发现它的pre-write barrier却是类似这样的： 

C++代码 

```c++
void pre_write_barrier(oop* field) {  

oop old_value = *field;  

if (old_value != null) {  

    if ($gc_phase == GC_CONCURRENT_MARK) { // SATB invariant only maintained during concurrent marking  

      $current_thread->satb_mark_queue->enqueue(old_value);  

    }  

}  

}  
```

这比原本的湯浅式设计少了些东西：没有检查目标对象是否已经mark，也不去对目标对象做mark和扫描它的字段。 

实际上该做的事情还是得做，只是不在这里做而已。后面讲到logging barrier的时候就会展开说明了。 

（Pre-write barrier的实际代码有好几个版本，其中最简单明白的版本是： 

C++代码  

```c++
// This notes that we don't need to access any BarrierSet data  

// structures, so this can be called from a static context.  

template <class T> static void write_ref_field_pre_static(T* field, oop newVal) {  

T heap_oop = oopDesc::load_heap_oop(field);  

if (!oopDesc::is_null(heap_oop)) {  

    enqueue(oopDesc::decode_heap_oop(heap_oop));  

}  

}  
```

enqueue动作的实际代码则在G1SATBCardTableModRefBS::enqueue(oop pre_val)。 

它判断当前是否在concurrent marking phase用的是： 

C++代码 

```c++
JavaThread::satb_mark_queue_set().is_active()  

SATBMarkQueueSet只有在concurrent marking时才会被置为active。 

） 
```

CMS的incremental update设计使得它在remark阶段必须重新扫描所有线程栈和整个young gen作为root；G1的SATB设计在remark阶段则只需要扫描剩下的satb_mark_queue。 