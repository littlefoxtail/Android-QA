# RemoteViews是什么

RemoteView是用来描述一个视图的，它描述的这个视图将显示在另外一个进程中。
RemoteViews并不是一个控件，它仅仅是为了生成控件和修改控件属性提供一系列的方法。

## RemoteViews实现跨进程更新UI

RemoteViews实现跨进程更新UI同样既可以通过AIDL也可以使用BroadcastReceiver

## 深入理解RemoteViews

RemoteViews最常用的两个场景是Notification和AppWidget小部件，因为这两者的界面都运行在其他进程，确切来说它们所属systemServer

问题：

1. RemoteViews为什么可以通过一次IPC实现多个View的操作。
2. 其他进程怎么获取布局文件。
