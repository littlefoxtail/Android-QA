# WeakReference

## Reference入ReferenceQueue队列（Reference.enqueue()）

可以把自己放入ReferenceQueue的单链表中。

```java
public class Reference {
    public boolean enqueue() {
        return this.queue.enqueue(this);
    }
}
```

## ReferenceQueue

引用队列，在检测到适当的可到达性更改后，垃圾回收器将已注册的引用对象添加到该队列中

实现了一个队列的入队和出队，内部元素就是泛型Reference，并且Queue的实现，是由Reference自身的链表结构所实现的。
queue是一个链表的容器，其自己仅存储当前的head节点，而后面的节点由每个reference节点自己通过next来保持即可。

```java
public class ReferenceQueue {
    boolean enqueue(Reference<? extends T> r) {
        synchronize(lock) {
            ReferenceQueue<?> queue = r.queue;
            if ((queue == null) || queue == ENQUEUED)) {
                return false;
            }
            assert queue == this;
            r.queue = ENQUEUED;
            r.next = (head == null) ? r : head;
            head = r;
            queueLength++;
            if (r instanceof FinalReference) {
                sun.misc.VM.addFinalRefCount(1);
            }
            lock.notifyAll();
            return true;
        }
    }
}
```

## Reference

## 构造

1. 带queue，只有不断的轮询reference对象，通过判断里面的get是否返回null
2. 不带queue，queue的意义在于，可以外部对这个queue进行监控。

## 主要成员

1. pending和discovered
    pending：jvm在gc时会将要处理的对象放到这个静态字段上面。
    discovered：表示要处理的对象下一个对象
2. referent
    由GC特别处理
    表示引用的对象，即我们在构造的时候需要被包装在其中的对象。对象即将被回收的定义：此对象除了被reference引用之外没有其它引用了( 并非确实没有被引用，而是gcRoot可达性不可达,以避免循环引用的问题 )。如果一旦被回收，则会直接置为null，而外部程序可通过引用对象本身( 而不是referent，这里是reference#get() )了解到回收行为的产生( PhntomReference除外 )。
3. next
    描述当前引用节点所存储下一个即将被处理的节点。但next仅在放到queue中才会有意义
4. discovered
    被VM使用，
    当处于active状态时：discovered reference的下一个元素是由GC操纵的。
    当处于pending状态：discovered为pending集合中的下一个是元素。
    其他状态：discovered为null。
5. ReferenceHandler线程
    这个线程在Reference类的static构造块中启动，并且被设置为高优先级和daemon状态。此线程要做的事情，是不断的检查pending 是否为null，如果pending不为null，则将pending进行enqueue，否则线程进入wait状态。

- Concurrent Mark-Sweep GC Thread
    并发标记清除垃圾回收器线程，该线程（CMS GC）主要针对于年老代垃圾回收。
- Surrogate Locker Thread（Concurrent CC）
    这个线程主要用于配合CMS垃圾回收器使用，它是一个守护线程，其主要负责处理GC过程中，Java层Reference（软引用、弱引用）与jvm内部层面的对象状态同步。

JVM进行CMS GC的时候，会创建一个ConcurrentMarkSweepThread（简称CMST）线程进行GC，ConcurrentMarkSweepThread线程被创建的同时会创建一个SurrogateLockerThread(简称SLT)线程并且启动它，SLT启动之后，处于等待阶段。

CMST开始GC时，会发一个消息给SLT让它去获取Java层Reference对象的全局锁：Lock。直到CMS GC完毕之后，JVM 会将所有被回收的对象所属的WeakReference容器对象放入到Reference的pending属性当中(每次GC完毕之后，pending属性基本上都不会为null了)，然后通知SLT释放并且notify全局锁:Lock。此时激活了ReferenceHandler线程的run方法，使其脱离wait状态，开始工作了。

```java
private static class ReferenceHandler extends Thread {

    public void run() {
        while(true) {
            tryHandlePending(true);
        }
    }
}

static boolean tryHandlePending(boolean waitForNotify) {
    Reference<Object> r;
    Clean c;
    try {
        synchronized(lock) {
            if (pending != null) {
                c = r instanceOf Cleaner ? (Cleaner) r : null;
                pending = r.discovered;
                r.discovered = null;
            } else {
                if (waitForNotify) {
                    //没有的话等待，后续加入元素调用lock.notify
                    lock.wait();
                }
                return waitForNotify;
            }
        }
    } catch (OutOfMemoryOfError x) {
        Thread.yield();
        return true;
    } catch (InterruptedException x) {
        return true;
    }

    if (c != null) {
        c.clean();
        return true;
    }

    ReferenceQueue<? super Object> q = r.queue;
    if (q != ReferenceQueue.NULL) q.enqueue(r);
    return true;

}
```

ReferenceHandler在Reference类的static构造快中启动，并且被设置为高优先级和daemon状态，此线程要做的事请，是不断的检查pending是否为null。

ReferenceQueue是作为 JVM GC与上层Reference对象管理之间的一个消息传递方式，它使得我们可以对所监听的对象引用可达发生变化时做一些处理
