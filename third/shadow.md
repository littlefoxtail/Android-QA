# Shadow

宿主应用打包了很简单的一些接口，并在Manifest中注册了壳子代理，还打包了插件管理器（manager）的动态升级逻辑。

- manager负责下载、安装插件，还带有一个动态的View表达Loading态。

插件中的业务App和loader、runtime是同一个版本代码编译出来的

- loader可以包含一些业务逻辑，针对业务进行特殊处理。loader是多个实例的，因此同一个宿主中可以有多种不同的loader实现。
- runtime

## 设计原理解析

所有的插件框架在解决的问题都不是如何动态加载类，而是动态加载的Activity没有在AndroidManifest中注册。如果Android系统没有AndroidManifest的限制，那么所有的插件框架都没有存在的必要了。因为Java语言本身就支持动态更新实现的能力。
动态化

Shadow采用了用一个壳子代理转调插件组件的技术手段实现组件生命周期。同时，反过来插件Activity想调用的父类方法，通过中间层`ShadowActivity`转调回壳子Activity。

全动态指的就是除了插件代码之外，插件框架本身的所有逻辑也都是动态的。Shadow将框架分为四部分：Host、Manager、Loader和Runtime。其中除Host外，Manager、Loaded、Runtime都是动态的。

## 解决Activity生命周期的方法

1. 是用代理Activity作为壳子注册在宿主中真正运行起来，然后让它持有插件Activity，想办法在收到系统的生命周期方法调用时转调插件Activity的对应生命周期方法。
2. 是Hack修改宿主PathClassLoader，让它能在收到系统查询AndroidManifest中注册的Activity的类时返回插件的Activity类。

旧框架就是代理Activity转调插件Activity的方案，

### Activity实现

1. 替换插件`Activity`的父类。
2. 宿主中如何启动插件`Activity`。
3. 插件中如何启动插件`Activity`。

#### 替换插件`Activity`的父类

插件开发的时候，插件的Activity还是正常继承Activity，在打包的时候，通过Transform替换其父类为ShadowActivity

```kt
class ApplicationTransform : SimpleRenameTransform ()
```

#### 宿主中如何启动插件Activity

![shadow启动插件Activity](/img/shadow启动插件Activity.jpg)

经过一些列的流转进入`convertActivityIntent`，将插件intent转化成宿主的intent，然后调用系统的context.startActivity启动插件

```java
// FastPluginManager
public Intent convertActivityIntent(..) {
    // 创建mPluginLoader
    loadPlugin(installedPlugin.UUID, partKey);
    // 调用Application onCreate方法
    mPluginLoader.callApplicationOnCreate();
    // 转化插件intent为代理Activity intent
    mPluginLoader.convertActivityIntent();


}
```

![shadow对应图](/img/shadow_Binder对应.png)

这里的`mPluginLoader.convertActivityIntent`相当于调用了`DynamicPluginLoader.convertActivityIntent`

```kt
//DynamicPluginLoader
fun convertActivityIntent(pluginActivityIntent: Intent): Intent? {
    return mPluginloader.mComponentManager.convertPluginActivityIntent(pluginActivityIntent)
}
```

代理Activity为`PluginDefaultProxyActivity`

中间通过`HostActivityDelegate`

```kt
//ShadowActivityDelegate
override fun onCreate(savedInstanceState: Bundle?) {
    //设置application，resources等等
    mDi.inject(this, partKey)
    //创建插件资源
    mMixResources = MixResources(mHostActivityDelegator.superGetResources(), mPluginResources)
    //设置插件主题
    mHostActivityDelegator.setTheme(pluginActivityInfo.themeResource)

    //创建插件Activity
    initPluginActivity(pluginActivity)
    //调用插件activity onCreate
    pluginActivity.onCreate(pluginSavedInstanceState)

}
```

#### 插件中如何启动插件Activity

![启动插件的Activity](/img/shadow_启动插件的Activity.png)

插件Activity会在打包过程中替换其父类为`ShadowActivity`

```kt
// ShadowContext
public void startActivity(..) {
    final Intent pluginIntent = new Intent(intent);
    final boolean success = mPluginComponentLauncher.startActivity(this, pluginIntent);
}

// ComponentManager
override fun startActivity(shadowContext: ShadowContext, pluginIntent: Intent) {
    return if (pluginIntent.isPluginComponent()) {
        shadowContext.superStartActivity(pluginIntent.toActivityContainerIntent(), option)
        true
    } else {
        false
    }
}
```

通过调用`toActivityContainerIntent`转换intent为代理Activity的intent，然后调用系统的`startActivity`启动代理Activity

## Service实现

### 插件中如何启动

```java
// ShadowContext
public ComponentName startService(Intent service) {
    if (service.getComponent() == null) {
        return super.startService(service);
    }
    Pair<Boolean, ComponentName> ret = mPluginComponentLauncher.startService(this, service);
    if (!ret.first)
        return super.startService(service);
    return ret.second;
}
```

也是调用`mPluginComponentLauncher.startService`

```kt
// ComponentManager
override fun startService(context: ShadowContext, service: Intent): Pari<Boolean, ComponentName> {
    if (service.isPluginComponent()) {
        val component = mPluginServiceManager!!.startPluginService(service)
    }
}

//PluginServiceManager
fun startPluginService(intent: Intent): ComponentName? {
    val componentName = intent.component
    if (!AliveServiceMap.containsKey(componentName)) {
        //不存在则创建
        val service = createServiceAndCallOnCreate(intent)
        mAliveServicesMap[componentName] = service
        //通过startService启动集合
        mServiceStartByStartServiceSet.add(componentName)
    }
}

```

## 框架自身动态化

1. 抽象接口类
2. 在插件中实现工厂类
3. 通过工厂类动态创建接口的实现



## DynamicPluginManager

位于Host模块中，它是启动插件的入口。它负责加载Manager插件，并实例化PluginManagerImpl，之后将处理逻辑交给PluginManagerImpl。

DynamicPluginManager#enter

- 加载Manager插件
- 通过反射实例化Manager内的PluginManagerImpl
- 将处理逻辑委托给PluginManagerImpl的enter

```java
// 创建PluginLoaderImpl的工厂类
private final static String  sLoaderFactoryImplClassName = "com.tencent.shadow.dynamic.loader.impl.LoaderFactoryImpl";

// ManagerImplLoader
PluginManagerImpl load() {
    // 1. 加载Manager插件，这个ClassLoader很神奇，创建插件ClassLoader
    ApkClassLoader apkClassLoader = new ApkClassLoader(
        installedApk,//Manager插件apk
        getClass().getClassLoader(),
        loadWhiteList(installedApk),//ClassLoader白名单
    )
    //支持Resource相关和ClassLoader，允许Manager插件使用资源
    Context pluginManagerContext = new ChangeApkContextWrapper(..);
    // 2. 反射得到Manager中的类com.tencent.shadow.dynamic.impl.ManagerFactoryImpl的实例，调用buildManager生成PluginManagerImpl。
    ManagerFactory managerFactory = apkClassLoader.getInterface(ManagerFactory.class, MANAGER_FACTORY_CLASS_NAME);
    return managerFactory.buildManager(pluginManagerContext);
}
```

工厂类和`PluginLoaderImpl`的实现都在插件中，达到了框架自身的动态化。

## PluginProcessService

类也位于Host模块中，Shadow采用了跨进程的设计，将宿主和插件的进程区分开。插件进程的创建就是通过启动PluginProcessService完成的。Binder代理分别是宿主进程中的PpsController和插件进程中的UuidManager。

- 宿主通过PpsController控制PluginProcessService加载Runtime、Loader。
- 插件通过UuidManager获取Runtime、Loader、插件的安全信息。

### Runtime的动态化实现

```java
// PluginProcessService
void loadRuntime(String uudi) {
    InstalledApk installedRuntimeApk = new InstalledApk(..);
    boolean loaded = DynamicRuntime.loadRuntime(installedRuntimeApk);
}
```

loadRuntime主要完成两件事：

1. 通过mUuidManager获取Runtime的安全信息。之前说过Runtime的安装是在Manager中完成的，而Manager运行在宿主进程中，因此需要Binder通信。
2. 调用DynamicRuntime.loadRuntime()加载Runtime

```java
// DynamicRuntime
public static boolean loadRuntime() {
    if (TextUtils.equals(apkPath, installedRuntimeApk.apkFilePath)) {
        return false;
    } else {
        recoveryClassLoader();
    }
    hackParentToRuntime(installedRuntimeApk, contextClassLoader);
    return true;
}
```

Runtime主要是一些壳子类，例如壳子Activity系统启动插件中的Activity时，其实是启动这些壳子Activity。这就是保住系统的PathClassLoader必须找到壳子Activity。简单的就是利用双亲委派模型，把PathClassLoader的父加载器设置成RuntimeClassLoader。

### Loader的动态化实现

Loader的加载过程，也是在PluginProcessService中完成，主要流程：

1. 获取Loader的安全信息。
2. 在LoaderImplLoader加载Loader插件。

```java
void loadPluginLoader(String uuid) {
    // 获取Loader的安全信息
    installedApk = mUuidManager.getPluginLoader(uuid);
    // 交给LoaderImplLoader.load()加载loader
    PluginLoaderImpl pluginLoader = new LoaderImplLoader().load(installedApk, uuid, getApplicationContext());

}
```

## 安装插件

由于Loader和Runtime的动态化，在发布插件时，还要发布Loader和Runtime。

## 加载插件

Manager中PluginLoader通过Binder通信向Loader中的PluginLoaderBinder发送消息。PluginLoaderBinder收到消息后，委托给PluginLoaderImpl处理。

![加载插件](/img/加载插件.png)

1. 利用ClassLoader加载插件APK。
2. 利用PackageManager获取插件信息。
3. 创建存储插件信息的PluginPackageManager，提供获取插件ActivityInfo、PackageInfo等信息的功能。
4. 创建查找插件资源的Resources
5. 为当前插件创建ShadowApplication，为插件Application的生命周期做准备。
6. 缓存以上内容

```kt
fun loadPlugin() {
    val buildClassLoader = executorService.submit(
        Callable {
            lock.withLock {
                LoadApkBloc.loadPlugin(..)
            }
        }
    )
    //利用packageManager获取插件信息
    val getPackageInfo = executorService.submit(Callable {
        val archiveFilePath = installedApk.apkFilePath
        val packageManager = hostAppContext.packageManager
        val packageArchiveInfo = packageManager.getPackageArchiveInfo(
                        archiveFilePath,
                        PackageManager.GET_ACTIVITIES
                                or PackageManager.GET_META_DATA
                                or PackageManager.GET_SERVICES
                                or PackageManager.GET_PROVIDERS
                                or PackageManager.GET_SIGNATURES
                )
    })
    //创建存储插件信息的PluginPackageManager
    val buildPackageManager = executorService.submit(
        Callable{
            val packageInfo = getPackageInfo.get();
            val hostPackageManager = hostAppContext.packageManager
            PluginPackageManagerImpl(hostPackageManager, packageInfo, allPluginPackageInfo)
        }
    )
    //创建查找插件资源的Resources
    val buildResources = executorService.submit(Callable {
        val packageInfo = getPackageInfo.get()
        CreateResourceBloc.create(packageInfo, installedApk.apkFilePath, hostAppContext)
    })
    val buildApplication = executorService.submit(Callable {
        val pluginClassLoader = buildClassLoader.get();
        val resources = buildResources.get();
        val pluginInfo = buildPluginInfo.get();
        CreateApplicationBloc.createShadowApplication(..);
    })
    val buildRunningPlugin = executorService.submit {
        val pluginPackageManager = buildPackageManager.get()
        val pluginClassLoader = buildClassLoader.get()
        val resources = buildResources.get()
        ..
    }
}
```

## 插件ClassLoader的实现

插件中的ClassLoader必须可以加载宿主和其他插件中的类，这样插件才能与宿主或者其他插件交互

### 加载宿主中的类

只能加载白名单中配置了的宿主类

```kt
class PluginClassLoader(dexPath: String,
    optimizedDirectory: File?,
    librarySearchPath: String?,
    parent: ClassLoader, //parent是宿主PathClassLoader
    private val specialClassLoader: ClassLoader?,//是宿主PathClassLoader的父加载器
    hostWhiteList: Array<String>? //hostWhiteList表示插件可以加载宿主类的白名单
    ) : BaseClassLoader(dexPath, optimizedDirectory, librarySearchPath, parent) {

    override fun loadClass(className: String, resolve: Boolean): Class<*> {
        if (specialClassLoder == null) {
            //
            return super.loadClass(className, resolve)
        } else if (className.inPackage(allHostWhiteList)) {

            return super.loadClass(className, resolve)
        } else {
            var clazz: Class<*>? = findLoadedClass(className)
            if (clazz == null) {
                clazz = findClass(className)!!
                if (clazz == null) {
                    clazz = specialClassLoader.loadClass(className)!!
                }
            }
            return clazz
        }
    }

}
```

### 加载其他插件中的类

shadow在加载插件时，会先判断它所依赖的插件是否已经加载。如果依赖的插件已经全部加载，则把加载这些插件的PathClassLoader组装到一个CombineClassLoader中。这个CombineClassLoader就是当前插件PathClassLoader的父加载器。

在加载插件时，先加载其依赖插件。

```kt
//LoadApkBloc
fun loadPlugin(..) {
    if (dependsOn == null || dependsOn.isEmpty()) {
        //没有依赖，直接创建一个相关的插件classLoader
        return PluginClassLoader(..);
    } else if (dependsOn.size == 1) {
        if (puginParts == null) {
            throw LoadApkException("加载" + loadParameters.partKey + "时它的依赖" + partKey + "还没有加载");
        } else {
            //只有一个依赖插件
            return PluginClassLoader(apk.absolutePath,
                                    odexDir,
                                    installedApk.libraryPath,
                                    pluginParts.classLoader,
                                    null,
                                    loadParameters.hostWhiteList)
        }
    } else {
        val dependsOnClassLoaders = dependsOn.map {
            val pluginParts = pluginPartsMap[it]
            if (pluginParts == null) {
                throw LoadApkException("加载" + loadParameters.partKey + "时它的依赖" + it + "还没有加载")
            } else {
                pluginParts.classLoader
            }.toTypeArray()
            val combineClassLoader = CombineClassLoader(dependsOnClassLoaders, hostParentClassLoader)
            return PluginClassLoader(..)
        }
    }
}
```

### 创建查找插件资源的Resources

```kt
object CreateResourceBloc {
    fun create(..): Resources {
        //这里还考虑用宿主context初始化一个webview呢

        val packageManager = hostAppContext.packageManager
        packageArchiveInfo.applicationInfo.publicSrouceDir = archiveFilePath
        packageArchiveInfo.applicationInfo.sourceDir = archiveFilePath
        packageArchiveInfo.applicationInfo.sharedLibraryFiles = hostAppContext.applicationInfo.sharedLibraryFiles
        return packageManager.getResourcesForApplication(packageArchiveInfo.applicationInfo)
    }
}

```

将插件apk的路径赋值给publicSourceDir和sourceDir，利用PackageManager.getResourcesForApplication()创建一个新的Resources


