# AbstractQueuedSynchronizer

AQS是一个用来构建锁和同步器的框架，采用了volatile state变量来利用底层unsafe原子性来对状态进行修改，同步器内部依赖一个FIFO的双向队列来完成资源获取线程的排队工作，使用AQS能简单且高效地构造出应用广泛的大量同步器，比如ReentrantLock、Semaphore。

AQS是基于FIFO的队列实现，并且内部维护了一个状态变量，通过原子更新这个状态变量state即可实现加锁解锁操作

![aqsmind2](/img/aqsmind2.png)

只有程序中需要使用condition的时候才会用到condition队列程序中有可能存在多个

1. 使用node实现FIFO队列，可以用于构建锁或者其他同步装置基础框架
2. 内部维护了一个状态变量state，通过院子更新这个状态变量state即可实现加锁解锁操作
3. AQS设计是一个模板方法的设计模式，子类使用都是继承并通过他的方法管理其状态（acquire和release）方法操作其状态
4. AQS可以同时使用独占锁和共享锁

独占锁和共享锁：

锁在一个时间点只能被一个线程战占有。根据锁的获取机制，又分为公平锁和非公平锁等待队列中按照FIFO的原则获取锁，等待时间越长的线程越先获取到锁，这就是公平的获取锁，即公平锁。而非公平锁，线程获取锁的时候，无视等待队列直接获取锁。ReentrantLock和ReentrantLookReadWriteLock.WriteLock都是独占锁

共享锁：同一个时候能够被多个线程获取的锁，能被共享的锁。JUC包中ReentrantReadWriteLock.ReadLock，CyclicBarrier，CountDownLatch和Semaphore都是共享锁。

## 为什么

为什么需要AQS框架，而不是用已有的监视器锁来实现这些同步器。

基于synchronized关键字的监视器锁模式在性能上的优化更针对于几乎只有一个线程、无竞争的加锁的情景，减少加锁内存开销和锁获取时间延迟，如偏向锁、轻量级锁等优化。不适合在多线程多处理器的情况下使用。
相对于内存开销和获取延迟，AQS的性能目标更针对于扩张性（scalability），无论多少线程尝试使用同步器，需要同步的开销都应该是常量级的。

## 设计与实现

同步框架提供两个基本类型的操作acquire和release，acquire表示根据同步器状态请求获取，获取不成功则需要排队等待，获取成功并且完成操作后需要release释放状态让其它线程。
另外还有非阻塞获取、设这获取等待超时时间、可中断的获取等需求场景、排他和共享模式等。

1. 需要保存状态提供给子类去表示具体含义，并且提供查询、设置、原子修改的操作。
2. 其他，要能够阻塞、唤醒线程
3. 有一个容器来存放等待的线程。

### 具体设计

#### 同步状态synchronization管理

state字段可以通过volatile修饰，这样get和set方法具有了voaltile的相关语义，通过可以AtomicInteger或Unsafe类来实现原子CAS更新操作，基于减少indirection的考虑，JDK一般都是使用Unfafe类。

#### 阻塞、唤醒线程

除了基于内置的监视器的synchronizer机制外。`LockSupport`类来解决问题，LockSupport给每个线程绑定类似一个Semaphore的permit许可数量。
park()操作阻塞一直到permit为1并且将permit减1.unpark则进行加1，不过unpark并不会累加permit。

park()返回的几个情况：

1. 之前有其他线程调用剩余的unpark()或在park()之后其他调用了unpark()
2. 其他线程中断了目标线程
3. 调用了虚假返回（类似Object.wait(）的伪通知，所以调用返回时需要检查是否等待条件已经达成

#### 线程等待队列

AQS的核心是一个线程等待队列，采用的是一个先进先出FIFO队列。用来实现一个非阻塞的同步器队列有主要有两个选择：

- Mellor-Crummey and Scott (MCS) locks
- Craig,Landin,and Hagersten (CLH) locks

CLH锁更适合处理取消和超时，所有AQS基于CLH进行修改作为线程等待队列。

#### ConditionObject

ConditionObject来表示监视器风格的等待通知操作，主要是在Lock中，和传统的监视器的区别在于一个Lock可以创建多个Condition。ConditionObject使用相同的内部队列节点，但是维护在一个单独的条件队列中，当收到signal操作的时候将条件队列的节点转移到等待队列。

## AQS原理

### 继承结构

AQS核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且共享资源设置为锁定状态，即将暂时获取不到的锁的线程加入队列中。

AQS使用一个int成员变量来表示同步状态，通过内置的FIFO队列来完成获取资源线程的排队工作。AQS使用CAS对该同步状态进行原子操作实现对其值的修改。

AQS继承于`AbstractOwnableSynchronizer`，用于表示在排他模式下存储获取当前持有线程。目前在`ReentrantLock`、`ReentrantReadWriteLock`、`ThreadPollExecutor.Worker`中有用到。

```java
public abstract class AbstractOwnableSynchronizer {
    //没有使用volatile修饰，提供的get、set方法也没有使用同步或者Unsafe等方式获取字段
    private transient Thread exclusiveOwnerThread;

    //exclusive模式下调用setExclusiveOwnerThread设置为自己的线程和之后清空为null肯定是在同一个线程
    // 
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }

    //通常为判断自己是不是当前同步器持有线程
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

### Node结构

Node表示一个等待线程的节点
AQS中的等待队列节点使用pred节点控制等待状态，并且具有pred和next指针。

```java
class Node {
    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;
    Node nextWaiter;
}
```

其中waitStatus有如下几个值，

- SIGNAL: 需要唤醒next节点
- CANCELLED：表示线程等待已经取消，是唯一一个大于0的状态
- CONDITION：表示线程正在等待一个条件
- PROPAGATE：用于acquireShared中向后传播

#### state状态相关操作

state用voltile修饰，原子操作使用CAS

#### 入列代码

```java
/**
 *
 * 为当前线程和给定的模式创建节点并计入到同步队列中
 */
private Node addWaiter(Node node) {
    //创建一个节点
    Node node = new Node(Thread.currentThread(), mode);
    // 快速尝试添加尾节点，如果失败则调用enq(Node node)方法设置尾节点
    Node pred = tail;
    //判断tail节点是否为空，不为空则添加节点到队列中
    if (pred != null) {
        node.prev = pred;
        // CAS设置尾节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

/**
 * 插入节点到队列中
 */
private Node enq(final Node node) {
    //死循环
    for(;;) {
        Node t = tail;
        //如果队列为空，则首先添加一个空节点到队列中
        if (t == null) {
            if (compareAndSetHead(new Node))
                tail = head;
        } else {
            //tail不为空，则CAS设置尾节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

#### CHL出列

同步队列遵循FIFO规范，首节点的线程在释放同步状态后，将会唤醒后继节点的线程
并且后继节点的线程在获取到同步状态后将会将自己设置为首节点
因为设置首节点是通过获取同步状态成功的线程来完成的，因此设置头结点的方法并不需要使用CAS来保证
因为只有一个线程能获取到同步状态

#### acquire

##### 独占模式acquire

首先尝试一次tryAcquire，如果不成功则添加一个Node节点到等待队列

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
            selfInterrupt();
    }
}
```

- addWaiter先判断tail如果不为空则进行一次快速的插入，否则使用enq进行可能包括初始化的入队操作。
- acquireQueued则在循环中判断node的前继节点是不是head，如果是则继续尝试tryAcquire，如果acquire成功则说明成功通过了acquire，则将自己设置为新的head，则将自己设置为新的head，并设置一些null值防止多余引用导致一些内存GC不掉。否则将前继节点的waitStatus设置为SIGNAL。

##### 共享模式的acquire

shared模式调用的是acquireShared，需要子类实现tryAcquireShared(arg)，需要子类实现tryAcquireShared返回值小于0说明获取失败，等于0表示获取成功，但是接下来的acquireShared不会成功，。

## AQS对资源的共享方式

