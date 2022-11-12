# EventBus

Android可以使用基于发布、订阅的框架

> 观察者模式：定义了对象之间的一对多依赖，但一个对象改变状态时，它的所有依赖着都会收到通知并自动更新。

# Subscribe 注解

```java
public @interface Subscribe {
.t}
```

# 获取EventBus

```java

EventBusEventBus(EventBusBuilder builder) {
    logger = builder.getLogger();
    //Map<Class<?>, CopyOnWriteArrayList<Subscription>>
    //以event为key，订阅列表为value，是一个线程安全的容器
    subscriptionByEventType = new HashMap<>();
		stickyEvents = new ConcurrentHashMap<>();
}
```

# 构造方法

```java
EventBus(EventBusBuilder builder) {
//（1）start
 //Map<Class<?>, CopyOnWriteArrayList<Subscription>>
    //以event为key，订阅列表为value，是一个线程安全的容器
	subscriptionsByEventType = new HashMap<>();
	typesBySubscriber = new HashMap<>();
	stickyEvents = new ConcurrentHashMap<>();
//（1） end

// (2) 
	mainThreadSupport = builder.getMainThreadSupport();
  mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
// (3)
	backgroundPoster = new BackgroundPoster(this);
// （4）
	asyncPoster = new AsyncPoster(this);
// （5）
	indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
  subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
	                builder.strictMethodVerification, builder.ignoreGeneratedIndex);

}
```

1.  创建的3个对象都是 map 集合
2.  创建了一个 mainThreadPoster 对象，该对象主要用来将线程模型为 MAIN 和 MAIN_ORDERED 的事件加入到队列中，然后进行处理。
    1.  创建了一个 HandlerPoster。
3.  创建了一个 BackgroundPoster 对象，这对象主要用来将线程模型为 BACKGROUND 的事件加入对队列中，然后进行处理。
4.  创建一个 AsyncPoster 对象，该对象主要用来将线程模型为 ASYNC 的事件加入到队列中，然后进行处理。
5.  创建了一个 SubscribeMethodFinder 对象，该对象主要用来获取注册类上所有的订阅方法的集合。
6.  从EventBusBuilder 中取出一个创建好的线程池（CachedThreadPool）赋值给 executorService。

# 注册

```java
/**
 * 将给定对象注册用来接收event，当不在需要观察event的变化时，必须调用unregister
 * 同时，注册的对象内部必须有Subscibe注解的方法
 */
//EventBus
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
（1）
 //subscriberMethodFinder当前对象注册的所有事件监听的方法
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
（2）
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

1.  EventBus#register()中的关注点（1）—findSubscriberMethod()

获取注册类上所有订阅方法的集合，也就是加了`@Subscribe`注解的那些方法。

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
（1）
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
（2）
    if (ignoreGeneratedIndex) {
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
（3）
        subscriberMethods = findUsingInfo(subscriberClass);
    }
订阅方法为空，说明注册类中不存在@Subscribe 注解的方法
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
（4）
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```

1.  查找缓存的订阅方法。
2.  默认是 false。
3.  查找所有订阅方法。
    
    ```java
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            //反射获取注册类中的所有方法
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            
            try {
                methods = findState.clazz.getMethods();
            } catch (LinkageError error) { // super class of NoClassDefFoundError to be a bit more broad...
                String msg = "Could not inspect methods of " + findState.clazz.getName();
                ....
                
            }
            findState.skipSuperClasses = true;
        }
    		//循环遍历所有方法
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
    										//获取方法上的第一个参数，也就是事件类型
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
    										//获取线程模式
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
    										//将方法、事件类型、线程模式、优先级、是否是粘性事件封装到 SubscribeMethod 对象，然后添加到 FindState 类中的 subscriberMethod 集合中存起来
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
    ```
    
4.  找到订阅方法后缓存到集合中。
5.  **EventBus#register 中的关注点（2）—subscribe()**
    
    主要是循环获取每个订阅方法进行订阅，实际内部只是给 subscriptionByEventType、typesBySubscriber 集合添加数据。
    
    ```java
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    				//获取事件类型
            Class<?> eventType = subscriberMethod.eventType;
    				（1）
            Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    				（2）start
            CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
            if (subscriptions == null) {
                subscriptions = new CopyOnWriteArrayList<>();
                subscriptionsByEventType.put(eventType, subscriptions);
            } else {
                if (subscriptions.contains(newSubscription)) {
                    throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                            + eventType);
                }
            }
    				（2） end
            int size = subscriptions.size();
    				（3）
            for (int i = 0; i <= size; i++) {
                if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                    subscriptions.add(i, newSubscription);
                    break;
                }
            }
    				（4）start
            List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
            if (subscribedEvents == null) {
                subscribedEvents = new ArrayList<>();
                typesBySubscriber.put(subscriber, subscribedEvents);
            }
            subscribedEvents.add(eventType);
    				（4）end
    				
    					//（5）
            if (subscriberMethod.sticky) {
                if (eventInheritance) {
            
                    Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                    for (Map.Entry<Class<?>, Object> entry : entries) {
                        Class<?> candidateEventType = entry.getKey();
                        if (eventType.isAssignableFrom(candidateEventType)) {
                            Object stickyEvent = entry.getValue();
                            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                        }
                    }
                } else {
                    Object stickyEvent = stickyEvents.get(eventType);
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        }
    ```
    
    关注点：
    
    - （1）：将注册类和订阅方法封装到 Subscription。
        
    -   （2）：HashMap subscriptionsByEventType 存储一个 key 为事件类型，value 为 Subscription 集合的 HashMap。
        
    -   （3）：根据设置的优先级将 newSubscription 添加到 subscriptions 集合中，也就是优先级越高的排在集合中越前的位置。
        
    -   （4）：HashMap typesBySubscriber 是一个存储 key 为注册类，value 为事件类型集合。typesBySubscriber 的作用是判断某个对象是否注册过，防止重复注册。
        
        ```java
        public synchronized boolean isRegistered(Object subscriber) {
            return typesBySubscriber.containsKey(subscriber);
        }
        ```
        
    -   （5）：粘性事件相关逻辑
        

## 小结

通过反射获取注册类上所有的订阅方法，然后将这些订阅方法进行包装保存到集合中。

# 解除注册

```java
public synchronized void unregister(Object subscriber) {
**根据注册类获取到 typesBySubscriber 集合中保存的事件类型集合**
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
						**看下面**
            unsubscribeByEventType(subscriber, eventType);
        }
				**移出 typesBySubscriber 中保存的事件类型集合**
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
```

```java
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
**按照事件类型获取集合中保存的 Subscription 集合**
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
								**从 Subscription 集合中移出 Subscription**
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

## 小结

注册的时候使用 subscriptionsByEventType 集合保存了所有的订阅方法信息，使用 typesBySubscriber 集合保存了所有事件类型。那么解注册的时候就是为了移出这两个集合中保存的内容。

# 发送普通事件

```java
public void post(Object event) {
		(1) start
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
		（2）end
    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
								（2）
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

-   （1）从 currentPostingThreadState 中获取 PostingThreadState，然后拿到事件队列，最后将传进来的事件保存到该事件队列。
    
    -   ThreadLocal 的好处是保证 PostingThreadState 是线程私有的，其他线程无法访问，避免出现线程安全问题。
-   （2）postSingleEvent
    
    从事件队列中取出一个事件进行发送
    
    ```java
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
    		**是否找到订阅者**
        boolean subscriptionFound = false;
    		（1）
        if (eventInheritance) {
    				（2）
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
    						（3）
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
    				（4）
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
    						（5）
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
    ```
    
    -   （1）：eventInheritance 表示是否需要发送父类与接口中的事件，默认为 true，可配置。
        
    -   （2）：查找所有事件类型，包括当前事件、父类和接口中的事件类型。
        
    -   （3）：根据事件类型发送事件，包括父类、接口。
        
        ```java
        private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
            CopyOnWriteArrayList<Subscription> subscriptions;
            synchronized (this) {
        				（1）
                subscriptions = subscriptionsByEventType.get(eventClass);
            }
            if (subscriptions != null && !subscriptions.isEmpty()) {
                for (Subscription subscription : subscriptions) {
                    postingState.event = event;
                    postingState.subscription = subscription;
                    boolean aborted;
                    try {
        								（2）
                        postToSubscription(subscription, event, postingState.isMainThread);
                        aborted = postingState.canceled;
                    } finally {
                        postingState.event = null;
                        postingState.subscription = null;
                        postingState.canceled = false;
                    }
                    if (aborted) {
                        break;
                    }
                }
                return true;
            }
            return false;
        }
        ```
        
        -   （1）：从 subscriptionsByEventType 集合中取出之前注册的时候保存的 Subscription 集合，然后遍历集合拿到 Subscription。
        -   （2）：根据线程模式判断是否需要切换线程，不需要就反射调用订阅方法。
    -   （4）：根据事件类型只发送当前注册类的事件，忽略父类以及接口。
        
    -   （5）：没有找到订阅者，发送一个 NoSubscriberEvent 事件。
        
    
    ## 根据线程模式判断是否需要切换线程
    
    ```java
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            casePOSTING:
                invokeSubscriber(subscription, event);
                break;
            caseMAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            caseMAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            caseBACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            caseASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
    ```
    
    ### 切换至主线程接收事件
    
    ```java
    if (isMainThread) {
        invokeSubscriber(subscription, event);
    } else {
        mainThreadPoster.enqueue(subscription, event);
    }
    ```
    
    为了避免频繁的向主线程 sendMessage，EventBus 的做法是在一个消息里尽可能多的处理更多的消息事件，所以使用 while 循环，持续从消息队列 queue 中获取消息
    
    ```java
    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
    避免长期占有主线程，间隔10ms 会重新发送 sendMessage，用于让出主线程的执行权，避免造成 UI 卡顿和 ANR
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
    ```
    
    ### 切换至子线程执行
    
    BACKGROUND 会区分发生事件的线程，是否是主线程，非主线程直接分发事件
    
    ```java
    if (isMainThread) {
        backgroundPoster.enqueue(subscription, event);
    } else {
        invokeSubscriber(subscription, event);
    }
    ```
    
    维护了一个用链表实现的消息队列，通过 synchronized 同步锁来保证队列数据的线程安全，同时利用 volatile 标识的 executorRunning 来保证不同线程下看到的执行状态是可见的。
    
    ```java
    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!executorRunning) {
                executorRunning = true;
                eventBus.getExecutorService().execute(this);
            }
        }
    }
    ```
    
    AsyncPoster没有做任何特殊的处理，所有的事件，都是无脑的抛给 EventBus 的线程池去处理，这也保证了，无论发生事件的线程和接收事件的线程，必然是不同的。
    
    **两者的区别：**
    
    -   BACKGROUND 同一时间，只会利用一个子线程，用来循环从事件队列中获取事件并进行处理，也就是说前面的事件的执行效率，会影响后续事件的执行。所以如果你追求执行的效率，立刻马上就要执行的事件，可以使用 ASYNC
    -   ASYNC 会无脑向线程池 executorService 发送任务，而这个线程池，如果你不配置的话，默认情况下使用的是 Executors 的 newCachedThreadPool 创建的
    
    # 发送粘性事件
    
    ```java
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }
    ```
    
    将事件保存到 stickyEvents 集合中，然后调用 post 方法发送事件，如果还没有注册，不会执行后续的发送。
    
    在注册的时候有如下逻辑：
    
    ```java
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
    								关注点
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
    
    ```
    
    如果是粘性事件，则从 stickyEvents 集合中取出事件，然后调用 checkPostStickyEventToSubscription 方法：
    
    ```java
    private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
            // --> Strange corner case, which we don't take care of here.
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }
    ```
    
    # 总结
    
    里面用到了 ThreadLocal、反射、设计模式等
    
    1.  优点
        
        事件发布/订阅框架，主要用来简化 Activity、Fragment、Service、线程等之间的通讯。优点是开销小、使用简单、以及解耦事件发送者和接受者。
        
    2.  为什么要使用 EventBus 来替代广播
        
        -   广播：广播是重量级的，消耗资源较多的方式。如果不做处理也是不安全的。
        -   EventBus：开销小、使用简单、以及解耦事件发送者和接收者。