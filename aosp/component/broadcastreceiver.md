# 源码分析

![发送广播时序图](/img/发送广播时序图.png)

![动态注册广播时序图](/img/动态注册广播时序图.png)

## 注册过程

不论静态广播还是动态广播，在使用之前都是需要注册的：

- 动态广播的注册需要借助Context类的registerReceiver，注册的信息存储在AMS中。
- 静态广播的注册直接在AndroidManifest.xml中声明即可，注册的信息存储在PMS中。

```java
public class ContextImpl {
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId, ...) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mLoadedApk != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mLoadedApk.getReceiverDispatcher(receiver, context, scheduler, mMain)
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            final Intent intent = ActivityManager.getService().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                    broadcastPermission, userId, flags);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```

广播的分发过程是在AMS中进行的，而AMS所在的进程和BroadcastReceiver所在的进程不一样，因此要把广播分发到BroadcastRecevier具体的进程需要进行跨进程通信，这个通信的载体就是`IIntentReceiver`类。

```java
public class ActivityManagerService {
    public Intent registerReceiver(.. IIntentReceiver receiver) {
        // 对发送者的身份和权限做出一定的校验

        // 把这个BroadcastReceiver以BroadcastFilter的形式存储在AMS的mReceiverResolver
        // 变量中，供后续使用
        mReceiverResolver.addFilter(bf);

    }
}
```

## 发送过程

```java
public ContextImpl {
    public void sendBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(mMainThread.getApplicationThread(), intent, resolvedType, ...);
        } catch(RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```

```java
public class ActivityManagerService {
    final int broadcastIntentLocked() {
        List receivers = null;
        List<BroadcastFilter> registeredReceivers = null;

        if((intent.getFlags() & Intent.FLAG_RECEIVER_REGISTERED_ONLY)
            == 0) {
            receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
        }
        if (intent.getComponent() == null) {
            if (userId == UserHandler.USER_ALL && callingUid == SHELL_UID) {

            } else {
                // mReceiverResolver存储了动态注册的BroadcastReceiver的信息，从此次取出动态注册的东西
                registeredReceivers = mReceiverResolver.queryIntent(intent, resolvedType, false, userId);
            }
        }
    }

    private List<ResolveInfo> collectReceiverComponents(Intent intent, String resolvedType, int callingUid, int[] users) {
        for (int user : users) {
            // 通过PackageManagerService获取这个广播匹配的静态的BroadcastReceiver信息，
            // 静态BroadcastReceiver的注册过程在PMS进行的
            List<ResolveInfo> newReceivers = AppGlobals.getPackageManager()
                .queryIntentReceivers(intent, resolvedType, pmFlags, user).getList();
        }
    }
}
```

## 接收过程

AMS的broadcastIntentLocked方法中找出符合要求的所有BroadcastReceiver

```java
public class ActivityManagerService {
    final int broadcastIntentLocked() {
        BroadcastQueue queue = broadcastQueueForIntent(intent);
        // 创建了一个BroadcastRecord代表此次发送的这条广播
        BroadcastRecord r = new BroadcastRecord(queue, ...);
        final BroadcastRecord oldRecord = replacePending ? queue.replaceOrderedBroadcastLocked(r) : null;

        if (oldRecord != null) {

        } else {
            queue.enqueueOrderedBroadcastLocked(r);
            queue.scheduleBroadcastsLocked();
        }
    }
}
```

```java
public class BroadcastQueue {
    final void precessNextBroadcast(boolean fromMsg) {
        ..
        performReceiveLocked(..);
    }

    void performReceiveLocked() {
        if (app != null) {

        } else {
            receiver.performReceiver(...);
        }
    }
}
```

```java
public class LoadedApk {
    static final class ReceiverDispatcher {
        final static class InnerReceiver extends IIntentReceiver.Stub {
            if (rd != null) {
                rd.performReceiver(...);
            }
        }

        public void performReceiver(...) {
            final Args args = new Args(..);
            if (intent == null || !mActivityThread.post(args.getRunnable())) {
                if (mRegistered && ordered) {
                    IActivityManager mgr = ActivityManager.getService();
                    args.sendFinished(mgr);
                }
            }
        }

        final class Args extends BroadcastReceiver.PendingResult {
            public final Runnable getRunnable() {
                final BroadcastReceiver receiver = mReceiver;
                final boolean ordered = mOrdered;

                if (receiver == null || intent == null || mForgotten) {
                    if (mRegistered && ordered) {
                        sendFinished(mgr);
                    }
                    return;
                }

                try {
                    ClassLoader cl = mReceiver.getClass().getClassLoader();
                    intent.setExtrasClassLoader(cl);
                    intent.prepareToEnterProcess();
                    setExtrasClassLoader(cl);
                    receiver.setPendingResult(this);
                    receiver.onReceiver(mContext, intent);
                }
            }
        }
    }
}
```


