# 插件化原因
1. 减少升级成本
2. 解决app 65536问题(Multidex方案之前，现在该问题并不是插件化原因)
3. 减少app包大小
4. 快速修改线上BUG或者发布新功能
5. 模块解耦，协同开发

四大组件都是需要在AndroidManifest中注册的，而插件apk中的组件是不可能预先知晓名字，所以需要hack方案类解决。

|组件|解决方案|
|:--:|:---:|
|Activity|在宿主apk中提前占几个坑，通过“欺上瞒下”的方式，启动插件apk的Activity；因为要支持不同的launchMode以及一些特殊的属性，需要占多个坑|
|Service|通过代理Service的方式去分发；主进程和其他进程，VirtualAPK使用两个代理|
|BroadcastReceiver|静态转动态|
|ContentProvider|通过一个代理的Provider进行分发|

- ActivityThread：管理着一个主线程的执行应用程序进程，调度和执行活动，广播和其他操作作为活动管理请求
- Instrumentation: 该类会在应用的任何代码执行前被实例化，用来监控系统与应用的交互。可以以应用的AndroidManifest.xml中<instrumentation>标签来注册一个Instrumentation的实现
- ActivityManager: 这个类提供有关信息，并进行交互activities、services、和包含的process。这个类中的一些方法是用于调试或信息的目的，他们应该不能影响任何运行时的行为。大多数应用程序开发人员不应该需要使用这个类，其中大多数的方法是专门用例。

- DexClassLoader：能够加载自定义的jar/apk/dex
- PathClassLoader：只能加载系统中已经安装过的apk
所以Android系统默认的类加载器为PathClassLoader，而DexClassLoader可以像JVM的ClassLoader一样提供动态加载。

- ActivityManagerService简称AMS，是android内核三大功能之一(另外两个是WindowManagerService和View)
AMS主要提供主要功能
![image](../img/ams)
  1. 统一调试应用的Activity。应用程序要运行Activity，首先会报告给AMS，然后由AMS决定该Activity是否启动
  2. 内存管理。Activity退出后，其所有的进程并不会被立即杀死，从而在下次启动的时候，提高Activity的启动速度。这些Activity只有在内存吃紧的时候，才会被自动杀死
  3. 进程管理。AMS向外提供了查询系统正在运行的进程API

# 加载插件
```java
PluginManager.loadPlugin(apk)
```

```java
PackageParser.Package pkg = PackageParser.parsePackage();
```

# 替换Activity

startActivity->Instrumentation.execStartActivity()
Activity是否存在的校验是发生在AMS端，所以在AMS交互前面