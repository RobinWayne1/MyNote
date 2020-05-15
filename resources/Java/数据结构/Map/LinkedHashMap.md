# LinkedHashMap

LinkedHashMap继承自HashMap，通过继承自`HashMap.Node`的内部类`LinkedHashMap.Entry`的两个字段----`before`和`after`,使得用户可以按结点插入的顺序(或者按LRU排列,最近用过的结点排在链表尾)访问table中的结点。

