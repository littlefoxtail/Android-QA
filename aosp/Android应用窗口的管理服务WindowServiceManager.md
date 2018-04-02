# WindowManagerService
WindowManagerService，是一个窗口管理系统服务，主要功能如下：
* 窗口管理，绘制
* 转场动画--Activity切换动画
* Z-ordered的维护，Activity窗口显示前后顺序
* 输入法管理
* Token管理
* 系统消息收集线程
* 系统消息分发线程

WindowManagerService是继ActivityManagerService与PackageManagerService之后又一个复杂却十分重要的系统服务

先了解下(Window)是什么？

Android系统中的窗口是屏幕上的一块用于绘制各种UI元素并可以响应用户输入的一快矩形区域。从原理上来讲，窗口的概念是独自占有一个
Surface实例的显示区域。例如Dialog、Activity的界面、壁纸、状态栏以及Toast等都是窗口。

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/ui/WindowManagerService_class.png">

主要角色：
* WindowManager:应用于窗口管理服务WindowManagerService交互的接口
* WindowManagerService:窗口管理服务，继承于IWindowManagerService交互接口
* SurfaceFlinger:SurfaceFlinger服务运行在Android系统的System进程中，它负责管理Android系统的帧缓冲区(Frame Buffer)，Android设备的显示屏被抽象为一个
帧缓冲区，而Android系统中的SurfaceFlinger服务就是向这个帧缓冲区写入内容来绘制应用程序的用户界面
* Surface：每一个显示界面的窗口都是一个Surface
* PhoneWindowManager:实现了窗口的各种策略

WindowManager与WindowManagerService的跨进程通信。Android的各种服务都是基于C/S结构来设计的，系统层提供服务，应用层使用服务。
WindowManager也是一样，它与WindowManagerService的通信都是通过WindowSession来完成的。
1. 首先调用ServiceManager.getService("window")获取WindowManagerService，该方法返回的是IBinder对象，然后调用IWindowManager.Stub.asInterface()方法将WindowManagerService转化为一个IWindowManager对象。
2. 然后调用openSession()方法与WindowManagerService建立一个通信会话，方便后续的跨进程通信。



