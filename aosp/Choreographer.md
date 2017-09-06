# Choreographer启动流程

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
    //创建回调对象
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
}
```
* mLastFrameTimeNanos:指的上一次帧绘制时间点
* mFrameIntervalNanos：帧间时长，一般等于16.7ms

## FrameHandler
```java
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
```
## 
