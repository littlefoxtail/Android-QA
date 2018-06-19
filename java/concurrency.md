# 同步

## synchronized

*synchronized可以保证方法或者代码块在运行时，同一时刻只有一个方法可以进入临界区，同时它还可以保证共享变量的内存可见性*
Java中每一个对象都可以作为锁，这是synchronized实现同步的基础：

1. 普通同步方法，锁是当前实例对象
2. 静态同步方法，锁是当前类的class对象
3. 同步方法块，锁是括号里面的对象

当一个线程访问同步代码块，它首先是需要得到锁才能执行同步代码，当退出或者抛出异常时必须要释放锁

```java
public class SynchronizedTest {
    public synchronized void test1(){
      //
    }

    public void test2(){
        synchronized (this){
          //
        }
    }
}
```

```text
Classfile /Users/yetu/iwrotecode/learnleetcode/java/src/SynchronizedTest.class
  Last modified May 1, 2018; size 412 bytes
  MD5 checksum afee2856bc50b730b543fe6392b3ba31
  Compiled from "SynchronizedTest.java"
public class SynchronizedTest
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #3.#16         // java/lang/Object."<init>":()V
   #2 = Class              #17            // SynchronizedTest
   #3 = Class              #18            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               test1
   #9 = Utf8               test2
  #10 = Utf8               StackMapTable
  #11 = Class              #17            // SynchronizedTest
  #12 = Class              #18            // java/lang/Object
  #13 = Class              #19            // java/lang/Throwable
  #14 = Utf8               SourceFile
  #15 = Utf8               SynchronizedTest.java
  #16 = NameAndType        #4:#5          // "<init>":()V
  #17 = Utf8               SynchronizedTest
  #18 = Utf8               java/lang/Object
  #19 = Utf8               java/lang/Throwable
{
  public SynchronizedTest();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public synchronized void test1();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 4: 0

  public void test2();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter //监视器进入，获取锁
         4: aload_1
         5: monitorexit //监视器退出，释放锁
         6: goto          14
         9: astore_2
        10: aload_1
        11: monitorexit
        12: aload_2
        13: athrow
        14: return
      Exception table:
         from    to  target type
             4     6     9   any
             9    12     9   any
      LineNumberTable:
        line 7: 0
        line 9: 4
        line 10: 14
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 9
          locals = [ class SynchronizedTest, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
}
SourceFile: "SynchronizedTest.java"

```

同步代码块是使用`monitorenter`和`monitorexit`指令实现的，同步方法依靠的是方法修饰符上的ACC_SYNCHRONIZED实现
同步代码块：monitorenter指令插入到同步代码块的开始位置，monitorexit指令插入到同步代码块的结束位置，JVM需要保证每一个monitorenter都有一个monitorexit与之相对应。任何对象都有一个monitor与之相关联，当且一个monitor被持有之后，他将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor所有权，即尝试获取对象的锁；

同步方法：synchronized方法则会被翻译成普通的方法调用和返回指令如:invokevirtual、areturn指令，在VM字节码层面并没有任何特别的指令来实现被synchronized修饰的方法，而是在Class文件的方法表中将该方法的access_flags字段中的synchronized标志位置1，表示该方法是同步方法并使用调用该方法的对象或该方法所属的Class在JVM的内部对象表示Klass做为锁对象

## AtomicInteger源码分析

## Thread

Thread的生命周期

1. 新建状态 NEW，此时由于JVM为其分配内存，并初始化其成员变量的值
2. 就绪状态，当线程对象调用了start()方法之后，该线程处于就绪状态。java虚拟机会为其创建方法调用栈和程序计数器，等待调度运行
3. 运行状态，处于就绪状态的线程获得了CPU，开始执行run()方法的线程执行体，则该线程处于运行状态
4. 阻塞状态，当运行状态线程失去所占资源之后，便进行了阻塞状态
5. 死亡状态

```java
public ennm State {
    /**
     * Thread还没有开始
     */
    New,

    /**
     * 就绪阶段，thread 可在java虚拟机中执行，但是可能等待操作系统的其他资源
     */
    RUNNABLE,

    /**
     * 线程阻塞的线程状态，等待监视器锁定,
     * 当线程调用Object.wait()方法进入一个synchronized块/方法或重进入一 * 个synchronized锁/方法时等待获取monitor锁
     */
    BLOCKED,

    /**
     * 一个线程在等待另一个线程执行一个动作时在这个状态
     * 当线程调用以下方法时会进入WAITING状态
     * Object#wait()而且不加超时参数
     * Thread#join()而且不加超时参数
     * LockSupport#park()
     * 在对象上的线程
     */
    WAITING,

    /**
     *  一个线程在一个特定的等待时间等待另一个线程完成一个动作会在这个状态
     * 
     */
    TIMED_WAITING,

}
```

CPU是时分复用的，也即是CPU的时间片，分配给不同的thread/process轮流执行，时间片与时间片之间，需要进行CPU切换，也就是会发生进程的切换、
切换设计到清空寄存器，缓存数据。然后就重新加载新的thread所需数据。当一个线程被挂起时，加入到阻塞队列，在一定的时间或条件下，在通过notify()，notifyAll()唤醒回来。在某个资源不可用的时候，就将cpu让出，把当前等待线程切换为阻塞状态。等待资源可用了，那么就将线程唤醒，让他进入runnable状态等待cpu调度。这就是典型的悲观锁的实现。独占锁是一种悲观锁，synchronized就是一种独占锁，它假设最坏的情况，并且只有在确保其它线程不会造成干扰的情况下执行，会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。

但是，由于在进程挂起和恢复执行过程中存在很大的开销。当一个线程正在等待锁时，它不能做任何事，所以悲观锁有很大的缺点。如果一个线程需要某个资源，但是这个资源占用的时间很短，当线程第一次抢占这个资源时，可能这个资源被占用，如果此时挂起这个线程，可能立即就发现资源可用，然后又需要花费很长的时候重新抢占锁，时间代价就会非常的高。

所以就有了乐观锁的概念，核心思想，每次不加锁而是假设没有冲突去完成某项操作，如果因为冲突失败就重试，直到成功为止。在上面的例子中，某个线程可以不让出CPU，而是一直while循环，如果失败就重试，直到成功为止。所以，当数据争用不严时，乐观锁效果更好。比如CAS就是一种乐观思想的应用。

## java中的CAS的实现

CAS就是Compare and Swap的意思，比较并操作。很多的cpu直接支持CAS指令。CAS是项乐观锁技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量
的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。CAS有三个操作数，内存值V，旧的预期值A，要修改的新值B。并且仅当预期值A和内存值
V相同时，将内存值V修改为B，否则什么都不做

## AtomicInteger实现

AtomicInteger是一个支持原子操作的Integer类，就是保证AtomicInteger类型的变量的增加和减少操作是原子性的，
不会出现多个线程下的数据不一致问题。如果不使用AtomicInteger，要实现一个按顺序获取的ID，就必须在每次获取时进行锁操作，以避免出现并发时获取到同样的ID的现象、
