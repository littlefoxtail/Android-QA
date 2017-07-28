# 一.ANR场景
无论是四大组件或者进程等只要发生ANR，最终都调用AMS.appNotResponding()方法：

以下会触发调用AMS.appNotResponding方法：
1. Service Timeout：比如前台服务在20s内执行完成；
2. BroadcastQueue Timeout：比如前台广播在10s内未执行完成
3. InputDispatching Timeout：输入事件分发超时5s，包括按键和触摸事件