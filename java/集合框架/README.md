# 概览

<!-- TOC -->

- [概览](#%E6%A6%82%E8%A7%88)
  - [接口和实现（Interfaces and Implementations）](#%E6%8E%A5%E5%8F%A3%E5%92%8C%E5%AE%9E%E7%8E%B0interfaces-and-implementations)
  - [Collection](#collection)
    - [Set](#set)
    - [List](#list)
      - [ArrayList与LinkedList异同](#arraylist%E4%B8%8Elinkedlist%E5%BC%82%E5%90%8C)
    - [Queue](#queue)
  - [Map](#map)

<!-- /TOC -->

容器，就是可以容纳其他Java对象的对象。
优点：

- 降低编程难度
- 提高程序性能
- 提高API间的互操作性ma
- 降低学习难度
- 降低设计和实现相关API的难度
- 增加程序的重用性

## 接口和实现（Interfaces and Implementations）

JCF(Java Collections Framework)定义了14种容器接口（collection interfaces）：

![图](/img/JCF_Collection_Interfaces.png)
Map接口没有继承自Collection接口，因为Map表示的是关联容器而不是集合。但Java为我们提供了从Map转换到Collection的方法，可以方便的将Map切换到集合视图。上图提供了Queue接口，却没有Stack，这是因为Stack的功能已被JDK1.6引入的Deque取代

## Collection

存储着对象的集合

### Set

- TreeSet：基于红黑树实现，支持有序性操作，例如根据一个范围查找元素的操作。但是查找效率不如HashSet
- HashSet：基于哈希表实现，支持快速查找，但不支持有序性操作，并且失去了元素的插入顺序信息
- LinkedHashSet：具有HashSet的查找效率，且内部使用双向链表维护元素的插入顺序

### List

- [ArraryList](arraylist.md)：基于动态数据实现，支持随机访问ArrayList
- Vector：和ArrayList类似，但它是线程安全的
- [LinkedList](linkedlist.md)：基于双向链表实现，只能顺序访问，但是可以快速地在链表中间插入和删除元素。不仅如此，LinkedList还可以用作栈、队列和双向队列

#### ArrayList与LinkedList异同

- 是否保证线程安全：都不同步
- 底层数据结构：ArrayList底层使用Object数组；LinkedList底层使用的是双向链表数据结构
- 插件和删除是否受元素位置影响：
  - ArrayList采用数组存储，所以插件和删除时间复杂度受位置影响
  - LinkedList采用链表存储，所以插入，删除元素时间复杂度不受元素位置的影响
  - 是否支持快速随机访问：LinkedList不支持，ArrayList支持
  - 内存空间占用：ArrayList的空 间浪费主要体现在在list列表的结尾会预留一定的容量空间，而LinkedList的空间花费则体现在它的每一个元素都需要消耗比ArrayList更多的空间（因为要存放直接后继和直接前驱以及数据）

### Queue

- [LinkedList](linkedlist.md)：可以用它来实现双向队列
- PriorityQueue：基于堆结构实现，可以用来实现优先队列

## Map

存储着键值对的映射表

- TreeMap：基于红黑树实现。
- [HashMap](hashmap.md)：基于哈希表实现。
- HashTable：和HashMap类似，但是它是线程安全的，但是它是遗留类，不应该去使用它。现在可以使用ConcurrentHashMap来支持线程安全，并且ConcurrentHashMap的效率会更高。
- LinkedHashMap：使用双向链表来维护元素的顺序，顺序为插入顺序或者最近最少使用顺序。
