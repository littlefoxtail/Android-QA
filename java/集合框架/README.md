# 概览

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

[ArrayList](arraylist.md)
[LinkedList](linkedlist.md)

## ArrayList与LinkedList异同

- 是否保证线程安全：都不同步
- 底层数据结构：ArrayList底层使用Object数组；LinkedList底层使用的是双向链表数据结构
- 插件和删除是否受元素位置影响：
  - ArrayList采用数组存储，所以插件和删除时间复杂度受位置影响
  - LinkedList采用链表存储，所以插入，删除元素时间复杂度不受元素位置的影响
  - 是否支持快速随机访问：LinkedList不支持，ArrayList支持
  - 内存空间占用：ArrayList的空 间浪费主要体现在在list列表的结尾会预留一定的容量空间，而LinkedList的空间花费则体现在它的每一个元素都需要消耗比ArrayList更多的空间（因为要存放直接后继和直接前驱以及数据）