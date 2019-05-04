# 什么是Android Context

一个Context意味着一个场景，一个场景就是我们和软件进行交互的一个过程

## 一个应用程序中应该有多少个Context对象

![Image](/img/extend.gif)
![context_uml](/img/context_uml.png)

Context类本身是一个纯abstract类。为了使用方便又定义Context包装类

1. ContextImpl类真正实现了Context中所有的函数:
   - ContextWrapper Context的代理类，委托到另一个Context
   - Application/Activity/Service通过attach()设置委托对象
   - ContextThemeWrapper内部包含了与主题相关的接口。这里的主题就是指在Androidmanifest.xml中通过android:theme，为Application或者Activity指定的主题。
只有Activity才需要主题，Service默默的后台工作不需要，所以Service直接继承ContextWrapper
2. Application:四大组件属于某一Application, 获取所在Application:
   - Activity/Service：是通过其方法getApplication，可主动获取当前的所在mApplication；
     - mApplication是由LoadedApk.makeApplication过程所初始化
   - Receiver：只能在onReceiver方法里
   - provider：无

## 什么时候创建的Context

每一个应用程序在客户端都是ActivityThread类开始的，创建Context对象也是在该类中完成，
具体创建ContextImpl类的地方6处：

* PackageInfo.makeApplication()
* performLaunchActivity()
* handleCreateBackupAgent()
* handleCreateService()
* handleBindApplication()
* attach()

其中attach()方法仅在Framework进程启动时调用，应用程序运行时不会调用该方法。

### Context与四大组件的关系

#### Activity的创建流程

```java
public final class ActivityThread {
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");

        ActivityInfo aInfo = r.activityInfo;
        // 获取LoadedApk对象
        if (r.loadedApk == null) {
            r.loadedApk = getLoadedApk(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }
        // 创建ContextImpl对象
        ContextImpl appContext = createBaseContextForActivity(r);
        // 创建Activity对象
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            // 创建Application对象
            Application app = r.loadedApk.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.loadedApk.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.loadedApk.getAppDir());

            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    //5.执行Activity的onCreate()回调方法
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
}
```

#### Service的创建流程

```java
public final class ActivityThread {

    private void handleCreateService(CreateServiceData data) {
    // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        // 1.获取LoadedAPK对象
        LoadedApk loadedApk = getLoadedApkNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            // 2. 创建Service对象
            java.lang.ClassLoader cl = loadedApk.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
            //3.创建ContextImple对象
            ContextImpl context = ContextImpl.createAppContext(this, loadedApk);
            context.setOuterContext(service);
            //4.创建Application
            Application app = loadedApk.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            //5. 执行Service的onCreate()回调方法
            service.onCreate();
            mServices.put(data.token, service);
            try {
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }
}
```

#### 静态广播的创建流程

```java
public final class ActivityThread {
    private void handleReceiver(ReceiverData data) {
        unscheduleGcIdler();

        String component = data.intent.getComponent().getClassName();
        // 1.获取LoadedApk对象
        LoadedApk loadedApk = getLoadedApkNoCheck(data.info.applicationInfo, data,compatInfo);
        IActivityManager mgr = ActivityManager.getService();

        Application app;
        BroadcastReceiver receiver;
        ContextImpl context;
        try {
            // 2. 创建Application对象
            app = LoadedApk.makeApplication(false, mInstrumentation);
            // 3. 获取ContextImpl对象
            context = (ContextImpl) app.getBaseContext();
            if (data.info.splitNmae != null) {
                context = (ContextImpl) context.createContextForSplit(data.info.splitName);
            }
            java.lang.ClassLoader cl = context.getClassLoader();
            data.intent.setExtrasClassLoader();
            data.intent.prepareToEnterProcess();
            data.setExtrasClassLoader(cl);
            // 4. 创建BroadcastReceiver对象
            receiver = (BroadcastReceiver)cl.loadClass(component).newInstance();
        } catch (Exception e) {

        }
        try {
            sCurrentBroadcastIntent.set(data.intent);
            receiver.setPendingResult(data);
            // 5. 回调onReceiver()方法
            receiver.onReceive(context.getReceiverRestrictedContext(),
                    data.intent);
        } catch (Exception e) {
            data.sendFinished(mgr);
        } finally {
            sCurrentBroadcastIntent.set(null);
        }

        if (receiver.getPendingResult() != null) {
            data.finish();
        }

    }
}
```

#### Content Provider的创建流程

```java
public final class ActivityThread {
    private ContentProvideerHolder installProvider(Context context, ContextProviderHolder holder, ProviderInfo info,
    boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;
        if (holded == null || holder.provider == null) {
            Context c = null;
            ApplicationInfo ai = info.applicationInfo;
            if (context.getPackageName().equals(ai.packageName)) {
                c = context;
            } else if (mInitialApplication != null && mInitialApplication.getPackageName().equals(ai.packageName)) {
                c = mInitialApplication;
            } else {
                try {
                    // 1. 创建ContextImpl对象
                    c = context.createPackageContext(ai.packageName, Context.CONTEXT_INCLUDE_CODE);
                } catch (PackageManager.NameNotFoundException e) {

                }
            }
            if (c == null) {
                return null;
            }

            try {
                //2. 创建Content Provider对象
                final java.lang.ClassLoader cl = c.getClassLoader();
                localProvider = (ContentProvider)cl.loadClass(info.name).newInstance();
                provider = localProvider.getIContentProvider();
                if (provider == null) {
                    return null;
                }
                // 3. 将ContextImpl对象绑定到C
                locaalProvider.attachInfo(c, info);
            } catch (java.lang.Exception e) {
                return null;
            }

        } else {
            provider = holder.provider;
        }
    }
}
```

### Application的创建流程

对于四大组件，Application的创建和获取方式也是不尽相同的：

- Activity：通过LoadApk的makeApplication()方法创建
- Service：通过LoadApk的makeApplication()方法创建
- 静态广播：通过其回调方法onReceiver()方法的第一个参数指向Application
- ContentProvider：无法获取Application，因此此时Application不一定初始化

```java
public final class LoadApk {
    public Application makeApplication(boolean forceDefaultAppClass,
                    Instrumentation instrumentation) {
        // Application只会创建一次，如果Application对象已经存在则不再创建，一个APK对应一个
        // LoadedApk对象，一个LoadedApk对象对应一个Application对象
        if (mApplication != null) {
            return mApplication;
        }
    }

    try {
        //1. 创建加载Application的ClassLoader对象
        java.lang.ClassLoader cl = getClassLoader();

        //2. 创建ContextImpl对象
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        //3. 创建Application对象
        app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
        //4. 将Application对象设置给ContextImpl
        appContext.setOuterContext(app);
    } catch(Exception e) {

    }
    // 5. 将Application对象添加到ActivityThread的Application列表中
    mActivityThread.mAllApplication.add(app);

    if (instrumentation != null) {
        try {
            // 6. 执行Application的回调方法onCreate()
            instrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {

        }
    }
}
```

> Application只会创建一次，如果Application对象已经存在则不再创建，一个APK对应一个LoadedApk对象，；一个LoadedApk对象对应一个Application对象

Application对象构建时候通过Instrumentation的newApplication()方法完成的

```java
public class Instrumentation {
    static public Application newApplication(Class<?> clazz, Context context)
        throws InstrantiationException, IllegalAccessException,
        ClassNotFoundException {
            Application app = (Application)clazz.newinstance();
            app.attach(context);
            return app;
        }
}
```