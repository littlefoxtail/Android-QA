
# ContentProvider

## 概念

四大组件之一，没有activity那样复杂的生命周期，只有简单的onCreate过程。ContentProvider是一个抽象类，当实现自己的ContentProvider类，只需继承与ContentProvider。
ContentProvider(内容提供者)用于提供数据的统一访问格式，封装底层的具体实现。对于数据的使用者来说，无需知晓数据的来源是数据、文件，或者网络，只需要简单地使用ContentProvider提供的数据操作接口，也就是增删改查。

### 统一资源标识符

- 定义：`Uniform Resource Identifier`，即统一资源标识符
- 作用：唯一标识`ContentProvder`&其中的数据
Uri固定的数据格式，例如：`content://com.gityuan.articles/android/3`

|字段|含义|对应项|
|---|---|--|
|前缀|默认的固定开头格式|content://|
|授权|唯一标识provider|com.gityuan.articles|
|路径|数据类别以及数据项|/android/3|

其他app或者进程想要操作`ContentProvider`，则需要先获取其相应的`ContentResolver`，利用ContentResolver类来完成对数据的增删改查操作

## ContentProvider的线程

### quer/insert/update/delete/etc时候

1. 你的ContentProvider与调用者处于同一进程中，ContentProvider函数与调用者相同的线程上同步运行。
2. 你的ContentProvider与调用者处于不同进程中，ContentProvider函数在ContentProvider进程中的Binder线程池中（不是主线程）上运行。
    - 存在多线程并发访问，需要实现线程同步
      - 若ContentProvider的数据存储方式是使用SQLite一个，则不需要，因为SQLite内部实现好了线程同步，若是多个SQLite则需要，因为SQL对象之间无法进行线程同步
      - 若ContentProvider的数据存储方式是内存，则需要自己实现线程同步

### onCreate

ContentProvider创建后或者打开系统后其他进程第一次访问该ContentProvider时，由系统进行调用，主进程，所以不能做耗时操作

## ContentResolver类

### 作用

统一管理不同`ContentProvider间的操作`

1. 通过`URI`即可操作不同的ContentProvider中的数据
2. 外部进程通过`ContentResolver`类从而与`ContentProvider`类进行交互

### 为什么要使用ContentResolver

- 一般来说，一款应用要使用多个ContentProvider，若需要了解每个ContentProvider的不同实现从而再完成数据交互，操作成本高 & 难度大
- 所以再ContentProvider类上加多了一个 ContentResolver类对所有的ContentProvider进行统一管理。

### 具体使用

提供了与`ContentProvider`类相同名字&作用的4个方法

#### 辅助类

`ContentUris`、`UriMatcher`和`ContentObserver`

## 源码

### query到AMS调用过程

```java
//ContentResolver.java
public final Cursor query(..) {
    //拿到IContentProvider对象，它是ContentProvider的本地代理
    /**
     * 
     */
    IContentProvider unstalbeProvider = acquireUnstableProvider(uri);
    //调用IContentProvider的query函数来进行query
    qCursor = unstableProvider.query(..);
}

//ActivityThread.java
public final IContentProvider acquireProvider(..) {
     /**
      * 1. acquireExistingProvider 方法主要检查 ActivityThread 全局变量
      * mProviderMap 中是否有目标 ContentProvider 存在，有就返回，没有就通过注释 2 处获取，
        */
    final IContentProvider provider = acquireExistingProvider(..);
    if (provider != null) {
        return provider;
    }
    /**
     * 2.调用IActivityManager获取ContentProviderHolder对象
     */
    holder = ActivityManager.getService().getContentProvider(
                getApplicationThread(), auth, userId, stable);

    /**
     * 3.用来安装ContentProvider，并将ContentProvider相关数据存储在mProviderMap中，起到缓存作用
     */
    holder = installProvider(..);
}
```

```java
// ActivityManagerService
private ContentProviderHolder getContentProviderImpl(/*这还是调用者的AT*/IApplicationThread caller, ..) {

    // synchronized(this) {
    //     //获取调用者的ProcessRecord对象
    //     ProcessRecord r = null;
    //     if (caller != null) {
    //         r = getRecordForAppLoced(caller)；
    //     }
    //     /**
    //      * 通过url authority name来获取ContentProviderRecord
    //      * 检查provider已经发布了
    //      */
    //     cpr = mProviderMap.getProviderByName(name, userId);
    //     if (cpr == null && userId != UserHandle.USER_OWNER) {
    //         //插件user 0是否已存有ContentProviderRecord
    //     }
    //     /**
    //      * 判断provider是否正在运行
    //      */
    //     if (providerRunning) {
    //         cpi = cpr.info;
    //         /**
    //          * 如果provider能够在客户端进行直接运行，那么就返回provider给客户端
    //          */
    //         if (r != null && cpr.canRunHere()) {
    //             ContentProvider holder = cpr.newHolder(null);
    //             holder.provider = null;
    //             return holder;
    //         }
    //     }

    //     conn = incProviderCountLocked(r, cpr, token, stable);
    // }

    /**
     * 1. 获取目标ContentProvider应用程序进程的信息，如果进程已经启动就调用注释2，否则调用注释3
     */
    ProcessRecord proc = getProcessRecordLocked(..);
    if (proc != null && proc.thread != null && !proc.killed) {
        if (!proc.pubProviders.containKey(cpi.name)) {
            proc.pubProviders.put(cpi.name, cpr);
            try {
                /**
                 * 2.调用IApplicationThread scheduleInstallProvider函数
                 */
                proc.thread.scheduleInstallProvider(cpi);
            } catch(RemoteException e) {

            }
        } else {
            /**
             *  3.启动新进程
             */
            proc = startProcessLocked(..);
            if (proc == null) {
                return null;
            }
        }
    }

    //等待provider发布

    sychronized(cpr) {
        while(cpr.provider == null) {
            if (cpr.launchingApp == null) {
                return null;
            }
            try {
                if (conn != null) {
                    conn.waiting = true;
                }
                //等待provider发布完成
                cpr.wait(wait);
            } catch (InterruptedException ex) {

            } finally {
                if (con != null) {
                    con.waiting = false;
                }
            }
        }
    }
    return cpr != null ? cpr.newHolder(conn) : null;

}
```

```java
//ActivityThread
private void handleBindApplication(AppBindData data) {
    if (!data.restrictedBackupMode) {
        if (!ArrayUtils.isEmpty(data.providers)) {
            installContentProviders(app, data.providers);
        }
    }
}

//ActivityThread
private void installContentProvider(Context context, List<ProviderInfo> providers) {
    /**
     * 1.遍历当前应用程序进程的ProviderInfo列表，得到每个ContentProvider的存储信息
     */
    for (ProviderInfo cpi : providers) {
        /**
         * 2.调用installProvider方法来启动这些ContentProvider
         */
        ContentProviderHolder cph = installProvider(..);
    }

    /**
     * 3.将启动了的ContentProvider存入AMS的mProviderMap中，用来缓存启动过的ContentProvider
     */
    try {
        ActivityManager.getService().publishContentProviders(
            /*这个是provider所在进程的AT*/getApplicationThread(), results);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }

}

//ActivityThread
private ContentProviderHolder installProvider(Context context, ContentProviderHolder holder, ProviderInfo info) {
    try {
        /**
         * 1. 反射实例化ContentProvider对象
         */
        localProvider = packageInfo.getAppFactory().instantiateProvider(cl, info.name);
        provider = localProvider.getIContentProvider();

        /**
         * 2. 调用它的attachInfo方法
         */
        localProvider.attachInfo(c, info);

    }
}

//AMS
public final void publishContentProviders(IApplicationThread caller, List<ContentProviderHolder> providers) {
    synchronized (this) {
        final ProcessRecord r = getRecordForAppLocked(caller);
        final int N = providers.size()
        for (int i = 0; i < N; i++) {
            ContentProviderHolder src = providers.get(i);

            ContentProviderReecord dst = r.pubProviders.get(src.info.name);
            if (dst != null) {
                ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
                /**
                 * 根据类名进行缓存
                 */
                mProviderMap.putProviderByName(comp, dst)
                String names[] = dst.info.authority.split(";");
                /**
                 * 根据url authority name进行缓存
                 */
                for (in j = 0; j < names.length; j++) {
                    mProviderMap.putProviderByName(names[j], dst);
                }
                //这里移除provider ANR的“定时炸弹”
                if(wasInLaunchingProviders) {
                    mHandler.removeMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
                }

                synchronized (dst) {
                    dst.provider = src.provider;
                    dst.proc = r;
                    //通知AMS provider已经发布成功
                    dst.notifyAll();
                }
            }
        }

    }
}
```

## ContentProvider联级诛杀

```java
//ProcessRecord
void kill(String reason, boolean noisy) {
    if (!killedByAm) {
        if (pid > 0) {
            Process.killProcessQuiet(pid);
            ActivityManagerService.killProcessGroup(uid, pid);
        }
        if (!persistent) {
            killed = true;
            killedByAm = true;
        }
    }
}
```

进程死亡后，将会调用在`attachApplication`时注册的死亡回调

```java
private final class AppDeathRecipient implements IBinder.DeathRecipient {

    public void binderDied() {
        //进程死亡后，将会调用到里面
        synchronized (ActivityManagerService.this) {
            appDiedLocked(mApp, mPid, mAppThread, true);
        }
    }
}
```


```java
//ActivityManagerService
private final void handleAppDiedLocked(..) {
    boolean kept = cleanUpApplicationRecordLocked(..);
}

private final boolean cleanUpApplicationRecordLocked(..) {
    /**
     * 删除和进程相关的已发布的provider
     */
    for (int i = app.pubProviders.size() - 1; i >= 0; i--) {
        ContentProviderRecord cpr = app.pubProviders.valueAt(i);
        final boolean always = app.bad || !allowRestart;
        boolean inLaunching = removeDyingProviderLocked(..)
        if ((inLaunching || always) && cpr.hasConnectionOrHandle()) {
            restart = true;
        }
        cpr.provider = null;
        cpr.proc = null;
    }
    app.pubProviders.clear();
}
```

### 进行provider级联诛杀

```java
//AMS
private final boolean removeDyingProviderLocked(..) {
    final boolean inLaunching = mLaunchingProviders.contains(cpr);
    //如果这个provider尚在运行
    if (!inLaunching || always) {
        synchronized (cpr) {
            cpr.launchingApp = null;
            cpr.notifyAll();
        }
        mProviderMap.removeProviderByClass(cpr.name, UserHandle.getUserId(cpr.uid));
        String names[] = cpr.info.authority.split(";");
        for (int j = 0; j < names.length; j++) {
            mProviderMap.removeProviderByName(names[j], UserHandle.getUserId(cpr.uid));
        }
    }
    //遍历这个进程所有的Connection
    for (int i = cpr.connections.size() - 1; i >= 0; i--) {
        ContentProviderConnection conn = cpr.connections.get(i);
        if (conn.waiting) {
            if (inLaunching && !always) {
                continue;
            }
        }
        //获取这个provider的客户端
        ProcessRecord capp = conn.client;
        conn.dead = true;
        //这个connection的stable计数大于0才会杀死其客户端
        if (conn.stableCount > 0) {
            //如果这个app不是常驻进程且正在运行中，那么会进行联级诛杀，diu boom 呆呆呆(dead dead dead)
            if (!capp.persistent && capp.thread != null
                    && capp.pid != 0
                    && capp.pid != MY_PID) {
                 capp.kill("depends on provider "
                            + cpr.name.flattenToShortString()
                            + " in dying proc " + (proc != null ? proc.processName : "??")
                            + " (adj " + (proc != null ? proc.setAdj : "??") + ")", true);
            }
        } else if (capp.thread != null && conn.provider.provider != null) {
            try {
                capp.thread.unstableProviderDied(conn.provider.provider.asBinder());
            } catch (RemoteException e) {

            }
            cpr.connections.remove(i);
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
```

## 时序图

初始化解析：
![ContentProvider初始化时序图](/img/ContentProvider初始化解析时序图.png)

获取：
![ContentProvider获取时序图](/img/ContentProvider的query时序图.png)
