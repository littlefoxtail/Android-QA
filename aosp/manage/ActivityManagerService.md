# ActivityManagerService

组件启动后，首先需要依赖进程，那么就需要先创建进程，系统需要记录每个进程，这便产生了ProcessRecord。Android中，对于进程的概念被弱化，通过抽象后的四大组件，让开发者感受不到进程的存在。当应用退出时，进程也并非马上退出，而是称为cache/empty进程，下次该应用再启动的时候，可以不用创建进程直接初始化组件即可。

ActivityManagerService是贯穿Android系统组件的核心服务，在ServiceServer执行run()方法的时候被创建，运行在独立的线程中，负责Activity、Service、BroadcastReceiver的启动、切换、调度以及应用进程的管理和调度工作。

> ActivityManager相当于前台接待，她将客户的各种需求传达给大总管ActivityMangerService，但是大总管自己不干活，他招来了很多小弟，他最信赖的小弟ActivityThread替他完成真正的启动、切换、以及退出操作，至于其他的中间环节就交给ActivityStack、ActivityStarter等其他小弟来完成。

## 组件管家ActivityManagerService
负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作，其职责与操作系统中的进程管理和调度模块类似。

### ActivityManagerService的启动流程

SystemServer将系统服务分为三类：

- 引导服务
- 核心服务
- 其他服务

```java
public class SystemServer {
    private void run() {
        startBootstrapServices();
    }

    private void startBootstrapServices() {
        //创建AMS
        mActivityManagerService = mSystemServiceManager.startService(
        ActivityManagerService.Lifecycle.class).getService();
    }
}
```

ActivityManagerService的构造方法如下：

```java
public ActivityManagerService(Context systemContext) {
    //创建并启动系统线程及相关handle
    mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
    mHandlerThread.start();
    mHandler = new MainHandler(mHandlerThread.getLooper());

    //创建用来存储各种组件Activity、Broadcast的数据结构
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", BROADCAST_FG_TIMEOUT, false);
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "background", BROADCAST_BG_TIMEOUT, true);
    mBroadcastQueues[0] = mFgBroadcastQueue;
    mBroadcastQueues[1] = mBgBroadcastQueue;

    mServices = new ActiveServices(this);
    mProviderMap = new ProviderMap(this);
    mAppErrors = new AppErrors(mUiContext, this);
}
```

构造方法主要干了两件事：

- 创建并启动系统线程以及相关Handle
- 创建用来存储各种组件Activity、Broadcast的数据结构。

MainHandle里的Looper来源于线程ServiceThread，该Handle主要用来处理组件调度相关操作。

UiHandle里的Looper来源于线程UiThread（继承于ServiceThread），它的线程名"android.ui"，该Handler主要用来处理UI相关操作。

### 工作流程

- IActivityManager：系统私有API，用于与活动管理器服务进行通信。 此提供从应用程序返回活动管理器的调用。
- ActivityManager：此类提供有关活动，服务和包含过程的信息，并与之交互。

ActivityManager
> ActivityManager是提供给客户端调用的接口，日常开发中我们可以利用ActivityManager来获取系统中正在运行的组件（Activity、Service）、进程（Process）、任务（Task）等信息，ActivityManager定义了相应的方法来获取和操作这些信息

概念区分：

- 进程（Process）：Android系统进行资源调度和分配的基本单位，需要注意的是同一个栈的Activity可以运行在不同的进程里。
- 任务（Task）：Task是一组以栈的形式聚集在一起的Activity的集合，这个任务栈就是一个Task。

## ActivityThread

> ActivityThread管理着应用进程里的主线程，负责Activity、Service、BroadcastReceiver的启动、切换、以及销毁等操作。

### ActivityThread的启动流程

```java
public class ActivityThread {
    public void main() {
        //主线程的looper
        Looper.prepareMainLooper();
        ActivityThread thread = new ActivityThread();
        // 调用attach方法将ApplicationThread对象关联AMS，以便AMS调用ApplicationThread里的方法，IPC的过程
        thread.attach(false, startSeq);
        //主线程的Handle
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        // 开始消息循环
        Looper.loop();
    }
}
```

attach:

```java
public final class ActivityThread {
     @UnsupportedAppUsage
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        //是否是系统进程
        if (!system) {
            // 应用进程的处理
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManager.getService();
            try {
                // 将ApplicationThread对象关联给AMS，以便AMS调用ApplicationThread里的方法，这同样也是一个IPC的过程
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            // Watch for getting close to heap limit.
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                        mSomeActivitiesChanged = false;
                        try {
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                            throw e.rethrowFromSystemServer();
                        }
                    }
                }
            });
        } else {
            // 系统进程的处理方式
            // Don't set application object here -- if the system crashes,
            // we can't display an alert, we just want to die die die.
            android.ddm.DdmHandleAppName.setAppName("system_process",
                    UserHandle.myUserId());
            try {
                mInstrumentation = new Instrumentation();
                mInstrumentation.basicInit(this);
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
                mInitialApplication.onCreate();
            } catch (Exception e) {
                throw new RuntimeException(
                        "Unable to instantiate Application():" + e.toString(), e);
            }
        }

        ViewRootImpl.ConfigChangedCallback configChangedCallback
                = (Configuration globalConfig) -> {
            synchronized (mResourcesManager) {
                // We need to apply this change to the resources immediately, because upon returning
                // the view hierarchy will be informed about it.
                if (mResourcesManager.applyConfigurationToResourcesLocked(globalConfig,
                        null /* compat */)) {
                    updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(),
                            mResourcesManager.getConfiguration().getLocales());

                    // This actually changed the resources! Tell everyone about it.
                    if (mPendingConfiguration == null
                            || mPendingConfiguration.isOtherSeqNewer(globalConfig)) {
                        mPendingConfiguration = globalConfig;
                        sendMessage(H.CONFIGURATION_CHANGED, globalConfig);
                    }
                }
            }
        };
        ViewRootImpl.addConfigCallback(configChangedCallback);
    }
}
```

### ActivityThread工作流程

![activity_thread_structure.png](/img/activity_thread_structure.png)

ActivityThread内部有个Binder对象ApplicationThread，AMS可以调用ApplicationThread里的方法，而
ApplicationThread里的方法利用mH（Handler）发送消息给ActivityThread里的消息队列，ActivityThread再去处理这些消息，进而完成诸如Activity启动等各种操作。
