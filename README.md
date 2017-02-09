# Android-QA
1. [Effective Java for Android (cheatsheet)](/Effective.md)
2. ​[管理在ViewGroup中的Touch Event](TouchEvent.md)
3. [Android Context](Context.md)
4. [RemoteViews](RemoteViews.md)
5. [Get与Post的区别](GetAndPost.md)
6. [handler机制](handler.md)
* app 被杀掉进程后，是否还能收到广播的问题？

  自android 3.1开始，系统自动给所有intent添加了FLAG_EXCLUDE_STOPPED_PACKAGES，导致app处于停止状态就不能收到广播。要想处于停止状态的app收到广播，需要添加FLAG_INCLUDE_STOPPED_PACKAGES这个标记。这样的话，停止的app应该是收不到系统广播了

* 列举java的集合和继承关系

```
├── Collection
│   ├── List
│   │   ├── LinkedList
│   │   ├── ArrayList
│   │   ├── Vector
│   │   │   ├── Stack
│   ├── Set 

├── Map
│   ├── Hashtable
│   ├── HashMap
│   ├── WeakHashMap
```


* HashMap的实现原理

  1. HashMap概述：HashMap是基于哈希表的Map接口的非同步实现。此实现提供所有可选的映射操作，并允许使用null和null键。此类不保证映射的顺序

* HashMap和HashTable的区别

  1. 继承和实现的区别

     Hashtable是基于陈旧的Dictionary类，完成了Map接口；HashMap是Java 1.2引进的Map接口的一个实现。

  2. 线程安全不同

     HashTable的方法是同步的，HashMap是未同步，所以在多线程场合要手动同步HashMap。

  3. 对null处理不同

     Hashtable不允许null值(key和value都不可以)，HashMap允许null值(key和value都可以)。

  4. 方法不同

     Hashtable有一个contains(Object value)，功能和containsValue(Object value)功能一样。

  5. Hashtable使用Enumeration,HashMap使用使用Iterator。

  6. Hashtable中hash数组默认大小是11，增加的方式是old*2 + 1。HashMap中hash数组的默认大小是16，而且一定是2的指数。

* 进程和线程的区别

  * 程序至少有一个进程，一个进程至少有一个线程
  * 线程的划分尺度小于进程，使得多线程程序的并发性高。
  * 进程在执行过程中拥有独立的内存单元，而多个线程共享内存，从而极大地提高了程序的运行效率。
  * 线程在执行过程中与进程还是有区别的。每一个独立的线程有一个程序运行的入口、顺序执行序列和程序的出口。但是线程不能够独立执行，必须依存在应用程序中，由应用程序提供多个线程执行控制。
  * 线程是进程的一个实体，是CPU调度和分派的基本单位，它是比进程更小的能独立运行的基本单位，线程自己基本上不拥有系统资源，只拥有一点在运行中必不可少的资源，但是它可与同一个进程的其他线程共享进程所拥有的全部资源。
  * 一个线程可以创建和撤销另一个线程；同一个进程的多个线程可以并发执行

* 抽象类接口区别

  1. 默认的方法实现 抽象类可以有默认的方法实现完全是抽象的。接口根本不存在方法的实现。(在java8中新增了新特性，接口也可以有default扩展方法)

  2. 实现子类使用extends关键字来继承抽象类。如果子类不是抽象类的话，它需要提供抽象类中所有声明的方法的实现。子类使用关键字implements来实现接口。它需要提供接口中所有声明的方法的实现

  3. 构造器

     抽象类可以有构造器 接口不能有构造器

  4. 与正常Java类的区

     除了你不能实例化抽象类之外，它和普通Java类没有任何区别，接口是完全不同的类型。

  5. 访问修饰符

     抽象方法可以有public、protected和default这些修饰符，接口方法默认修饰符是public。你不可以使用其它修饰符。

  6. main方法

     抽象方法可以有main方法并且我们可以运行它

     接口没有main方法，因此我们不能运行它。

  7. 多继承

     抽象类在java语言中所表示的是一种继承关系，一个子类只能存在一个父类，但是可以存在多个接口。

  8. 添加新方法

     如果你往抽象类中添加新的方法，你可以给它提供默认的实现。因此你不需要改变你现在的代码。

     如果你往接口中添加方法，那么你必须改变实现该接口的类。
  
  * 描述handler机制的原理
  android提供了Handler和Looper来满足线程间的通信。
  Handler先进先出原则。Looper类用来管理线程内对象之间的消息交换(Message Exchange)
    1. Looper：一个线程可以产生一个Looper对象，由它来管理此线程里的Message Queue(消息队列)
    2. Handler：你可以构造Handler对象来与Looper沟通，以便push新消息到Message Queue里；或者接收Looper从Message Queue所送出来的消息。
    3. Message Queue：用来存放线程放入的消息。
    4. UI thread通常就是main thread，

* Android中如何访问自定义ContentProvider

  通过ContentProvider的Uri访问开放的数据。
   1. ContentResolver对象通过Context提供的方法getContenResolver()来获得。
   2. ContentResover提供了以下方法来操作：insert delete update query这些方法分别会调用ContenProvider中与之对应的方法并得到返回的结果。。