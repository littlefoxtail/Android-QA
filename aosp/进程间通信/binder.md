# 一、概述

Android系统中，每个应用程序是由Android的Activity,Service,Broadcast,ContentProvider
这四剑客中一个或多个组合，四大组件涉及的多进程间的通信底层都是依赖于Binder IPC机制。

1. 从IPC角度来说：Binder是Android中的一种跨进程通信方式，该通信方式在linux中没有，是Android独有
2. 在Android Driver层：Binder还可以理解一种虚拟的物理设备，它的设备驱动是/dev/binder
3. 从Adnroid Native层：Binder是创建Service Manager以及BpBinder/BBinder模型，搭建与binder驱动的桥梁
4. 从Android Framework层：Binder是各种Manager(ActivityManager、WindowManager等)和相应xxxManagerService的桥梁
5. 从Android APP层：Binder是客户顿和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务

## 从内核空间与用户空间角度

![binder_structure](/img/binder_structure.png)
每一个Android应用都是一个独立的Android进程，它们拥有自己独立的虚拟地址空间，应用进程处于用户空间之中，彼此之间互相独立，不能共享。但是内核空间却是可以共享的，Client进程向Server进程通信，就是利用进程间可以共享的内核地址空间来完成底层的通信工作。Client进程与Server端进程往往采用ioctl等方法跟内核空间的驱动进行交互

## 从Java与C++分层的角度

![binder_detail_structure](/img/binder_detail_structure.png)

整个Binder通信机制中，从大的方面可以分为:

- Framework Binder
- Native Binder

Framework Binder最终通过JNI调用Native Binder的功能，它们在架构上的设计都是C/S架构

关键角色：

- Client：客户端
- Server：服务端
- ServiceManager：C++层的ServiceManager，Binder通信机制的大管家，Android进程间通信机制Binder的守护进程
- Binder Driver：Binder驱动

整个流程：

1. Server进程将服务注册到ServiceManager
2. Client进程向ServiceManager获取服务
3. Client进程得到的Service信息后，建立Server进程的通信通道，然后就可以和Server进程进行交互了

## Binder通信流程

### Binder原理

Binder通信采用C/S架构，从组件视角来说，包含Client、Server、ServiceManager以及binder驱动，其中ServiceManager用于管理系统中的各种服务。架构图如下所示：
![image](/img/cs.jpg)

可以看出无论是注册服务和获取服务的过程都需要ServiceManager，需要注意的是此处的ServiceManager是指Native层的ServiceManager，并非指framework的ServiceManager。
ServiceManager是整个Binder通信机制的大管家，是Android进程间通信机制Binder的守护进程。

1. 注册服务(addService)：Server进程要先注册Service到ServiceManager。该过程：Server是客户端，ServierManager是服务端。
2. 获取服务(getService)：Client进程使用某个Service前，须向ServiceManager中获取相应的Service。该过程：Client是客户端，ServiceManager是服务端。
3. 使用服务：Client根据得到的Service信息建立与Service所在的Server进程通信的通路，然后就可以直接与Service交互。该过程:client是客户端，server是服务端。

图中的Client,Server,Service Manager之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与Binder驱动进行交互的，从而实现IPC通信方式。其中Binder驱动位于内核空间，Client,Server,Service Manager位于用户空间。

Binder驱动和Service Manager可以看做是Android平台的基础架构，而Client和Serer是Android的应用层。

### C/S模式

BpBinder(客户端)和BBinder(服务端)都是Android中Binder通信相关的代表，它们都从IBinder类中派生而来：

- client端:BpBinder.transact()来发送事务请求;
- server端:BBinder.onTransact()会接收到相应事务。

### 架构

![image](/img/java_binder.jpg)

- 图中红色代表整个framework层binder架构相关组件
  - Binder类代表Server端，BinderProxy类代码Client端；
- 图中蓝色代表Native层Binder架构相关组件
- 上层framework层的Binder逻辑是建立在Native层架构基础之上，核心逻辑都是交予Native层方法来处理。
- framework层的ServiceManager类与Native层的功能并不完全对应，framework层的ServiceManager类实现最终是通过BinderProxy传递给Native层来完成的

### 类图

![image](/img/class_ServiceManager.jpg)

1. ServiceManager:通过getIServiceManager方法获取的是ServiceManagerProxy对象；
    ServiceManager的addService，getService实际工作都交由ServiceManagerProxy的相应方法来处理；
2. ServiceManagerProxy:其成员变量mRemote指向BinderProxy对象，ServiceManagerProxy的addService,getService()方法最终交由mRemote来完成。
3. ServiceManagerNative：其方法asInterface()返回的是ServiceManagerProxy对象，ServiceManager便是借助ServiceManagerNative类来找到ServiceManagerProxy；
4. Binder：其成员变量mObject和方法execTransact()用于native方法
5. BinderInternal：内部有一个GcWatcher类，用于处理和调试与Binder相关的垃圾回收
6. IBinder:接口中常量FLAG\_ONEWAY，客户端利用binder跟服务端通信是阻塞式的，但如果设置了FLAG\_ONEWAY，这成为非阻塞的调用方式，客户端能立即返回，服务端采用回调方式来通知客户端完成情况。另外IBinder接口有一个内部接口DeathDecipient

### Binder类分层

整个Binder从kernel至，native, JNI, Framework层所涉及的全部类
![image](/img/java_binder_framework.jpg)

## 二、初始化
