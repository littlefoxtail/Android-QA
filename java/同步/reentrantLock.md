
# Java中的同步操作

[同步档](https://fangjian0423.github.io/2016/04/18/java-synchronize-way/)
[ReentrantLock源码分析](http://www.jianshu.com/p/4ca65aa20763)

# ReentrantLock

使用`Synchronize`来做同步处理时，锁的获取和释放都是隐式的，实现的原理是通过编译后加上不同的机器指令来实现。
ReentrantLock，字面意思就是可重入锁，它支持同一个线程对资源的重复加锁，不会出现自己阻塞自己的情况，也是我们平时在处理java并发情况下用的最多的同步组件之一
(volatile, synchronized)，它是基于AQS（AbstractQueueSynchronizer）来实现。

## 使用方法

```java
public class ReentrantLockTest {
    privaate ReentrantLock lock = new ReentrantLock();

    public void execute() {
        lock.lock();
        try {
            ...
        } finally {
            lock.unlock();
        }

    }

    public static void main(String[] args) {
        ReentrantLockTest reen = new ReentrantLockTest();
        Thread thead1 = new Thread(new Runnable() {
            public void run() {
                reen.execute();
            }
        });
        Thread thead2 = new Thread(new Runnable() {
            public void run() {
                reen.execute();
            }
        });
        thread1.start();
        thread2.start();
    }
}
```

ReentrantLock是通过Sync及其子类来实现同步控制。
ReentrantLock也是通过FairSync与NoFairSync来支持ReentrantLock在获取锁时公平与非公平选择

## 公平与非公平锁

|分类|释义|解释|
|:----:|:----|:---:|
|公平锁|多线程并发获取锁时，按照等待时间排序获得锁，等待时间最久的线程先获得锁|相当于买票，后面的人需要排到对尾依次买票，不能插队|
|非公平锁 |获取锁时不考虑排队问题，直接尝试获取锁|抢占式，没来一个人不会去管队列如何，直接尝试获取锁|

```java
//默认非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}

//公平锁
public ReentrantLock() {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

默认一般使用非公平锁，它的效率和吞吐量都比公平锁高的多

### lock方法源码

调用成员变量的lock

```java
public void lock() {
    sync.lock();
}
```

- 非公平锁
    非公平锁lock实现:
    ```java
    final void lock() {
        if (comparAndSetState(0, 1))
            //尝试通过CAS原子的设置state遍历[如果state变量为0则设置为1，设置成功表明当前线程成功获取了锁]，成功后设置当前线程为锁的持有者
            setExclusiveOwnerThread(Thread.currentThread());
        else
        // 否则继续调用AbstractQueueSynchronizer的acquire方法
            acquire(1);
    }
    ```

    非公平锁tryAcquire(int)实现：
    ```java
    final boolean nonfairyTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();//获取状态变量
        if (c == 0) {//表明没有线程占有该同步状态

            // 没有hasQueuedPredecessors，不需要判断队列中是否还有其他线程
            if (compareAndSetState(0, acquires)) {//以原子方式设置该同步状态
                setExclusiveOwnerThread(current);//该线程拥有该FairSync同步状态
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {//当前线程已经拥有该同步状态
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);//重复设置状态变量(锁的可重入特性)
            return true;
        }
        return false;
    }
    ```

- 公平锁

    公平锁lock实现:
    ```java
    public class ReentrantLock {
        static final class FairSync extend Sync {
            final void lock() {
                acquire(1);
            }
        }
    }
    ```

    ```java
    public class AbstractQueuedSynchronizer {
        public final void acquire(int arg) {
            //先调用tryAcquire方法尝试获取锁，如果失败，会构造一个独占的Node节点加入等待队列。对于tryAcquire方法，基于FairSync和NonfairSync又有两种实现。对于非公平锁，会执行该方法
            if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
        }
        // 当前线程写入队列：AQS中的队列由Node节点组成的双向链表实现的
        private Node addWaiter(Node mode) {
            Node node = new Node(Thread.currentThread(), mode);

            Node pred = tail;
            if (pred != null) {
                node.prev = pred;
                if (compareAndSetTail(pred, node)) {
                    pred.next = node;
                    return node;
                }
            }
            enq(node);
            return node;
        }
    }
    ```

    挂起等待线程：
    ```java
    public class AbstractQueuedSynchronizer {
        final boolean acquireQueed(final Node node, int arg) {
            boolean failed = true;
            try {
                 boolean interrupted = false;
                 for (;;) {
                     // 获取到上一个节点是否为头节点
                     final Node p = node.predecessor();
                     // 如果是则尝试获取一次锁，获取成功就万事大吉
                     if (p == head && tryAcquire(arg)) {
                         head = node;
                         p.next = null;
                         failed = false;
                         return interrupted;
                     }
                     // 根据上一个节点的`waitStatus`状态来处理
                     if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrput()) {
                         interruputed = true;
                     }
                 }
            } finally {
                if (failed) {
                    cancelAcquire(node);
                }
            }
        }

        /**
         * 利用LockSupport的part方法来挂起当前线程
         **/
        private final boolean parkAndCheckInterrupt() {
            LockSupport.park(this);
            return Thread.inerrupted();
        }
    }
    ```

    ```java
    public class ReentrantLock {
        static final class FairySync extends Sync {
            protected final boolean tryAcquire(int acquires) {
                final Thread current = Thread.currentThread();
                int c = getState(); //获取状态变量
                //表示目前没有其他线程获得锁，当前线程就可以尝试获取锁
                if (c == 0) {
                    //先判断该线程节点是否是队列的头结点
                    //是则以原子方式设置同步状态，获取锁
                    //否则失败返回
                    if (!hasQueuePredecessors() && compareAndSetState(0, acquires)) {
                        // 没有线程就利用CAS来将state修改为1，当前线程置为获取锁的独占线程
                        setExclusiveOwnerThread(current);
                        return true;
                    }

                } else if(current == getExclusiveOwnerThread()) {//重入锁
                    // 更新值
                    int nextc = c + acquires;
                    if (nextc < 0) 
                        throw new Error("Maximum lock count exceeded");
                    setState(nextc);
                    return true;
                }
                return false;
            }
        }

    }
    ```

### unlock方法源码

公平锁和非公平锁释放流程都是一样的：
ReentrantLock调用unlock方法

```java
public class ReentrantLock {
    public void unlock(){
        Sync.release(1);//每次调用unlock方法，只对state变量减1操作，所以多次加锁后需要多次解锁
    }
}
```

```java
public class AbstractQueuedSynchronizer {
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0) {
                unparkSuccessor(h);
            }
            return true;
        }
        return false;
    }

    //  来唤醒被挂起的线程
    private void unparkSuccessor(Node node) {

    }
}
```

```java
public class ReentrantLock {
    protected final boolean tryRelease(int release) {
        int c = getState() - release;
        if (Thread.currentThread() != getExclusiveOwnerThreadJ())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);//表明目前没有线程持有该锁
        }
        setState(c);
        return free;
    }
}
```

### 总结

- 非公平锁一开始会直接尝试设置获取同步状态变量，获取锁
- 在tryAcquire方法中，公平锁会判断当前线程是否在锁的等待队列的头结点，由于该队列是一个FIFO队列，这样判断可以实现公平锁追求的公平性-即等待时间最长的锁先获得锁
- 我们平时使用ReentrantLock时，默认是使用非公平锁，因为在实际情况中，公平锁往往没有非公平锁的效率高。非公平锁的吞吐量会更高一些
- 公平锁机制能够减少"饥饿"的发生，按队列顺序，排队的线程总能够得到锁。而非公平锁可能导致在队列里的某些线程长时间无法获取锁。

## AbstractQueueSynchronizer的实现分析



> AQS核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制ASQ是用CLH队列锁实现的，即将暂时获取不到锁的线程加入队列中。
> CLH(Cralg,Landin,and Hagersten)队列是一个虚拟的双向队列（虚拟的双向队列即不纯在队列实例，仅存在结点之间的关联关系）。AQS是将每条请求共享资源的线程封装成一个CLH锁队列的一个结点(Node)来实现锁的分配

![aqs_schematic](/img/aqs_schematic.png)

Node的成员变量

```java
static final class Node {
    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;

}
```

waitStatus表示Node节点的一些状态，pre/next表示该队列是由双向链表组成，thread表示该线程入队等待获取锁。对于waitStatus，该Node节点规定了6种状态。

|||
|:--:|:--:|
|SHARED|表示节点处于共享模式，该模式会在AQS提供的acquire/releaseShared接口中使用，而该接口又会在能够共享的同步组件中使用，如读写锁中的读锁等|
|EXCLUSIVE|表示节点处于独占模式，ReentrantLock就是使用这种模式|
|CANCELLED(1)|由于同步队列中 等待的线程等待超时或者被中断，需要从同步队列中取消等待，节点进入该状态不会变化。|
|SIGNAL(-1)|后续节点处于等待状态，而当前节点的线程如果释放了同步状态或被取消，将会会通知后继节点，是后继节点的线程得意继续运行。|
|CONDITION(-2)|节点在等待队列中，节点 线程等待在Condition上，当其他线程对Condition调用了signal方法时，该节点将会从等待队列中转移到同步队列 ，加入到对同步状态的获取中。|
|PROPAGATE(-3)|表示下一次共享式 同步状态获取将会 无条件被传播下去|

### AQS的acquire方法的分析

没有成功获取同步状态的线程会被加入同步等待队列的尾部

* 先创建线程节点并加入同步队列
    ```java
    private Node addWaiter(Node node) {
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        if (pred != null ) {
            node.prev = pred;
            //CAS方式设置队列的尾节点
            // 成功则设置该节点前、后向指针
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
            return node;
        }
        enq(node);//CAS方式设置等待队列尾节点失败
        return node;
    }
    ```

    ```java
    private Node enq(final Node node) {
        for(;;) {
            Node t = tail;
            if (t == null) {
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndsSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
    ```

* 节点入队后，自旋尝试获取同步状态
    ```java
    final boolean acquireQueue(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            //自旋起点
            for (;;) {
                final Node p = node.predecessor();
                //新节点的前驱节点是队列的头结点且尝试获取同步状态
                if (p == head && tryAcquire(arg)) {
                        // 成功则当前节点设置为头结点
                    setHead(node);
                    p.next = null;
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    ```

### compareAndSetState

贯穿于整个ReentrantLock实现原理的重中之重，比较和替换是设计并发算法用到的一种技术。如果当前变量的值与我们期望的值相等，就使用一个新值替换当前变量的值。这些方法都是native方法，利用JNI来完成CPU指令的操作，JAVA的CAS最终利用了CPU的原子操作来保证JAVA原子操作。

### setExclusiveOwnerThread

只是一个简单的set操作，他更新了lock中exclusiveOwnerThread属性，

# synchronized(this)

synchronized关键字和ReentrantLock一样，也支持可重入锁(一个线程可以多次获取同一把锁，无需重新获得锁),但它是一个关键字,是一种语法级别的同步方式，成为内置锁。

synchronized跟ReentrantLock相比，有几点局限性：

1. 加锁的时候不能设置超时。ReentrantLock提供tryLock方法，可以设置超时时间，如果超过了这个时间并且没有获取到锁，就会放弃，而synchronized没有这种功能。
2. ReentrantLock可以使用多个Condition，而Synchronized只能有一个
3. ReentrantLock可以选择公平锁和非公平锁
4. ReentrantLock可以获得正在等待线程的个数，计数器等

## Condition条件对象

条件对象的意义在于对于一个已经获取锁的线程，如果还需要等待其他条件才能继续执行的情况下，才会使用Condition条件对象。

```java
public class ConditionTest {

    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();
        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                lock.lock();
                try {
                    System.out.println(Thread.currentThread().getName() + " run");
                    System.out.println(Thread.currentThread().getName() + " wait for condition");
                    try {
                        condition.await();
                        System.out.println(Thread.currentThread().getName() + " continue");
                    } catch (InterruptedException e) {
                        System.err.println(Thread.currentThread().getName() + " interrupted");
                        Thread.currentThread().interrupt();
                    }
                } finally {
                    lock.unlock();
                }
            }
        });
        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                lock.lock();
                try {
                    System.out.println(Thread.currentThread().getName() + " run");
                    System.out.println(Thread.currentThread().getName() + " sleep 5 secs");
                    try {
                        Thread.sleep(5000l);
                    } catch (InterruptedException e) {
                        System.err.println(Thread.currentThread().getName() + " interrupted");
                        Thread.currentThread().interrupt();
                    }
                    condition.signalAll();
                } finally {
                    lock.unlock();
                }
            }
        });
        thread1.start();
        thread2.start();
    }

}
```

thread1执行到condition.await()时，当前线程会被挂起，直到thread2调用了condition.signalAll()方法后，thread1才会重新被激活执行
thread1调用Condition的await方法之后，thread1线程释放锁，然后马上加入到Condition的等待队列，由于thread1释放了锁，thread2获取锁并执行