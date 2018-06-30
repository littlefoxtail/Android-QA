# virtual资源篇

## 概述

VirtualAPK的插件资源加载分为两种方式：

1. `COMBINE_RESOURCES`模式，将插件的资源全部添加到宿主`Resource`
2. 插件存在一份独立的`Resource`自己使用

## Android 资源加载

### Resource对象的生成

```java
public class ContextImpl {
    static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk loadedApk, ActivityInfo activityInfo, IBinder activityToken, int displayId,
            Configuration overrideConfiguration) {
        // 创建contextImpl
        context.setResource(resoucesManager.createBaseActivityResources(...));
    }
}
```

```java
public class ResourcesManager {
    public Resource createBaseActivityResources() {
        return getOrCreateResources(activityToken, key, classLoader);
    }
}
```

```java
public class ResourcesManager {
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
}
```

```java
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
```

```java
public class ResourcesManager {
    private Resource getOrCreateResourcesForActivityLocked(IBinder activityToke,
    ClassLoader classLoader, ResourceImpl impl, CompatibilityInfo compatInfo) {
        Resources resources = compatInfo.needsCompatResources() ? new CompatResources(classLoader)
        : new Resources(classLoader);
        resources.setImpl(impl);
        activityResources.activityResources.add(new WeakRefrence<>(resources));
        return resources;
    }
}
```

### VirtualAPK 资源加载

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

1. 由于资源做过分区，则在Android L后直接将插件包的apk地址addAssetPath`之后就可以，但是在Android L之前，addAssetPath只是把补丁包加入到资源路径列表里，但是资源的解析其实是在很早的时候就已经执行完了
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

mResources指向的是一个ResTable对象，如果它的值不等于NULL，那么就说明当前应用程序已经解析过它使用的资源包里面的resources.arsc文件，因此，这时候AssetManager类的成员函数getResources就可以直接将该ResTable对象返回给调用者。如果还没有初始化 mResources 则按照一定步骤遍历当前应用所使用的每个资源包进而生成 mResources 。
具体的初始化过程见[老罗的博客](http://blog.csdn.net/luoshengyang/article/details/8806798)
2. 由于有系统资源的存在，mResources 的初始化在很早就初始化了，所以我们就算通过addAssetPath方法将 apk 添加到mAssetPaths里，在查找资源的时候也不会找到这部分的资源，因为在旧的 mResources 里没有这部分的 id。
3. 所以在 Android L 之前是需要想办法构造一个新的AssetManager里的 mResources  才行，这里有两种方案，VirtualAPK 用的是类似 InstantRun 的那种方案，构造一个新的 AssetManager，将宿主和加载过的插件的所有 apk 全都添加一遍，然后再调用hookResources方法将新的 Resources  替换回原来的，这样会引起两个问题，一个是每次加载新的插件都会重新构造一个 AssetManger 和 Resources，然后重新添加所有资源，这样涉及到很多机型的兼容(因为部分厂商自己修改了 Resources 的类名)，一个是需要有一个替换原来Resources的过程，这样就需要涉及到很多地方，从hookResources的实现里看，替换了四处地方，在尽量少的 hook 原则下这样的情况还是尽量避免的。

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

## Activity启动过程中对资源的处理

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
