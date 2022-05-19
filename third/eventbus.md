# EventBus

Android可以使用基于发布、订阅的框架

> 观察者模式：定义了对象之间的一对多依赖，但一个对象改变状态时，它的所有依赖着都会收到通知并自动更新。

## 源码探究

### 获取EventBus

```java
//EventBus
EventBus(EventBusBuilder builder) {
    logger = builder.getLogger();
    //Map<Class<?>, CopyOnWriteArrayList<Subscription>>
    //以event为key，订阅列表为value，是一个线程安全的容器
    subscriptionByEventType = new HashMap<>();
    
    stickyEvents = new ConcurrentHashMap<>();
}
```

### 注册

```java
/**
 * 将给定对象注册用来接收event，当不在需要观察event的变化时，必须调用unregister
 * 同时，注册的对象内部必须有Subscibe注解的方法
 */
//EventBus
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    //subscriberMethodFinder当前对象注册的所有事件监听的方法
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

### subscribe

```java
//EventBus
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventTye;
    
}
```
