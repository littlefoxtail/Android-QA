# Choreographer

Android每隔16.6ms会刷新一次屏幕，也就是每过16.6ms底层会发出一个屏幕刷新信号，当我们的app接收到这个屏幕刷新信号时，就会去计算屏幕数据，也就是我们常说的测量、布局、绘制三大流程。这整个过程关键的一点，app需要先向底层注册监听下一个屏幕刷新信号事件，这样当底层发出刷新信号时，才可以找到上层app并回调它的方法来通知事件到达了，app才可以接着去做计算屏幕数据之类的工作。



Choreographer是线程单例的，而且必须要和一个Looper绑定，因为其内部有一个Handler需要和Looper绑定

DisplayEventReceiver是一个abstract class，其JNI的代码部分会创建一个IDisplayEventConnection的VSYNC监听者对象。这样，来自EventThread的VSYNC中断信号就可以传递给Choreographer对象。当VSYNC信号到来时，DisplayEventReceicer的onVsync函数将被调用。

另外DispalyEventReceiver还有一个scheduleVsync函数。当应用需要绘制UI时，将首先申请一次VSYNC中断，然后再在中断处理的onVsync函数进行绘制。

在Activity启动过程，执行完onResume后，会调用Activity.makeVisible()，然后再调用到addView(),层层调用会进入如下方法：

```java
public ViewRootImple(Context context, Display display) {
    mChoreographer = Choreographer.getInstance();
}
```

## 创建Choreographer

```java
private Choreographer(Looper looper) {
    mLooper = looper;
    //创建Handler对象【见小节2.3】
    mHandler = new FrameHandler(looper);
    //创建用于接收VSync信号的对象【见小节2.4】
    mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;  //指的上一次帧绘制时间点
    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate()); //帧间时长，一般等于16.7ms
    //创建回调对象，创建一个大小为3的CallbackQueue队列数组，用于保存不同类型的Callback
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
}
```

## 添加回调

```java
public class Choreographer {
    public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }

    public void postCallbackDelayed(int callbackType, Runnable action, Object token,
    long delayMillis) {
        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }

    private void postCallbackDelayedInternal(...) {
         synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            // 将要执行的回到封装成CallbackRecord对象，保存到mCallbackQueue数组
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
            // 函数执行时间
            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                // 通过异步消息方式实现函数延迟执行
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
}
```

## FrameHandler

```java
public class Choreographer {
    private final class FrameHandler extends Handler {
        public FrameHandler() {
            super(looper);
        }

        public void handleMessage(Message msg) {
            switch(msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
}
```

## scheduleFrameLocked

```java
private void scheduleFrameLocked(long now) {
    // 是否使用VSYNC
    if (USE_VSYNC) {
        // 如果当前线程具备消息循环，则直接请求VSync信号
        if (isRunningOnLooperThreadLocked()) {
            scheduleVsynclocked();
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtFrontOfQueue(msg);
        }

    } else {
        // 没有使用VSYNC，这种情况下，根据屏幕刷新频率计算下一次刷新时间，通过异步消息方式延时执行doFrame函数实现刷新
        final long nextFrameTime = Math.max(mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
        Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, nextFrameTime);
    }
}
```

## Vsync请求过程

```java
public class Choreographer {
    private scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
    }
}
```

```java
public class DisplayEventReceiver {
    public void scheduleVsync() {
        // 通过jni方式调用native层的NativeDisplayEventReceiver对象来请求VSync
        nativeScheduleVsync(mReceiverPtr);
    }
}
```

DisplayEventDispatcher.cpp:

```cpp
static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    //  得到NativeDisplayEventReceiver对象指针
    sp<NativeDisplayEventReceiver> receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    // 通过NativeDispalyEventReceiver请求VSync
    status_t status = receiver->scheduleVsync();
    if (status) {
        String8 message;
        message.appendFormat("Failed to schedule next vertical sync pulse.  status=%d", status);
        jniThrowRuntimeException(env, message.string());
    }
}
```

```cpp
status_t DisplayEventDispatcher::scheduleVsync() {  
    if (!mWaitingForVsync) {  
        ALOGV("receiver %p ~ Scheduling vsync.", this);  
        // Drain all pending events.  
        nsecs_t vsyncTimestamp;  
        uint32_t vsyncCount;  
        readLastVsyncMessage(&vsyncTimestamp, &vsyncCount);  
        status_t status = mReceiver.requestNextVsync();  
        if (status) {  
            ALOGW("Failed to request next vsync, status=%d", status);  
            return status;  
        }  
        mWaitingForVsync = true;  
    }  
    return OK;  
}  
```

DisplayEventReceiver:

```cpp
status_t DisplayEventReceiver::requestNextVsync() {
    if (mEventConnection != NULL) {
        mEventConnection->requestNextVsync();
        return NO_ERROR;
    }
    return NO_INIT;
}
```

## FrameDisplayEventReceiver

FrameDisplayEventReceiver对象用于请求并接收Vsync信号，当Vsync信号到来时，系统会自动调用其onVsync()函数
通过sendMessageAtTime，交由FrameHander来处理消息事件

```java
public class Choreographer {
    private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable {
        public void onVsync(long timestampNanos, int buildInDisplayId, int frame) {
            mFrame = frame;
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestamNanos / TimeUtils.NANOS_PER_MS);
        }
        public void run() {
            doFrame(mTimestampNanos, mFrame);
        }
    }
}
```

## doFrame

- mFrameIntervalNanos代表两帧之间的刷新时间，一般等于16.7ms
- mLastFrameTimeNanos：指的上一次帧绘制时间点

```java
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized(mLock) {
        if (!mFrameScheduled) {
            return;
        }
        // 保存起始时间
        startNanos = System.nanoTime();
        // 由于Vsync事件处理采用的是异步方式，因此这里计算消息发送与函数调用开始之前所花费的时间
        final jitterNanos = startNanos - frameTimeNanos;
        // 如果线程处理该消息的时间超过了屏幕刷新周期
        if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;

            frameTimeNanos = startNanos - lastFrameOffset;
        }
        // 如果这个屏幕刷新周期少于上一个刷新周期，则重新请求Vsync信号
        if (frameTimeNanos < mLastFrameTimeNanos) {
            scheduleVsyncLocked();
            return;
        }

        doCallbacks(...);

    }

}
```

分别是`CALLBACK_INPUT`，`CALLBACK_ANIMATION`，`CALLBACK_TRAVERSAL`. 可看出回调的处理顺序依次为：

- input事件;
- animation动画;
- view布局和绘制;

## doCallbacks

```java
public class Choreographer {
    void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized(mLock) {
            final long now = System.nanoTime();
            // 从队列查找相应类型的CallbackRecord对象
            callbacks = mCallbackQueue[callbackType].extractDueCallbacksLocked(now / TimeUtils.NANOS_PER_MS);
            if (callbaks == null) {
                return; //当队列为空，则直接返回
            }
            mCallbacksRunning = true;
            if (callbackType == Choreographer.CALLBACK_COMMIT) {
                final long jitterNanos = now - frameTimeNanos;
                // 当commit类型回调执行的时间超过2帧，则更新mLastFrameTimeNanos
                if (jitterNanos >= 2 * mFrameIntervalNanos) {
                    final long lastFrameOffset = jitterNanos % mFrameIntervalNanos + mFrameIntervalNonos;
                    frameTimeNanos = now - lastFrameOffset;
                    mLastFrameTimeNanos = frameTimeNanos;
                }
            }
        }
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            c.run(frameTimeNanos);
        }
    }

    private static final class CallbackRecord {
        public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                ((Runnable)action).run();
            }
        }
    }
}
```