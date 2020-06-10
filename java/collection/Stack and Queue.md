# Stack and Queue

## Stack

Java里有一个叫Stack的类，却没有叫Queue的类。当需要使用栈时，Java已不推荐使用Stack，而是使用更高效的ArrayDeque；既然Queue只是一个接口，但需要使用队列时也就首选ArrayDeque了。

1. Stack直接继承至Vector，在其基础上，只是增加栈相关的操作；
2. 在方法上使用`synchronized`来实现线程安全

## Vector

Vector内部实现与`ArrayList`类似，都是使用数组来存储元素。不同的是，Vector的方法上加上`synchronized`修饰词，来实现线程安全

## 总体介绍

Deque接口。