# 一.ANR场景

无论是四大组件或者进程等只要发生ANR，最终都调用AMS.appNotResponding()方法：

以下会触发调用AMS.appNotResponding方法：

1. Service Timeout：比如前台服务在20s内执行完成；
2. BroadcastQueue Timeout：比如前台广播在10s内未执行完成
3. ContentProvider Timeout：内容提供者，在publish过超时10s
4. InputDispatching Timeout：输入事件分发超时5s，包括按键和触摸事件

## Service

### 发送超时消息

system_server进程：

```java
public class ActiveService {
    private final void realStartServiceLocked(..) {
        bumpServiceExecutingLocked(..);
    }

    /**
     * 记录服务方法执行开始时间，并开始监听服务方法执行是否超时
     */
    private final void bumpSeviceExecutingLocked(..) {
        if (r.executeNesting == 0) {
            r.executeFg = fg;
            ServiceState stracker = r.getTracker();
            if (timeoutNeeded && r.app.executingServices.size() == 1) {
                //服务方法首次执行时，调用该方法
                scheduleServiceTimeoutLocked(r, app);
            }
        } else if (r.app != null && fg && !r.app.execServicesFg) {
            r.app.execServicesFg = true;
            //服务方法非首次执行，调用该方法
            scheduleServiceTimeoutLocked(r, app);
        }
        r.executeFg |= fg; //记录服务是否在前台执行
        r.executeNesting++; //记录服务执行方法的次数
        r.executingStart = now; //记录服务方法执行的开始时间
    }

    void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        Message msg = mAm.mHandler.obtainMessage(ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        // static final int SERVICE_TIMEOUT = 20*1000; 20s
        //SERVICE_BACKGROUND_TIMEOUT = 20 * 1000 * 10 200s
        mAm.mHandler.sendMessageDelayed(msg, proc.execServicesFg ? SERVICE_TIMEOUT: SERVICE_BACKGROUND_TIMEOUT);
    }
}
```

### 移除超时消息

应用进程：

```java
public class ActivityThread {
    private void handleCreateService(CreateServiceData data) {
        Service service = null
        try {
            ActivityManager.getService().serviceDoneExecuting(data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        } catch (Exception e) {
        }
    }
}
```

又回到了system_server进程：

```java
public class ActivityManagerService {
    public void serviceDoneExecuting(..) {
        synchronized(this) {
            mServices.serviceDoneExecutingLocked(..);
        }
    }
}

```

```java
public class ActiveService {
    void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {
        serviceDoneExecutingLocked(..);
    }

    private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying, boolean finishing) {
        r.executeNesting--;
        if (r.executeNesting <= 0) {
            if (r.app != null) {
                r.app.execServicesFg = false;
                r.app.executingServices.remove(r);
                if (r.app.executingServices.size() == 0) {
                    //执行完了服务的方法，就移除SERVICE_TIMEOUT_MSG消息，取消对服务方法执行时间的监控
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
                }
            }
        }
    }

}
```

### 执行超时消息

system_server进程：

```java
public class ActivityManagerService {
    final class MainHandler extends Handler {
        public MainHandler (Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handlerMessage(Message msg) {
            switch (msg.what) {
                case SERVICE_TIMEOUT_MSG: {
                    //原来这里可以加一个代码块！！
                    mServices.serviceTimeout((ProcessRecord)msg.obj);
                } break;
            }
        }
    }
}
```

```java
public class ActiveServices {
    void serviceTimeout(ProcessRecord proc) {
        synchronized(mAm) {
            final long now = SystemClock.uptimeMillis();
            //计算当前时间减去服务的超时时间
            final long maxTime  = now - (proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
            ServiceRecord timeout = null;
            long nextTime = 0;
            //遍历服务执行方法列表，如果有方法执行时间超过服务执行超时最大时间，则发送ANR消息
            for (int i=proc.executingServices.size()-1; i>=0; i--) {
                ServiceRecord sr = proc.executingServices.valueAt(i);
                //服务执行开始时间小于maxTime，则说明服务已经超时了
                if (sr.executingStart < maxTime) {
                    timeout = sr;
                    break;
                }
                if (sr.executingStart > nextTime) {
                    //没超时的，更新服务执行下一次开始时间
                    nextTime = sr.executingStart;
                }
            }

            if (timeout != null && mAm.mLruProcesses.contains(proc)) {

            } else {
                Message msg = mAm.mHandler.obtainMessage(ActivityManagerService.SERVICE_TIMEOUT_MSG);
                msg.obj = proc;
                mAm.mHandler.sendMessageAtTime(msg, proc.execServicesFg ? (nextTime+SERVICE_TIMEOUT) : (nextTime + SERVICE_BACKGROUND_TIMEOUT));
            }
        }
        if (anrMessage != null) {
            mAm.mAppErrors.appNotResponding(proc, null, null, false, anrMessage)
        }
    }
}
```

## ContentProvider

ContentProvider Timeout是位于“ActivityManager”线程中的AMS.MainHandler收到CONTENT_RPOVIDER_PUBLISH_TIMEOUT_MSG消息时触发。

ContentProvider 超时为CONTENT_PROVIDER_PUBLISH_TIMEOUT = 10s. 这个跟前面的Service和BroadcastQueue完全不同, 由Provider进程启动过程相关.

### 发送超时消息

```java
public class ActivityManagerService {
    private final boolean attachApplicationLocked(..) {
        boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
        List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;

        if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
            Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT);
            msg.obj = app;
            mHandler.sendMesageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
        }
    }
}
```

### 执行超时任务

```java

public class ActivityManagerService {
    final class MainHandler extends Handler {
        public void handleMessage(Message msg) {
            swtich (msg.what) {
                case CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG: {
                    ProcessRecord app = (ProcessRecord)msg.obj;
                    synchronized (ActivityManagerService.this) {
                        processContentProviderPublishTimedOutLocked(app);
                    }
                }
            }
        }
    }

    private final void processContentProviderPublishTimedOutLocked(ProcessRecord app) {
        cleanupAppInLaunchingProvidersLocked(app, true);
        removeProcessLocked(app, false, true, "timeout publishing content providers");
    }

    boolean cleanupAppInLaunchingProvidersLocked(ProcessRecord app, boolean alwayBad) {
        boolean restart = false;
        for (int i = mLaunchingProviders.size() - 1; i >= 0; i--) {
            ContentProviderRecord cpr = mLaunchingProviders.get(i);
            if (cpr.launchingAp == app) {
                if (!alwaysBad && !app.bad && cpr.hasConnectionOrHandle()) {
                    restart = true;
                } else {
                    removeDyingProviderLocked(app, cpr, true);
                }
            }
        }
    }

    private final boolean remoteDyingProviderLocked(..) {
        final boolean isLaunching = mLaunchingProviders.contains(cpr);
        //如果这个provider还在等待launching就remote
        // 需要把当前等待这个provider的线程都notifyAll，否则就再也没有机会notify了
        if (!inLaunching || always) {
            synchronized (cpr) {
                cpr.launchingApp = null;
                cpr.notifyAll();
            }
            // 从mProviderMap中移除
            mProviderMap.remoteProviderByClass(cpr, name, UserHandler.getUserId(cpr.uid));
            String names[] = cpr.info.authority.split(";");
            for (int j = 0; j < names.length; j++) {
                mProviderMap.removeProviderByName(name[j], UserHandler.getUserId(cpr.uid));
            }
        }
        //判断这个provider上的所有链接
        for (int i = cpr.connection.size() - 1; i >= 0; i--) {
             ContentProviderConnection conn = cpr.connections.get(i);
             if (conn.waiting) {
                 if (inLaunching && !alway) {
                     continue;
                 }
             }
             ProcesRecord capp = conn.client;
             conn.dead = true;
            //如果stableCount大于0，也就是说存在stable的链接
            // server挂掉，那么就会把client也给kill掉
            // stable和unstable的重大区别
             if (conn.stableCount > 0) {
                if (!capp.persistent && capp.thread != null
                            && capp.pid != null
                            && capp.pid != MY_PID) {
                    capp.kill(..);
                }
            } else if (capp.thread != null && conn.provider.provider != null) {
                try {
                    capp.thread.unstableProviderDied(conn.provider.provider.asBinder());
                } catch (RemoteException e) {
                }
                cpr.connection.remote(i);
                if (conn.client.conProviders.remove(conn)) {
                    stopAssociationLocked(capp.uid, capp.processName, cpr.uid, cpr.name);
                }
            }
        }
        if (inLaunching && always) {
            mLaunchingProviders.remove(cpr);
        }
        return inLaunching;
    }
}
```

### 移除超时时间

```java
public final void publishContentProviders(..) {
    synchronized (this) {
        if (wasInLaunchingProviders) {
            mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
        }
    }
}
```

## BroadcastReceiver

BroadcastReceiver Timeout是位于“ActivityManager”线程中的BroadcastQueue.BroadcastHandler收到`BROADCAST_TIMEOUT_MSG`消息时触发。

对于广播队列有两个： foreground队列和background队列:

- 对于前台广播，则超时为BROADCAST_FG_TIMEOUT = 10s；
- 对于后台广播，则超时为BROADCAST_BG_TIMEOUT = 60s

### 发送超时时间

```java
public class BroadcastQueue {
    final void processNextBroadcast(boolean fromMsg) {
        synchronized (mService) {
            processNextBroadcastLocked(fromMsg, false);
        }

        final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
            //处理有序广播
            do {
                r = mOrderedBroadcasts.get(0);
                //获取该广播所有的接受者
                int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
                if (mService.mProcessesReady && r.dispatchTime > 0) {
                    long now = SystemClock.uptimeMills();
                    if ((numReceivers > 0) &&
                            (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                        broadcastTimeoutLocked(false);

                        forceReceive = true;
                        r.state = BroadcastRecord.IDLE;
                    }
                }
                if (r.receivers == null || r.nextReceiver >= numReceivers
                        || r.resultAbort || forceReceive) {
                    if (r.resultTo != null) {
                        //处理广播消息
                        performReceiveLocked(..);
                        r.resultTo = null;
                    }
                    //发送广播超时的消息
                    cancelBroadcastTimeoutLocked();
                }
            } while (r == null);
            //取得的是下条有序广播
            r.receiverTime = SystemClock.uptimeMillis();
            if (!mPendingBroadcastTimeoutMessage) {
                long timeoutTime = r.receiverTime + mTimeoutPeriod;
                //设置广播超时消息
                setBroadcastTimeoutLocked(timeoutTime);
            }
        }
    }

    final void setBroadcastTimeoutLocked(long timeoutTime) {
        if (! mPendingBroadcastTimeoutMessage) {
            Message msg = mHandler.otainMessage(BROADCAST_TIMEOUT_MSG, this);
            mHandler.sendMessageAtTime(msg, timeoutTime);
            mPendingBroadcastTimeoutMessage = true;
        }
    }
}
```

### 移除超时时间

```java
public class BroadcastReceiver {
    public static class PendingResult {
        public final void finish() {
            if (mType == TYPE_COMPONENT) {
                final IActivityManager mgr = ActivityManager.getService();
                if (QueueWork.hasPendingWork()) {
                    //当sp有未同步到磁盘
                    QueueWork.queue(new Runnable()) {
                        public void run() {
                            sendFinished(mgr);
                        }
                    }
                }
            } else {
                sendFinish(mgr);
            }
        }
    }
}
```

执行ActivityManagerService#finishReceiver，才会执行下一个processNextBroadcastLocked

### 执行超时任务

```java
private final class BroadcastHandler extends Handler {
    public void handlerMessage(Message msg) {
        switch (msg.what) {
            case BROADCAST_TIMEOUT_MSG: {
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
        }
    }
}
```
