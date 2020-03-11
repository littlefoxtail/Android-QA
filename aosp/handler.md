# Handler机制

![handle时序图](/img/handler时序图.png)

## looper循环处理消息队列

主线程默认调用如下两个方法。如果在子线程使用handler就要在前后加入这两个方法，这样就会在子线程中轮询MessageQueue了

### looper.prepare()

创建Looper对象，创建MessageQueue，并且Looper持有MessageQueue的引用，当前线程持有Looper对象引用

```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //这里把new 出来的Looper对象保存到当前线程中
    // ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本
    sThreadLocal.set(new Looper(quitAllowed));
}

//new Looper() 创建MessageQueue,并且保留当前线程
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quiteAllowed);
    mThread = Thread.currentThread();
}
```

### looper.loop()

遍历MessageQueue，调用handler的dispatchMessage()处理Message

```java
//开始循环处理messageQueue
public static void loop() {
    //首先获取ThreadLocal的Looper对象
    final Looper me = myLooper();
    //获取Looper对象引用的MessageQueue
    final MessageQueue queue = me.mQueue;

    //无限循环
    for(;;) {
        //有消息取，否则阻塞
        Message msg = queue.next();
        if (msg == null) {
            return;
        }

        //调用Message对象引用的handler对象的dispatchMessage()分发消息
        msg.target.dispatchMessage(msg);
    }
}
```

## Handler的创建

```java
public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
    }
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException("Cant't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

在Handler构造中，主要初始化了一下变量，并判断Handler对象的初始化不应在内部类，静态类，匿名类中，并且保存了当前线程中的Looper对象。

## handler#sendMessage()将消息加入message队列

跟进源代码，其最后会调用：

```java
//将msg 加入到MessageQueue
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    //msg.target 保持当前handler对象的引用
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

MessageQueue#enqueueMessage()

```java
boolean enqueueMessage(Message msg, long when) {
    ...
    synchronized(this) {
        msg.markInUse();
        msg.when = when;
        Message p = mMessage;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessage = msg;
            needWake = mBlocked;
        } else {
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for(;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p;
            prev.next = msg;
        }
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

### handler处理Message

如果msg 有callback则先执行msg的callback；
否则执行handler的callback的handleMessage;
否则执行handler子类的handleMessage;

```java
public void dispatchMessage(Message msg) {
    //如果Message的回调不为空，则回调callback
    if (msg.callback != null) {
        handlerCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handlerMessage(msg)) {
                return;
            }
        }
        //处理Message
        handleMessage(msg);
    }

}
```

主线程默认就有一个Looper对象，在ActivityThread

```java
public static void main(String[] args) {
Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    AsyncTask.init();

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    Looper.loop();
}
```

### Handle.obtainMessage

Handler@obtainMessage

```java
public final Message obtainMessage() {
    return Message.obtain(this);
}
```

Message@obtain

```java
public static Message obtain(Handler h) {
    Message m = obtain();
    m.target = h;
    return m;
}
```

Message@obtain

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0;
            sPoolSize--;
            return m;
        }
    }
}
```

1. **sPoolSync**：给Message加一个对象锁，不允许多个线程同时访问Message类和obtain方法，保证获取到的sPool是最新的
2. **sPool**：存储循环利用Message的单链表。sPool只是链表的头节点
3. **sPoolSize**：单链表的链表长度，即存储Message对象的个数

obtain对链表操作，具体逻辑：

1. 加锁
2. 判断是否是空链表
3. 链表操作，链表头结点移除作为重用Message对象，第二个节点作为头节点
4. 链表长度减1

## HandlerThread

主要作用是很简单的创建一个带looper的线程，方便的使用handler的机制做异步处理。
但是他只有一个线程，适合串行任务，不适合做并行任务。比如网络IO阻塞时间比较长，推荐
使用线程池或者asyncTask+线程池来处理。

```java
HandlerThread workerThread = new HandlerThread("workerThread", Process.THREAD_PRIORITY_BACKGROUND);
workerThread.start();
mHandler = new Handler(workerThread.getLooper());
```

![handler](/img/thread_looper_messagequeu_message_handler.png)

![allhandler](/img/all_handler.jpg)

## IdleHandler

Idle就是Handler机制提供的一种，可以在Looper事件循环的过程中，当出现空闲的时候，允许执行任务的一种机制。

[你知道android的MessageQueue.idleHandler吗？](https://mp.weixin.qq.com/s/KpeBqIEYeOzt_frANoGuSg)

```java
public static interface IdleHandler {
    boolean queueIdle();
}
```

## Looper.loop()里的死循环卡死问题

涉及到Linux pipe/epoll机制，简单说就是在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写入数据来唤醒主线程工作。这里采用epoll机制，是一个种IO多路复用机制，可以监控多个描述符，当某个描述符就绪（读或写就绪），就立即通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。

### 消息队列的初始化工作

```java
public class MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit(); //通过在java层保存Native对象引用地址来实现关联

}
```

[native层的nativeInit方法](https://android.googlesource.com/platform/frameworks/base/+/master/core/jni/android_os_MessageQueue.cpp)

```cpp
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass class) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }
    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

[NativeMessageQueue的初始化](https://android.googlesource.com/platform/frameworks/base/+/master/core/jni/android_os_MessageQueue.cpp)

```cpp
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        // 创建了一个Looper，这与Java层的一点关系都没有
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```

### 消息队列的循环

循环过程将调用native方法`nativePollOnce`。
