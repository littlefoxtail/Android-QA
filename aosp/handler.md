Android中线程通信的工具类。它可以在不同的线程之间发送和处理，从而有效地实现多线程编程。
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
# Framework中的消息队列
> 用户空间会接收到来自内核空间的消息，从下图可知，这部分消息先被Native获知，所以：
> 通过Native层建立消息队列，它拥有消息队列的各种基本能力。
> 利用JNI打通Java层和Native层的Runtime屏障，在Java层映射出消息队列。
> 应用建立在Java层之上，在Java层中实现消息的分发和处理。
> 
> 
# 代码解析
Native中的MessageQueue源码
队列的创建：
```cpp
static jint android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }
    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jint>(nativeMessageQueue);
}
```
创建一个Native层的消息队列，如果创建失败，抛异常信息，返回0

---
NativeMessageQueue类的构造函数：
```cpp
NativeMessageQueue::NativeMessageQueue() :
	mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
	mLooper = Looper::getForThread();
	if (mLooper == NULL) {
		mLooper = new Looper(false);
		Looper::setForThread(mLooper);
	}
}
```
---
Java层的MQ中会使用到的native方法

```java
class MessageQueue {
    private long mPtr; // used by native code

    private native static long nativeInit();

    private native static void nativeDestroy(long ptr);

    private native void nativePollOnce(long ptr, int timeoutMillis); /*non-static for callbacks*/

    private native static void nativeWake(long ptr);

    private native static boolean nativeIsPolling(long ptr);

    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
}
```
## Native层Poll
```cpp
//Looper.h
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj, jlong ptr, jint timeoutMillis) {
	NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
	nativeMessageQueue->pollOnce(env, obj, timeoutMillis);

}

void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
	mPollEnv = env;
	mPollObj = pollObj;
	mLooper->pollOnce(timeoutMillis);
	mPollObj = NULL;
	mPollEnv = NULL;
	if (mExceptionObj) {
	
		env->Throw(mExceptionObj);
		
		env->DeleteLocalRef(mExceptionObj);
		
		mExceptionObj = NULL;
	
	}

}
```
实现：
```cpp
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE
                ALOGD("%p ~ pollOnce - returning signalled identifier %d: "
                        "fd=%d, events=0x%x, data=%p",
                        this, ident, fd, events, data);
#endif
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }
        if (result != 0) {
#if DEBUG_POLL_AND_WAKE
            ALOGD("%p ~ pollOnce - returning result %d", this, result);
#endif
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }
        result = pollInner(timeoutMillis);
    }
}

```

pollInner获取消息：

```java
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
//如果native消息队列指针映射已经为0，即虚引用，说明消息队列已经退出
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
//2.死循环，当为获取到需要分发处理的消息时，保持空转
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
//3.调用native方法，poll message
            nativePollOncej(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
//4.如果发现barrier，即同步屏障，则寻找队列中的下一个可能存在的异步消息
                if (msg != null && msg.target == null) {
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
//5.发现了消息，如果还没有到约定时间的消息，则设置一个“下次唤醒”的最大时间差
                    if (now < msg.when) {
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
//寻找到了“到处理时间”的消息，维护单链表消息
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }
                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }
            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler
                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }
            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;
            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
