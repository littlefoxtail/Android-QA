# 概览

容器，就是可以容纳其他Java对象的对象。
优点：

- 降低编程难度
- 提高程序性能
- 提高API间的互操作性
- 降低学习难度
- 降低设计和实现相关API的难度
- 增加程序的重用性

# 接口和实现（Interfaces and Implementations）

JCF(Java Collections Framework)定义了14种容器接口（collection interfaces）：

![图](/img/JCF_Collection_Interfaces.png)
Map接口没有继承自Collection接口，因为Map表示的是关联容器而不是集合。但Java为我们提供了从Map转换到Collection的方法，可以方便的将Map切换到集合视图。上图提供了Queue接口，却没有Stack，这是因为Stack的功能已被JDK1.6引入的Deque取代

[ArrayList](arraylist.md)
[LinkedList](linkedlist.md)