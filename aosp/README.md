
- **[View相关](view/README.md)**
- **[Jetpack](jetpack/README.md)**
- **[Android窗口管理](window_manager.md)**
- **[ClassLoader](classloader.md)**
- **[WebView遇到的一些问题和总结](webView.md)**
- **[Handler](handler.md)**
- **[Choreographer控制同步处理输入、动画、绘制](choreographer.md)**
- **[进程与AMS的关系](ams.md)**
- **[应用中的Context](context.md)**
- **[LayoutInflater如何加载布局的](layoutinflater.md)**
- **[动画](animation.md)**
- **[Android中的线程所有知识点](android_thread.md)**
- **[Material Style](color_resource.md)**
- **[Android在编码的时候经常使用到位运算](androidbit.md)**
- **[AsyncTask](asynctask.md)**
- **[ExecutorService中的关闭](executorservice.md)**
- **[内存泄露中的强弱引用](memory.md)**

- TODO
  - **[ActivityManagerService](ams.md)**
  - **[Application not responding](applicationnotresponding.md)**
  - **[Choreographer启动流程](choreographer.md)**
  - **[Fragment](fragment.md)**
  - **[futuretask](futuretask.md)**
  - **[图片缓存处理](image.md)**
  - **[Interpolator](Interpolator.md)**
  - **[Android性能调优](performance.md)**
  - **[自定义View的步骤](customView.md)**
  - **[对话框的源码分析](dialog.md)**
  - **[PriotiryBlockingQueue队列源码分析](priotiryblockingqueue.md)**
  - **[Window的源码](window.md)**

# Android系统

android系统底层采用Linux作为基底，上层采用包含虚拟机的Java层以及Native层，通过系统调用(Syscall)连通系统的内核空间与用户空间。用户空间主要采用C++和Java代码，通过JNI技术打通用户空间的Java层和Native层，从而融为一体

![android_boot](../img/android-boot.jpg)

## Framework层

- Zygote进程，是由init进程通过解析init.rc文件后fork生成的，Zygote进程主要包括：
  - 加载ZygoteInit类，注册Zygote Socket服务端套接字
  - 加载虚拟机
  - preloadClasses
  - preloadResources
- System Server进程，是由Zygote进程fork而来，`System Server是Zygote孵化的第一个进程`，System Server负责启动和管理整个Java framework，包含ActivityManager，PowerManager等服务
- Media Server进程，是由init进程fork而来，负责启动和管理整个C++ framework，包含AudioFlinger，Camera Service，等服务

## APP层

- Zygote进程孵化出的第一个APP进程是Launcher，这是用户看到的桌面App
- Zygote进程还会创建Browser，Phone，Email等App进程，每个App至少运行在一个进程上
- 所有App进程都是由Zygote进程fork生成的

## 通讯方式

### binder

- **[binder的理解](binder.md)**
- **[framework层binder](framework层binder.md)**

### Socket

Socket通信方式也是C/S架构，比Binder简单很多。在Android系统中采用Socket通信方式主要：

- zygote：用于孵化进程，系统进程system_server孵化进程时便通过socket向zygote进程发起请求
- installd：用于安装App的守护进程，上层PackageManagerService很多实现最终都是交给它来完成
- lmkd：lowmemorykiller的守护进程，Java层的LowMemoryKiller最终都是交给它来完成
- adbd：用于服务adb
- logcatd：用于服务logcat
- void：即volume Daemon，是存储类的守护进程，用于负责如USB、Sdcard等存储设置的事件处理

### Handler

Handler只能用于共享内存地址空间的两个线程间通信，即同进程的两个线程间通信。很多时候，Handler是工作线程向UI主线程发送消息，即App应用中只有主线程能更新UI，其他工作线程往往是完成相应工作后，通过Handler告知主线程需要做出相应地UI更新操作，Handler分发相应的消息给UI主线程去完成

由于工作线程与主线程共享地址空间，即Handler实例对象`mHandler`位于线程间共享的内存堆上，工作线程与主线程都能直接使用该对象，只需要注意多线程的同步问题。工作线程通过`mHandler`向成员变量`MessageQueue`中添加新Message，主线程一直处于loop()方法内，当收到信的Message时按照一定规则分发给相应的`handleMessage`来处理