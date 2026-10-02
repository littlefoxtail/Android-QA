# Android应用Crash是如何处理的

- 对于system_server进程：system_server启动过程中由于RuntimeInit.java的`commonInit`方法设置UncaughtHandler，用于处理未捕获异常；
- 对于普通应用进程：同样会调用RuntimeInit.java的commonInit方法设置UncaughtHandler。

## zygoteInit

```java
public static final Runnable zygoteInit(..) {
    RuntimeInit.commonInit();
    ZygoteInit.nativeZygoteInit();
    return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
}
```

```java
//RuntimeInit
protected static final void commonInit() {
    LoggingHandler loggingHandler = new LoggingHandler();
    RuntimeHooks.setUncaughtExceptionPreHandler(loggingHandler);
    Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler(loggingHandler));
}
```

## KillApplicationHandler

```java
//KillApplicationHandler
public void uncaughtException(Thread t, Throwable e) {
    //保证crash过程处理不会重入
    if (mCrashing) return;
    mCrashing = true;
    // 启动crash对话框，等待处理完成
    ActivityManager.getService().handleApplicationCrash(mApplicationObject, new ApplicationErrorReport.ParcelableCrashInfo(e));
}
```

1. 当system进程crash的信息：
    - 开头`*** FATAL EXCEPTION IN SYSTEM PROCESS: $进程名`；
    - 接着输出发生crash时的调用栈信息；
2. 当app进程crash时的信息：
    - 开头`FATAL EXCEPTION: $进程名`；
    - 紧接着`Process: $进程名, PID: $进程id`；
    - 最后输出发生crash时的调用栈信息。

