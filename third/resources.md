# 插件资源篇

## VirtualAPK资源处理

VirtualAPK的插件资源加载分为两种方式：

1. `COMBINE_RESOURCES`模式，将插件的资源全部添加到宿主`Resource`
2. 插件存在一份独立的`Resource`自己使用

### Android 资源加载

#### Resource对象的生成

```java
public class ContextImpl {
    static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk loadedApk, ActivityInfo activityInfo, IBinder activityToken, int displayId,
            Configuration overrideConfiguration) {
        // 创建contextImpl
        context.setResource(resoucesManager.createBaseActivityResources(..));
    }
}
```

```java
public class ResourcesManager {
    public Resource createBaseActivityResources() {
        return getOrCreateResources(activityToken, key, classLoader);
    }

    private Resources getOrCreateResources(IBinder activityToken, ResourceKey key, ClassLoader classLoader) {
        // 创建ResourcesImpl
        ResourcesImpl resourcesImpl = createResourcesImpl(key);

        if (activityToken != null) {
            resources = getOrCreateResourcesForActivityLocked(activityToken, classLoader, resourcesImpl, key.mCompatInfo);
        } else {
            resources = getOrCreateResourcesLocked(classLoader, resourcesImpl, key.mCompatInfo);
        }
        return resources;
    }

    private ResourcesImpl createResourcesImpl(ResourcesKey key) {
        final DisplayAdjustments daj = new DisplayAdjustments(key.mOverrideConfiguration);
        jaj.setCompatibilityInfo(key.mCompatInfo);
        // 创建AssetManager
        // key 中包含addAssetPath方法参数path
        final AssetManager assets = createAssetManager(key);
        if (assets == null) {
            return null;
        }
        final DisplayMetrics dm = getDisplayMetrics(key.mDisplayId, daj);
        final Configutation config = generateConfig(key, dm);
        final ResourcesImpl impl = new ResourcesImpl(assets, dm, config, daj);

        return impl;
    }

    private Resource getOrCreateResourcesForActivityLocked(IBinder activityToke, ClassLoader classLoader, ResourceImpl impl, CompatibilityInfo compatInfo) {
        Resources resources = compatInfo.needsCompatResources() ? new CompatResources(classLoader)
        : new Resources(classLoader);
        resources.setImpl(impl);
        activityResources.activityResources.add(new WeakRefrence<>(resources));
        return resources;
    }
}
```

- 在Android5.0以下：
  - 不支持动态增加资源路径，只要Resources.cpp被创建，就无法动态拓展资源表。
  - 为解决此问题，要重新创建AssetManager、Resources实例，达到组件加载目的。将ActivityThread.mActiveResources、ActivityThread.mActivities、ActivityThread.mServices等中非空Resources对象全部替换为新创建的Resources实例，同时将LoadedApk.mResources、ContextImpl.mResources等的Resources实例置空。创建新的Resources实例，不仅要加载组件的资源路径，而且还要加载已被加载的资源路径。那么首先要解决的是如何获取已加载资源路径。
  - 调用addAssetPath方法会返回一int值(大于等于0)，该值如果等于0说明资源加载失败，大于0则加载成功。在Native端会有一数组记录所有已加载的资源路径，addAssetPath的返回值表示被加载的资源路径在该数组中的索引值。那么AssetManager是否有提供通过索引值获取资源路径呢？答案是有的，通过AssetManager#getCookieName(int)我们就可以获取对应的资源路径，方法入参就是已加载资源路径的索引值，getCookieName方法被标记为隐藏方法，因此需要反射调用。知道了如何获取已加载资源路径，那么能否知道已加载多少个资源路径
- 在资源大于5.0小于7.0：
  - 在Android 5.0及以上，系统支持动态拓展资源表。因此我们只需反射调用AssetManager#addAssetPath(String)，就能将组件资源路径追加入AssetManager。系统为应用程序所创建的Resources实例，会被保存至ResourcesManager.mActiveResources。

- 资源加载版本大于等于7.0
  - 新增ResourcesImpl类，而Resources退化为ResourcesImpl的包装类。即Resources资源获取方法委托至ResourcesImpl实现，ResourcesImpl资源获取方法又委托给AssetManager实现。
  - Android 7.0及以上版本资源加载的思路同资源加载之版本大于等于5.0且小于7.0，需找到ResourcesImpl被存储之地，并调用AssetManager#addAssetPath(String)加载组件资源路径。

### VirtualAPK详细资源加载

在VirtualAPK里插件所有相关的内容都封装到`LoadedPlugin`里，插件的加载行为一般都在这个类的构造方法的实现里。

```java
public class LoadedPlugin {
    LoadedPlugin(PluginManager pluginManager, Context context, File apk) {
        this.mResources = createResources(context, apk);
    }
}

private static Resource createResources(Context context, File apk) {
    if (Constants.COMBINE_RESOURCES) {
        // 如果插件资源合并到宿主里面去的情况，插件可以访问宿主的资源
        Resources resources = ResourcesManager.createResources(context, apk.getAbsolutePath());
        ResourcesManager.hookResources(context, resources);
        return resources;
    } else {
        // 插件使用独立的Resources，不与宿主有关系。无法访问宿主的资源
        Resources hostResources = context.getResources();
        AssetMaanger assetManager = createAssetManager(context, apk);
        return new Resources(assetMnager, hostResources.getDisplayMetrics(),
            hostResources.getConfiguration());
    }
}

//创建一个新的AssetManager
private static AssetManager createAssetManager(Context context, File apk) {
    try {
        AssetManager am = AssetManager.class.newInstance();
        ReflectUtil.invoke(AssetManager.class, am, "addAssetPath", apk.getAbsolutePath());
        return am;
    }  catch (Exception e) {
        e.printStackTrace();
        return null;
    }
}
```

```java
class ResourcesManager {
    public static synchronized Resources createResources(Context hostContext, String apk) {
        Resources hostResources = hostContext.getResources();
        Resources newResources = null;
        AssetManager assetManager;
        try {
            if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
                // 在Android L之前，`addAssetPath`只是把补丁包加入资源路径列表里，但是资源的解析其实很早的时候执行完毕
                assetManager = AssetManager.class.newInstance();
                ReflectUtil.invoke(AssetManagerclass, assetManager, "addAssetPath", hostContext.getApplicationInfo().sourceDir);
            } else {
                assetManager = hostResources.getAssets();
            }
            // 在Android L后直接将插件包的apk地址`addAssetPath`之后就可以
            ReflectUtil.invoke(AssetManager.class, assetManager, "addAssetPath", apk);
            List<LoadedPlugin> pluginList = PluginManager.getInstance(hostContext).getAllLoadedPlugins();
            for(LoadedPlugin plugin : pluginList) {
                ReflectUtil.invoke(AssetManager.class, assetManager, "addAssetPath", plugin.getLocation());
            }
            ....

            newResources = new Resources(assetManager, hostResources.getDisplayMetrics(), hostResources.getConfiguration());

            for (LoadedPlugin plugin : pluginList) {
                plugin.updateResources(newResources);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return newResources;
    }

    private static Resources createResourcesForN(..) {
        String newAssetPath = apk.getAbsolutePath(); // Plugin apk的路径
        ApplicationInfo info = context.getApplicationInfo();
        String baseResDir = info.publicSourceDir; // host的文件路径

        //1. 获取当前项目的splitResDirs，splitResDirs是保存当前程序所有已经加载的资源路径，按索引顺序获取多个拆分apk的完整路径
        // 并检查是否已经包含插件路径，此时表示插件资源路径添加进去了
        info.splitSourceDirs = append(info.splitSourceDirs, newAssetPath); //添加到屁股后面

        // 2. 反射给PackageInfo设置合并之后的splitSourceDirs
        LoadedApk loadedApk = Reflector.with(context).field("mPackageInfo").get();
        Reflector rLoadedApk = Reflector.with(loadedApk).field("mSplitResDirs");
        String[] splitResDirs = rLoadedApk.get();
        rLoadedApk.set(append(splitResDirs, newAssetPath));
        // 3. 获取ResourceImpl及其配置的映射

        final ResoucesManager resourcesManager = ResourcesManager.getInstance();
         ArrayMap<ResourcesKey, WeakReference<ResourcesImpl>> originalMap = Reflector.with(resourcesManager).field("mResourceImpls").get();

        synchronized (resourcesManager) {
            HashMap<ResourcesKey, WeakReference<ResourcesImpl>> resolvedMap = new HashMap<>();
    
            if (Build.VERSION.SDK_INT >= 28
                || (Build.VERSION.SDK_INT == 27 && Build.VERSION.PREVIEW_SDK_INT != 0)) { // P Preview
                ResourcesManagerCompatForP.resolveResourcesImplMap(originalMap, resolvedMap, context, loadedApk);

            } else {
                ResourcesManagerCompatForN.resolveResourcesImplMap(originalMap, resolvedMap, baseResDir, newAssetPath);
            }
            //4. 把mResourceImpls相关的东西给换了
            originalMap.clear();
            originalMap.putAll(resolvedMap);
        }


    }

    private static final class ResourcesManagerCompatForP {
        @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
        public static void resolveResourcesImplMap(Map<ResourcesKey, WeakReference<ResourcesImpl>> originalMap, Map<ResourcesKey, WeakReference<ResourcesImpl>> resolvedMap, Context context, LoadedApk loadedApk) throws Exception {
            //中间map
            HashMap<ResourcesImpl, Context> newResImplMap = new HashMap<>();
            Map<ResourcesImpl, ResourcesKey> resKeyMap = new HashMap<>();
            Resources newRes;

            // Recreate the resImpl of the context

            // See LoadedApk.getResources()
            if (mDefaultConfiguration == null) {
                mDefaultConfiguration = new Configuration();
            }
            // 1. 这里会重新创建Resources，利用LoadedApk.getResources()会把splitResDirs字段加入到ResourcesKey中
            newRes = context.createConfigurationContext(mDefaultConfiguration).getResources();
            newResImplMap.put(newRes.getImpl(), context);

            // Recreate the ResImpl of the activity
            for (WeakReference<Activity> ref : PluginManager.getInstance(context).getInstrumentation().getActivities()) {
                Activity activity = ref.get();
                if (activity != null) {
                    //2. 这里面会重新创建Resources，利用LoadedApk.getResources()会把splitResDirs字段加入到ResourcesKey中
                    newRes = activity.createConfigurationContext(activity.getResources().getConfiguration()).getResources();
                    newResImplMap.put(newRes.getImpl(), activity);
                }
            }
            // Mapping all resKey and resImpl
            for (Map.Entry<ResourcesKey, WeakReference<ResourcesImpl>> entry : originalMap.entrySet()) {
                ResourcesImpl resImpl = entry.getValue().get();
                if (resImpl != null) {
                    // 反转一下key 和 value
                    resKeyMap.put(resImpl, entry.getKey());
                }
                //拷贝一遍
                resolvedMap.put(entry.getKey(), entry.getValue());
            }

            // Replace the resImpl to the new resKey and remove the origin resKey
            for (Map.Entry<ResourcesImpl, Context> entry : newResImplMap.entrySet()) {
                //在重新创建的Resources集合中获取到新的key
                ResourcesKey newKey = resKeyMap.get(entry.getKey());
                ResourcesImpl originResImpl = entry.getValue().getResources().getImpl();

                resolvedMap.put(newKey, new WeakReference<>(originResImpl));
                resolvedMap.remove(resKeyMap.get(originResImpl));
            }
        }
    }
}

public static void hookResources(context base, Resources resources) {
    try {
        ReflectUtil.setField(base.getClass(), base, "mResources", resources);
        Object loadedApk = ReflectUtil.getPackageInfo(base);
        ReflectUtil.setField(loadedApk.getClass(), loadedApk, "mResources", resources);

        Object activityThread = ReflectUtil.getActivityThread(base);
        Object resManager = ReflectUtil.getField(activityThread.getClass(), activityThread, "mResourcesManager");
        if (Build.VERSION.SDK_INT < 24) {
            Map<Object, WeakReference<Resources>> map = (Map<Object, WeakReference<Resources>>) ReflectUtil.getField(resManager.getClass(), resManager, "mActiveResources");
            Object key = map.keySet().iterator().next();
            map.put(key, new WeakReference<>(resources));
        } else {
            // still hook Android N Resources, even though it's unnecessary, then nobody will be strange.
            Map map = (Map) ReflectUtil.getFieldNoException(resManager.getClass(), resManager, "mResourceImpls");
            Object key = map.keySet().iterator().next();
            Object resourcesImpl = ReflectUtil.getFieldNoException(Resources.class, resources, "mResourcesImpl");
            map.put(key, new WeakReference<>(resourcesImpl));
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

1. 由于资源做过分区，则在Android L后直接将插件包的apk地址addAssetPath`之后就可以，但是在Android L之前，addAssetPath只是把补丁包加入到资源路径列表里，但是资源的解析其实是在很早的时候就已经执行完了。
   - mResources指向的是一个ResTable对象，如果它的值不等于NULL，那么就说明当前应用程序已经解析过它使用的资源包里面的resources.arsc文件，因此，这时候AssetManager类的成员函数getResources就可以直接将该ResTable对象返回给调用者。如果还没有初始化 mResources 则按照一定步骤遍历当前应用所使用的每个资源包进而生成 mResources。
   - 具体的初始化过程见[老罗的博客](http://blog.csdn.net/luoshengyang/article/details/8806798)

    ```cpp
    const ResTable* AssetManager::getResTable(bool required) const
    {
        ResTable* rt = mResources;
        if (rt) {
            return rt;
        }
        //....
    }
    ```

2. 由于有系统资源的存在，mResources 的初始化在很早就初始化了，所以我们就算通过addAssetPath方法将 apk 添加到mAssetPaths里，在查找资源的时候也不会找到这部分的资源，因为在旧的 mResources 里没有这部分的 id。
3. 所以在 Android L 之前是需要想办法构造一个新的AssetManager里的 mResources才行，这里有两种方案
   - VirtualAPK 用的是类似 InstantRun 的那种方案，构造一个新的 AssetManager，将宿主和加载过的插件的所有 apk 全都添加一遍，然后再调用hookResources方法将新的 Resources 替换回原来的，这样会引起两个问题，一个是每次加载新的插件都会重新构造一个 AssetManger 和 Resources，然后重新添加所有资源，这样涉及到很多机型的兼容(因为部分厂商自己修改了 Resources 的类名)
   - 一个是需要有一个替换原来Resources的过程，这样就需要涉及到很多地方，从hookResources的实现里看，替换了四处地方，在尽量少的 hook 原则下这样的情况还是尽量避免的。

```java
    public static synchronized Resources createResources(Context hostContext, String apk) {
        Resources hostResources = hostContext.getResources();
        try {
            AssetManager assetManager = hostResources.getAssets();
            if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
                                //我们需要将应用原来加载的地址取出来，详情见①
                List<String> cookieNames = new ArrayList<>();
                int stringBlockCount =
                    (int) ReflectUtil.invoke(AssetManager.class, assetManager, "getStringBlockCount");
                Method getCookieNameMethod = AssetManager.class.getDeclaredMethod("getCookieName", Integer.TYPE);
                getCookieNameMethod.setAccessible(true);
                for (int i = 0; i < stringBlockCount; i++) {
                    String cookieName =
                        (String) getCookieNameMethod.invoke(assetManager, new Object[] {i + 1});
                    cookieNames.add(cookieName);
                }
                ReflectUtil.invoke(AssetManager.class, assetManager, "destroy");
                ReflectUtil.invoke(AssetManager.class, assetManager, "init");
                ReflectUtil.setField(AssetManager.class, assetManager, "mStringBlocks", null);//②
                                //将原来的assets添加进去，有了此步骤就不用刻意添加sourceDir了
                for (String path : cookieNames) {
                    ReflectUtil.invoke(AssetManager.class, assetManager, "addAssetPath", path);
                }
                //插入插件的资源地址
                ReflectUtil.invoke(AssetManager.class, assetManager, "addAssetPath", apk);
                List<LoadedPlugin> pluginList = PluginManager.getInstance(hostContext).getAllLoadedPlugins();
                for (LoadedPlugin plugin : pluginList) {
                    ReflectUtil.invoke(AssetManager.class, assetManager, "addAssetPath", plugin.getLocation());
                }
                ReflectUtil.invoke(AssetManager.class, assetManager, "ensureStringBlocks");//③
                hostResources.updateConfiguration(hostResources.getConfiguration(), hostResources.getDisplayMetrics());//此行代码非常重要④
            } else {
                ReflectUtil.invoke(AssetManager.class, assetManager, "addAssetPath", apk);
                List<LoadedPlugin> pluginList = PluginManager.getInstance(hostContext).getAllLoadedPlugins();
                for (LoadedPlugin plugin : pluginList) {
                    ReflectUtil.invoke(AssetManager.class, assetManager, "addAssetPath", plugin.getLocation());
                }
            }
                        //省去了兼容性的验证
            //            if (isMiUi(hostResources)) {
            //                newResources = MiUiResourcesCompat.createResources(hostResources, assetManager);
            //            } else if (isVivo(hostResources)) {
            //                newResources = VivoResourcesCompat.createResources(hostContext, hostResources,
            // assetManager);
            //            } else if (isNubia(hostResources)) {
            //                newResources = NubiaResourcesCompat.createResources(hostResources, assetManager);
            //            } else if (isNotRawResources(hostResources)) {
            //                newResources = AdaptationResourcesCompat.createResources(hostResources, assetManager);
            //            } else {
            // is raw android resources
            //            newResources =
            //                new Resources(assetManager, hostResources.getDisplayMetrics(), hostResources
            // .getConfiguration());
            //            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return hostResources;
}
```

### Activity启动过程中对资源的处理

```java
public class VAInstrumenttation {
    @Override
    public Activity newActivity(...) {
        try {
            cl.loadClass(className);
        } catch(Exception e) {
            ComponetName component = PluginUtil.getComponent(intent);
            LoadedPlugin plugin = this.mPluginManager.getLoadedPlugin(component);
            String targeClassName = component.getClassName();

            if (plugin != null) {
                Activity activity = mBase.newActivity(plugin.getClassLoader(), targetClassName, intent);
                activity.setIntent(intent);

                try {
                    ReflectUtil.setField(ContextThemeWrapper.class, activity, "mResources", plugin.getResources());
                } catch(Exception ignored) {
                    // ignored
                }
                return activity;
            }
        }
        ...
    }
```

## InstantRun对资源的处理

### 资源修复

```java
public class MonkeyPatcher {
    public static void monkeyPatchExistingResources(..) {
        //1. 通过反射创建一个新的AssetManager，调用addAssetPath添加sdcard上的资源包
        AssetManager newAssetManager = AssetManager.class.getConstructor().newInstance();
        Method mAddAssetPath = AssetManager.class.getDeclaredMethod("addAssetPath", String.class);
            mAddAssetPath.setAccessible(true);
        if (((Integer) mAddAssetPath.invoke(newAssetManager, externalResourceFile)) == 0) {
            throw new IllegalStateException("Could not create new AssetManager");
        }
        //2. 调用ensureStringBlocks将系统资源表以及应用资源表中的字符串资源池地址 5.0以上没用，
        Method mEnsureStringBlocks = AssetManager.class.getDeclaredMethod("ensureStringBlocks");
        mEnsureStringBlocks.setAccessible(true);
        mEnsureStringBlocks.invoke(newAssetManager);

        if (activity != null) {
            //3. 反射获取Activity中的AssetManager的引用，替换成新创建的newAssetManager
            for (Activity activity : activities) {
                Resources resources = activity.getResources();
             try {
                Field mAssets = Resources.class.getDeclaredField("mAssets");
                mAssets.setAccessible(true);
                mAssets.set(resources, newAssetManager);
            } catch (Throwable ignore) {
                Field mResourcesImpl = Resources.class.getDeclaredField("mResourcesImpl");
                mResourcesImpl.setAccessible(true);
                Object resourceImpl = mResourcesImpl.get(resources);
                Field implAssets = resourceImpl.getClass().getDeclaredField("mAssets");
                implAssets.setAccessible(true);
                implAssets.set(resourceImpl, newAssetManager);
            }
        }
        }

        Collection<WeakReference<Resources>> references;
        //得到Resources的弱引用集合，把他们的AssetManager成员替换成newAssetManager
        if (SDK_INT >= KITKAT) {
            // Find the singleton instance of ResourcesManager
                Class<?> resourcesManagerClass = Class.forName("android.app.ResourcesManager");
                Method mGetInstance = resourcesManagerClass.getDeclaredMethod("getInstance");
                mGetInstance.setAccessible(true);
                Object resourcesManager = mGetInstance.invoke(null);
                try {
                    Field fMActiveResources = resourcesManagerClass.getDeclaredField("mActiveResources");
                    fMActiveResources.setAccessible(true);
                    @SuppressWarnings("unchecked")
                    ArrayMap<?, WeakReference<Resources>> arrayMap =
                            (ArrayMap<?, WeakReference<Resources>>) fMActiveResources.get(resourcesManager);
                    references = arrayMap.values();
                } catch (NoSuchFieldException ignore) {
                    Field mResourceReferences = resourcesManagerClass.getDeclaredField("mResourceReferences");
                    mResourceReferences.setAccessible(true);
                    //noinspection unchecked
                    references = (Collection<WeakReference<Resources>>) mResourceReferences.get(resourcesManager);
                }
        } else {
            Class<?> activityThread = Class.forName("android.app.ActivityThread");
            Field fMActiveResources = activityThread.getDeclaredField("mActiveResources");
            fMActiveResources.setAccessible(true);
            Object thread = getActivityThread(context, activityThread);
            @SuppressWarnings("unchecked")
            HashMap<?, WeakReference<Resources>> map =
                    (HashMap<?, WeakReference<Resources>>) fMActiveResources.get(thread);
            references = map.values();
        }
        for (WeakReference<Resources> wr : references) {
                Resources resources = wr.get();
                if (resources != null) {
                    // Set the AssetManager of the Resources instance to our brand new one
                    try {
                        Field mAssets = Resources.class.getDeclaredField("mAssets");
                        mAssets.setAccessible(true);
                        mAssets.set(resources, newAssetManager);
                    } catch (Throwable ignore) {
                        Field mResourcesImpl = Resources.class.getDeclaredField("mResourcesImpl");
                        mResourcesImpl.setAccessible(true);
                        Object resourceImpl = mResourcesImpl.get(resources);
                        Field implAssets = resourceImpl.getClass().getDeclaredField("mAssets");
                        implAssets.setAccessible(true);
                        implAssets.set(resourceImpl, newAssetManager);
                    }

                    resources.updateConfiguration(resources.getConfiguration(), resources.getDisplayMetrics());
                }
            }
    }
}
```

## Qigsaw资源处理

InstantApp的原理

```java
// SplitCompatResourcesLoader
static void loadResources(..) {
    //获取已在的资源，字符串目录，这需要根据android版本获取
    List<String> loadedResDirs = getLoadedResourcesDirs(preResources.getAssets());
    if (!loadedResDirs.contains(splitApkPath)) {
        installSplitResDirs(..);
    }
}

// SplitCompatResourcesLoader
// 从AssetManager中资源路径
private static List<String> getLoadedResourcesDirs(AssetManager asset) {
    // sdk 28前后使用的不同方式
    // 28 AssetManager#getApkAssets ApkAssets#getAssetPath
    // 28以下 AssetManager#mStringBlocks数量 AssetManager#getCookieName
}

// SplitCompatResourcesLoader
private static void installSplitResDirs(..) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODE.LOLLPOP) {
        v21.installSplitResDirs(..);
    } else {
        v14.installSplitResDirs(..);
    }
}

// SplitCompatResourcesLoader
private static class V21 extends VersionCompat {
    private static void installSplitResDirs(Resources preResources, List<String> splitResPaths) throws Throwable {
        Method method = VersionCompat.getAddAssetPathMethod();
        for (String splitResPath : splitResPaths) {
            method.invoke(preResources.getAssets(), splitResPath);
        }
    }
}

// SplitCompatResourcesLoader
private static class V14 extends VersionCompat {
    private static void installSplitResDirs(Context context, Resources preResources, List<String> splitResPaths) throws Throwable {
        //create a new Resources.
        Resources newResources = createResouces();
        //1.对当前context进行更新Resources
        checkOrUpdateResourcesForContext();
        //2.对所有的Activity进行更新Resources
        Object activityThread = getActivityThread();
        Map<IBinder, Object> activities = (Map<IBinder, Object>) mActivitiesInActivityThread().get(activityThread);
        for (Map.Entry<IBinder, Object> entry : activities.entrySet()) {
            Object activityClientRecord = entry.getValue();
            Activity activity = (Activity) HiddenApiReflection.findField(activityClientRecord, "activity").get(activityClientRecord);
            if (context != activity) {
                SplitLog.i(TAG, "pre-resources found in @mActivities");
                checkOrUpdateResourcesForContext(activity, preResources, newResources);
            }
        }
        //3.对mActiveResources进行更新Resources
        Map<Object, WeakReference<Resources>> activeResources;
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
            //对ActivityThread#mActiveResources更新Resources
            activeResources = (Map<Object, WeakReference<Resources>>) mActiveResourcesInActivityThread().get(activityThread);
        } else {
            //对ResourcesManager#mActiveResources
            Object resourcesManager = getResourcesManager();
            activeResources = (Map<Object, WeakReference<Resources>>) mActiveResourcesInResourcesManager().get(resourcesManager);
        }
        for (Map.Entry<Object, WeakReference<Resources>> entry : activeResources.entrySet()) {
            Resources res = entry.getValue().get();
            if (res == preResources) {
                activeResources.put(entry.getKey(), new WeakReference<>(newResources));
                SplitLog.i(TAG, "pre-resources found in @mActiveResources");
                break;
            }
        }
        //4. LoadedApk#mResources（ActivityThread#mPackages 类型ArrayMap<String, WeakReference<LoadedApk>>）
        Map<String, WeakReference<Object>> instance_mPackages =
                    (Map<String, WeakReference<Object>>) mPackagesInActivityThread().get(activityThread);
        for (Map.Entry<String, WeakReference<Object>> entry : instance_mPackages.entrySet()) {
            Object packageInfo = entry.getValue().get();
            Resources resources = (Resources) mResourcesInLoadedApk().get(packageInfo);
            if (resources == preResources) {
                SplitLog.i(TAG, "pre-resources found in @mPackages");
                mResourcesInLoadedApk().set(packageInfo, newResources);
            }
        }
        //5.LoadedApk#mResources（ActivityThread#mResourcePackages 类型ArrayMap<String, WeakReference<LoadedApk>>）
         Map<String, WeakReference<Object>> instance_mResourcePackages =
                    (Map<String, WeakReference<Object>>) mResourcePackagesInActivityThread().get(activityThread);
        for (Map.Entry<String, WeakReference<Object>> entry : instance_mResourcePackages.entrySet()) {
            Object packageInfo = entry.getValue().get();
            Resources resources = (Resources) mResourcesInLoadedApk().get(packageInfo);
            if (resources == preResources) {
                SplitLog.i(TAG, "pre-resources found in @mResourcePackages");
                mResourcesInLoadedApk().set(packageInfo, newResources);
            }
        }

    }

    private static Resources createResources(Context context, Resources oldRes, List<String> splitResPaths) {
        String appResDir = context.getPackageResourcePath();
        AssetManager oldAsset = oldRes.getAssets();
        List<String> resDirs = getAppResDirs(appResDir, oldAsset);
        resDirs.addAll(0, splitResPaths);
        //创建一个新的AssetManager
        AssetManager newAsset = createAssetManager();
        for (String recent : resDirs) {
            int ret = (int) getAddAssetPathMethod().invoke(newAsset, recent);
            if (ret == 0) {

            }
        }
        return newResources(oldRes, newAsset);
    }


}


```

## Replugin资源处理

```java
public class PluginLoadedApk {
    private void createPluginResource() {
        if (VERSION.SDK_INT < 21) {
            am = (AssetManager) AssetManager.class.newInstance();
            ReflectionUtil.on(am).call("addAssetPath", ..);
        } else {
            am = pm.getResourcesForApplication(..);
        }

        ReflectionUtils.on(am).call("addAssetPath", ..sourceDir);
        addWebviewAssetPath(am);

        this.mPluginResources = new ResourcesProxy(..);
    }

}
```

## Tinker资源处理


