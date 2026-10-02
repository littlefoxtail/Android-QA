# 插件化

主要就是代理和 hook 系统两种方式。

hook系统一般都用到了一些黑科技，主要有三种：

- Hook App运行的关键点，以达到动态监测是否调用了插件中的代码和资源，比如Hook Instrumentation等
- 动态的加载插件中的类，这分为单Class Loader和多Class Loader技术，这两种技术都用到了反射技术，以达到替换系统的ClassLoader或者是增加自己的Class Path的目的
- 动态的加载插件中的资源，一般会反射调用AssetManager.addAssetPaths

## 名称解释

宿主：
负责加载插件的apk，一般来说就是已经安装的应用本身。
StubActivity：
宿主中的占位Activity，注册在宿主Manifest文件中，负责加载插件Activity。
PluginActivity：
插件Activity，在插件apk中，没有注册在Manifest文件中，需要StubActivity来加载

## android中ClassLoader

创建DexClassLoader实例以后，只要调用其loadClass方法就可以加载插件中的类。

## 插件化需要解决的难点

- Activity
  1. 生命周期如何调用
  2. 如何使用插件中的资源

- Service
  - 生命周期如何调用

- BroadcastReceiver
  1. 静态广播和动态广播的注册

- ContentProvider
  1. 如何注册插件Provider到系统

