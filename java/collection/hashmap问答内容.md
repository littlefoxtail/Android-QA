# 问答内容

## `HashMap`主要用途

HashMap是基于`Map`接口实现的一种键-值对`<key, value>`的存储结构，允许`null`值，同时非有序，非同步。`HashMap`的底层实现是数组+链表+红黑树。它存储和查找数据时，是根据`key`的`hashcode`的值计算出具体存储位置。`HashMap`最多允许一条记录的键`key`为`null`，HashMap增删改查等常规操作都有不错的执行效率，是ArrayList和LinkedList等数据结构的一种折中实现

## Q：当两个对象的hashCode相同会发生什么

以为hashCode相同，不一定就是相等的（equals方法比较），所以两个对象所在数组的下标相同，”碰撞“就此发生。又因为

## Q：hash的实现，为啥要这样实现

JDK1.8，是通过hashCode的`高16位异或低16位`实现的，主要从速度，功效和质量来考虑的，减少系统的开销，也不会造成因为高位没有参与下标的计算，从而引起的碰撞。

## Q：为什么要用异或运算符

异或运算符保证了对象的hashCode的32位值只要有一位发生改变，整个hash返回值就会改变。尽可能减少碰撞

## Q：HashMap的table的容量如何确定？loadFactor是什么？该容量如何变化？这种变化带来什么问题

1. loadFactor是装载因子，主要目的是用来确定table数组是否需要动态扩容，默认值为0.75
2. threshold为阈值
3. 扩容，调用resize方法，将table长度为原来的两倍
4. 如果数据很大的情况下，扩展时将会带来性能的损失，在性能要求很高的地方，这种损失可能是致命

## Q：HashMap的遍历方式及其性能对比

1. `for-each map.keySet()`--只需要K值的时候，推荐使用
2. `for-each map.entrySet()`--当需要V值的时候，推荐使用
3. `for-each map.entrySet() + 临时遍历`
4. `for-each map.entrySet().iterator()`

## Q：HashMap，LinkedHashMap，TreeMap 有什么区别

`LinkedHashMap`保存了记录的插入顺序，在用 Iterator 遍历时，先取到的记录肯定是先插入的；遍历比 HashMap 慢；
`TreeMap`实现`SortMap`接口，能够把它保存的记录根据键排序（默认按键值升序排序，也可以指定排序的比较器）

## Q：HashMap & TreeMap & LinkedHashMap 使用场景

一般情况下，使用最多的是 HashMap。
`HashMap`：在 Map 中插入、删除和定位元素时；
`TreeMap`：在需要按自然顺序或自定义顺序遍历键的情况下；
`LinkedHashMap`：在需要输出的顺序和输入的顺序相同的情况下。

## Q：HashMap 和 HashTable 有什么区别

1. HashMap 是线程不安全的，HashTable 是线程安全的；
2. 由于线程安全，所以 HashTable 的效率比不上 HashMap；
3. HashMap最多只允许一条记录的键为null，允许多条记录的值为null，而 HashTable 不允许；
4. HashMap 默认初始化数组的大小为16，HashTable 为 11，前者扩容时，扩大两倍，后者扩大两倍+1；
5. HashMap 需要重新计算 hash 值，而 HashTable 直接使用对象的 hashCode

## Q：Java 中的另一个线程安全的与 HashMap 极其类似的类是什么？同样是线程安全，它与 HashTable 在线程同步上有什么不同

ConcurrentHashMap 类（是 Java并发包 java.util.concurrent 中提供的一个线程安全且高效的 HashMap 实现）。
HashTable 是使用 synchronize 关键字加锁的原理（就是对对象加锁）；
而针对 ConcurrentHashMap，在 JDK 1.7 中采用 分段锁的方式；JDK 1.8 中直接采用了CAS（无锁算法）+ synchronized。

## Q：HashMap & ConcurrentHashMap 的区别

除了加锁，原理上无太大区别。
另外，HashMap 的键值对允许有null，但是ConCurrentHashMap 都不允许。

## Q：为什么ConcurrentHashMap比HashTable效率要高

`HashTable`使用一把锁（锁住整个链表结构）处理并发问题，多个线程竞争一把锁，容易阻塞
`ConcurrentHashMap`：
JDK1.7中使用分段锁（ReentrantLock+Segment+HashEntry），相当于把一个HashMap分成多个段，每段分配一把锁，这样支持多线程访问。锁粒度：基于`Segment`，包括多个HashEntry
JDK1.8中使用CAS+synchronized+Node+红黑树。锁粒度：Node（首结点）。锁粒度降低了。
