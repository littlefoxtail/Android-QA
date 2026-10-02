# 概述

四大组件都是需要在AndroidManifest中注册，而插件apk中的组件是不可能预先知晓名字，提前注册中宿主的apk中

- Activity：在宿主apk中提前占几个坑，然后通过“欺上瞒下”的方式，启动插件apk的Activity：因为要支持不同的launchMode以及一些特殊的属性，需要占多个坑
- Service：通过代理Service的方式去分发；主进程和其他进程，VirtualAPK使用两个代理Service
- BroadcastReceiver：静态转动态
- ContentProvider：通过一个代理Provider进行分发

## Activity

### 替换Activity

```text
startActivity
    ->Instrumentation.execStartActivity()
        ->ActivityManagerService.startActivity()
            ->ActivityStarter.startActivity()
```

```java
public class ActivityStarter {
    public int startActivity() {
        if (err == ActivityManager.START_SUCCESS && aInfo == null) {
            err = ActivityManager.START_CLASS_NOT_FOUND;
        }
        return err;
    }
}
```

```java
public class Instrumentation {
    public static void checkStartActivityResult(int res, Object intent) {
        switch (res) {
            case ActivityManager.START_CLASS_NOT_FOUND:
                if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                    throw new ActivityNotFoundException(
                            "Unable to find explicit activity class "
                            + ((Intent)intent).getComponent().toShortString()
                            + "; have you declared this activity in your AndroidManifest.xml?");
                throw new ActivityNotFoundException(
                        "No Activity found to handle " + intent);
        }
    }
}
```

欺上瞒下：我们可以在AndroidManifest.xml里面声明一个替身Activity，然后在合适的时候把这个假的替换成我们真正需要启动的Activity

Activity是否存在的校验是发生在AMS所在的系统进程system_server，由于进程隔离的存在，对别的进程无能为力，所以在AMS交互前面就要进行欺骗系统。

hook instrumentation；

```java
private void hookInstrumentationAndHandler() {
    try {
        Instrumentation baseInstrumentation = ReflectUtil.getInstrumentation(this.mContext);
        if (baseInstrumentation.getClass().getName().contains("lbe") {
            System.exit(0);
        })

        final VAInstrumentation instrumentation = new VAInstrumentation(this, baseInstrumentation);
        Object activityThread = ReflectUtil.getActivityThread(this.mContext);
        ReflectUtil.setInstrumentation(activityThread, instrumentation);
        ReflectUtil.setHandlerCallback(this.mContext, instrumentation);
        this.mInstrumentation = instrumentation;
    }
}
```

instrumentation.execStartActivity:

```java
public class VAStrumentation {
    public ActivityResult execStartActivity(...) {
        // 主要是当component为null时，根据启动Activity配置的Action、data
        // category等去已加载的plugin中匹配确定的Activity 
        // 这里是virtual支持隐式Intent的关键
        mPluginManager.getComponentsHandler()
            .transformIntentToExplicitAsNeeded(intent);
        if (intent.getComponent() != null) {
            this.mPluginManager.getComponentsHandler().markIntentIfNeeded(intent);
        }
        ActivityResult result = realExecStartActivity(...)
        return result;
    }
}
```

```java
public class ComponentsHandler {
    public void markIntentIfNeeded(Intent intent) {
        if (intent.getComponent() == null) {
            return;
        }
       String targetPackageName = intent.getComponent().getPackageName();
        String targetClassName = intent.getComponent().getClassName();
        // search map and return specific launchmode stub activity
        // 该方法中判断如果启动的是插件中的类，则将启动包名和Activity类名存到intent中，为了恢复使用哦
        if (!targetPackageName.equals(mContext.getPackageName()) && mPluginManager.getLoadedPlugin(targetPackageName) != null) {
            intent.putExtra(Constants.KEY_IS_PLUGIN, true);
            intent.putExtra(Constants.KEY_TARGET_PACKAGE, targetPackageName);
            intent.putExtra(Constants.KEY_TARGET_ACTIVITY, targetClassName);
            dispatchStubActivity(intent);
        }
    }

    private void dispatchStubActivity(Intent intent) {
        String stubActivity = mStubActivityInfo.getStubActivity(targetClassName, launchMode, themeObj);
        // 替换启动的目标Activity
        intent.setClassName(mContext, stubActivity);
    }
}
```

### 还原Activity

欺骗了AMS，AMS执行完，最终启动的不是要占坑Activity，应该是插件中的目标Activity

```text
-ActivityStackSupervisor.realStartActivityLocked()
    -app.thread.scheduleLaunchActivity()
        -ApplicationThread.scheduleLaunchActivity()
            -H.handleMessage()
                -ActivityThread.handleLaunchActivity()
                    -ActivityThread.performLaunchActivity()
                        -Instrumentation.newActivity()
                            -Activity`<init>`
```

```java
public class VAInstrumentation {
    public Activity newActivity(...) {
        try {
            cl.loadClass(className);
        } catch(ClassNotFoundException e) {
            // 从Intent中取出，之前保存的插件activity信息
            ComponentName component = PluginUtil.getComponent(intent);
            // 通过包名取出对应的插件信息
            LoadedPlugin plugin = this.mPluginManager.getLoadedPlugin(component);
            String targetClassName = component.getClassName();
            String targetClassName = componet.getClassName();

            if (plugin != null) {
                // 还原成生成插件的activity
                Activity activity = mBase.newActivity(Plugin.getClassLoader(), targetClassName, intent);
                activity.setIntent(intent);

                try {
                    ReflectUtil.setField(ContextThemeWrapper.class, activity, "mResources", plugin.getResoureces());
                } catch(Exception ignored) {

                }
                return activity;
            }

        }
        return mBase.newActivity(cl, className, intent);
    }

    public void callActivityOnCreate(Activity activity, Bundle icicle) {
        final Intent intent = activity.getIntent();
        if (PluginUtil.isIntentFromPlugin(intent)) {
            Context base = activity.getBaseContext();
            try {
                LoadedPlugin plugin= this.mPluginManager.getLoadPlugin(intent);
                // 设置mResourece
                ReflectUtil.setField(base.getClass(), base, "mResoureces", plugin.getResources());
                // 设置mBase
                ReflectUtil.setField(ContextWrapper.class, activity, "mBase", plugin.getPluginContext());
                // mApplication对象
                ReflectUtil.setField(Activity.class, activity, "mApplication", plugin.getApplication());
                // 这个字段有可能没有的
                ReflectUtil.setFiledNotException(ContextThemeWrapper.class, activity, "mBase", plugin.getPluginContext());

                ActivityInfo activityInfo = plugin.getActivityInfo(PluginUtil.getComponet(intent));
                if (activityInfo.screenOrientation != AcitivityInfo.SCREEN_ORIENTATION_UNSPECIFIED) {
                    activity.setRequestedOrientation(activityInfo.screenOrientation);
                }
            } catch(Exception e) {
                e.printStackTrace();
            }
        }
        mBase.callActivityOnCreate(activity, icicle);
    }
}
```

## Service的支持

Service和Activity不同:

- Activity在Standard模式下多次启动同一个占坑Activity会创建多个对象来启动我们的目标类。而Service多次启动只会调用onStartCommand方法

- Activity的生命周期是由用户交互决定的，而Service的生命周期是主动通过代码调用的

Virtual apk的做法，即将所有的操作进行拦截，都改为startService，然后统一在onStartCommand中分发

### hook IActivityManager

```java
public class PluginManager {
    private void hookSystemServices() {
        try {
            Singleton<IActivityManager> defaultSingleton;
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                // 拿到mInstance对象，即IActivityManager对象
                // O中 为ActivityManager的 IActivityManagerSingleton字段
                    defaultSingleton = (Singleton<IActivityManager>) ReflectUtil.getField(ActivityManager.class, null, "IActivityManagerSingleton");
            } else {
                // O之前版本为 gDefault字段
                defaultSingleton = (Singleton<IActivityManager>) ReflectUtil.getField(ActivityManagerNative.class, null, "gDefault");
            }
            // 通过动态代理，替换了一个代理对象
            IActivityManager activityManagerProxy = ActivityManagerProxy.newInstance(this, defaultSingleton.get());
        }
    }
}
```

```java
public class ActivityManagerProxy implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if ("startService".equals(method.getName())) {
            try {
                return startService(proxy, method, args);
            } catch (Throwable e) {

            }
        } else if ("stopService".equals(method.name))
    }
}
```

```java
-ContextWrapper.startService(Intent)
    -ContextImpl.startService(Intent)
        -ContextImpl.startServiceCommon(Intent)
            -ActivityManagerService.startService()
```

startService被拦截：

```java
public class ActivityManagerProxy {
    private Object startService(Object proxy, Method method, Object[] args) throws Throwable {
        IApplicationThread appThread = (IApplicationThread) args[0];
        ResolveInfo resolveInfo = this.mPluginManager.resolveService(target, 0);
        if (null == resolveInfo || null == resolveInfo.serviceInfo) {
            return method.invoke(this.mActivityManager, args);
        }
        return startDelegateServiceForTarget(target, resolveInfo.serviceInfo, null, RemoteService.EXTRA_COMMAND_START_SERVICE);
    }

    private ComponentName startDelegateServiceForTarget(Intent target, ServiceInfo serviceInfo, Bundle extras, int command) {
        Intent wrapperIntent = wrapperTragetIntent(target, serviceInfo, extras, command);
        return mPluginManager.getHostContext().startService(wrapperIntent);
    }

    private Intent wrapperTargetIntent(Intent target, ServiceInfo serviceInfo, Bundle extras, int command) {
        target.setComponent(new ComponentName(serviceInfo.packageName, serviceInfo.name));

        String pluginLocation = mPluginManager.getLoadedPlugin(target.getComponent()).getLocation();

        boolean local = PluginUtil.isLocalService()
        Class<? extends Service> delegate = local ? LocalService.class : RemoteService.class;
        Intent intent = new Intent();
        intent.setClass(mPluginManager.getHostContext(), delegate);
        // 将原本的Intent存储到了EXTRA_TARGET
        intent.putExtra(RemoteService.EXTRA_TARGET, target);
        intent.putExtra(RemoteService.EXTRA_COMMAND, command);
        intent.putExtra(RemoteService.EXTRA_PLUGIN_LOCATION, pluginLocation);
        if (extras != null) {
            intent.putExtras(extras);
        }

        return intent;
    }
}
```

### 代理分发

```java
public class LocalService {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        ...
        // 通过反射创建实例，并且调用attach方法，最后调用onCreate onStartCommand
        // 然后保存起来，stop时候取出来调用onDestory方法

        //hook stopServiceToken，在一些特殊的Service上，stopSelf是由自身调用的，最终会调用stopServiceToken，中断为Stop操作
    }
}
```

## BroadcastReceiver的支持

静态转动态

```java
Map<ComponentName, ActivityInfo> receivers = new HashMap<ComponentName, ActivityInfo>();

for (PackageParser.Activity receiver : this.mPackage.receivers) {
    receiver.put(receiver.getComponentName(), receiver.info);

    BroadcastReceiver br = BroadcastReceiver.class.cast(getClassLoader().loadClass(receiver.getComponentName().getClassName()).newInstance());
    for (PackageParser.ActivityIntentInfo aii : receiver.intents) {
        this.mHostContext.registerReceiver(br, aii);
    }

}
```

## ContentProvider的支持

### 插件内跳转

```java
class PluginContext extends ContextWrapper {
    @Overrie
    public ContentResolver getContentResolver() {
        return new PluginContentResolver(getHostContext());
    }
}
```

当调用query方法时候，会辗转调用到`ContentResolver.acquireUnstableProvider`方法。该方法被PluginContentResolver中复写：

```java
public class PluginContentResolver {
    protected IContentProvider acquireUnstableProvider(Context context, String auth) {
        if (mPluginManager.resolveContentProvider(auth, 0) != null) {
            return mPluginManager.getIContentProvider();
        }
        return (IContentProvider) sAcquireUnstableProvider.invoke(mBase, content, auth);
    }
}
```

```java
public class PluginManager {
    public synchronized IContentProvider getIContentProvider() {
        if (mIContentProvider == null) {
            hookIContentProviderAsNeeded();
        }
        return mIContentProvider;
    }

    /**
     * 这个方法的意思，通过反射查询AT中的ContentProvider集合，并利用ipc接口IContentProvider，动态代理，修改uri导致最后转向到
     * 代理的RemoteContentProvider
     **/
    private void hookIContentProviderAsNeed() {
        // 拿到占坑的contentProvider uri
        Uri uri = Uri.parse(PluginContentResolver.getUri(mContext));
        // 主动调用call方法
        mContext.getContentResolver().call(uri, "wakeup", null, null);

    }

```

```java
public class RemoteContentProvider {
    public Cursor quey(..) {
        ContentProvider provider = getContentProvider(uri);//这里完成插件等加载，拿到所要的ContentProvider
        Uri pluginUri = Uri.parse(uri.getQueryParameter(KEY_URI));// 重新取回原来的调用uri
        if (provider != null) {
            return provider.query(..);
        }
    }
}
```
