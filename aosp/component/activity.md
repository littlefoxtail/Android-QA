# Activity

![start_activity](/img/activity.png)

点击一个应用的图标，应用的LancherActivity开始启动，启动请求以一种IPC的方式传入AMS，AMS开始处理启动请求，伴随着Intent与Flag的解析和Activity栈的进出，Activity的生命周期从onCreate方法开始变化。

Activity的复杂性主要体现在两个方面：

- 复杂的启动流程，超长的函数调用链
- Activity栈的管理
- 启动模式、Flag及各种场景对Activity生命周期的影响

1. 点击桌面应用图标，Launcher进程将启动Activity的请求及Binder的方式发送给了AMS
2. AMS接收到启动请求后，交付ActivityStarter处理Intent和Flag等信息，然后再交给ActivityStackSupervisior/ActivityStack 处理Activity进栈相关流程。同时以Socket方式请求Zygote进程fork新进程。
3. Zygote接收到新进程创建请求后fork出新进程。
4. 在新进程里创建ActivityThread对象，新创建的进程就是应用的主线程，在主线程里开启Looper消息循环，开始处理创建Activity。
5. ActivityThread利用ClassLoader去加载Activity、创建Activity实例，并回调Activity的onCreate()方法。这样便完成了Activity的启动。

## Activity的回退栈

- ActivityRecord：管理栈里的Activity相关信息，对应着一个用户界面，是Activity管理的最小单位。
- TaskRecord：是一个栈式结构，每个TaskRecord可能包含一个或多个ActivityRecord，栈顶的ActivityRecord表示当前用户可见的页面;
- ActivityStack：是一个栈式管理结构，每个ActivityStack可能包含一个或多个TaskRecord，栈顶的TaskRecord表示当前用户可见的任务。
- ActivityStackSupervisior：管理者多个ActivityStack，当前只会有一个获取焦点（focused）的ActivityStack。
- ProcessReocord：保存着属于用一个进程的所有ActivityRecord，我们知道同一个应用的Activity可以运行在不同的进程里，同一个TaskRecord里的ActivityRecord 可能属于不同ProcessRecord，反之，运行在不同TaskRecord的ActivityRecord可能属于同一个ProcessRecord。

## ActivityStarter.startActivityUnchecked

```java
private int startActivityUnchecked(..) {
    //初始化启动Activity的各种配置，会先重置在重新配置，其中包括ActivityRecord、Intent等
    setInitialState(..);
    //计算出启动的FLAG，并将计算的值赋值给mLaunchFlags
    computeLaunchingTaskFlags();
    computeSourceStack();
    //将mLaunchFlags设置给Intent，达到设定Activity的启动方式的目的
    mIntent.setFlags(mLaunchFlags);
    //获取是否有可以复用的activity
    ActivityRecord reusedActivity = getReusableIntentActivity();
    //可复用Activity不为空
    if (reusedActivity != null) {
        ...
    }
    mTargetStack.startActivityLocked(..);
    if (mDoResume) {

    }
    mTargetStack.startActivityLocked(..)
    if (mDoResume) {
        final ActivityRecord topTaskActivity =
                mStartActivity.getTask().topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
                || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
            ... ...
        } else {
            if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                mTargetStack.moveToFront("startActivityUnchecked");
            }
            // 调用 resumeFocusedStackTopActivityLocked() 方法
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                    mOptions);
        }
    } else if (mStartActivity != null) {
        mSupervisor.mRecentTasks.add(mStartActivity.getTask());
    }
    ... ...
    return START_SUCCESS
}
```

## ActivityStackSupervisor.resumeFocusedStackTopActivityLocked

```java
boolean resumeFocusedStackTopActivityLocked(..) {
    if (targetStack != null && isFocusedStack(targetStack)) {
        // 调用resumeTopActivityUncheckedLocked方法
        return targetStack.resumeTopActivityUncheckedLocked(..);
    }
}
```

## ActivityStack.resumeTopActivityUncheckedLocked

```java
boolean resumeTopActivityUncheckedLocked(..) {

    mStackSupervisor.isResumeTopActivity = true;
    // 关键方法
    result = resumeTopActivityInnerLocked(prev, options);

}
```

## ActivityStack.resumeTopActivityInnerLocked

```java
private boolean resumeTopActivityInnerLocked(..) {
    if (!mService.mBooting && !mService.mBooted) {
        return false;
    }

    boolean pausing = mStackSupervisor.pauseBackStacks(..);
    if (mResumedActivity != null) {
        //先执行 startPausingLocked方法
        pausing |= startPausingLocked(..);
    }

    ActivityStack lastStack = mStackSupervisor.getLastStack();
    if (next.app != null && next.app.thread != null) {
        synchronized(mWindowManager.getWindowManagerLock()) {
            ... ...
            try {

            } catch (Exception e) {
                ... ...
                //执行startSpecificActivityLocked
                mStackSupervisor.startSpecificActivityLocked(..);
                return true;
            }
        } 
    }
}
```

在`resumeTopActivityInnerLocked`方法中会去判断是否有Activity处于`Resume`状态。
如果有的话先让这个Activity执行`Pausing`过程，然后再执行`startSpecificActivityLocked`方法来启动需要启动的Activity。

## 启动流程

接下来我们看看新进程的启动流程：`startSpecificActivityLocked()`方法。在这个方法中会去根据 进程 和 线程 是否存在判断 App 是否已经启动。

### ActivityStackSupervisor.startSpecificActivityLocked

```java
void startSpecificActivityLocked(..) {
    ProcessRecord app = mService.getProcessRecordLocked(..);
    //如果app存在，并且已经启动
    if (app != null && app.thread != null) {
        try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode,
                            mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
    }
}
```