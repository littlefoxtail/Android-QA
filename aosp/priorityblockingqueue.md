# PriotiryBlockingQueue

PriorityBlockingQueue是一个基于优先级堆得无界的并发安全的优先级队列，队列的元素按照其自然顺序进行排序，或者根据构造队列时提供的ComparatorJ进行排序，具体取决所使用的构造方法。

## 实现原理

PriorityBlockingQueue通过使用堆这种数据结构实现将队列中的元素按照某种排序规则进行排序，从而改变先进先出的队列顺序，提供开发者改变队列中元素的顺序的能力。队列中的元素必须是可比较的，即实现Comparable接口，或者在构建函数时候提供可对队列元素进行比较Comparator对象。

## 结构介绍

PriorityBlockQueue通过内部组合PriorityQueue的方式实现优先级队列，另外外层通过ReentrantLock实现线程安全，同时通过Condition实现阻塞唤醒。

### offer(E)

入队操作。此处虽然PriorityBlockingQueue是阻塞队列，但是并没有阻塞的入队方法，因为队列是无界的，所以入队是不会阻塞的。

```java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        boolean ok = q.offer(e);
        assert ok;
    } finally {
        lock.unlock();
    }
}
```
