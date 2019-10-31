
# Lifecycler

```java
public class LifecycleRegistry extends Lifecycle {
    // 上文在dispatchEvent前后进行的pushParentState和popParentState就是为了解决这个问题：
    // 如果在onStart中注销了observer又重新注册，这时重新注册的observer初始的状态为INITIALIZED；
    // 而如果想执行ON_START的对应回调，需要newObserver处于CREATED状态（之前的状态因为removeObserver，不存在于mObserverMap中）
    // mParentStates的作用就是为newObserver提供了CREATED这个状态
    private ArrayList<State> mParentStates = new ArrayList<>();
}
```

1. 假设当前集合中所有ObserverWithState元素都处于CREATED状态。此时接着收到了一个ON_START事件，从图可以看出，接下来应该是要转换到STARTED状态。由于STARTED大于CREATED，所以会执行forwardPass方法。forwardPass里调用 upEvent(observer.mState)，返回从CREATED往上到STARTED需要发送的事件，也就是ON_START，于是ON_START事件发送给了观察者。
2. 假设当前 LifecycleRegistry的mState处于RESUMED状态。然后调用addObserver方法新添加一个LifecycleObserver，该observer会被封装成ObserverWithState存进集合中，此时这个新的ObserverWithState处于INITIALIZED状态，由于RESUMED大于INITIALIZED，所以会执行forwardPass方法。ObserverWithState的状态会按照 INITIALIZED -> CREATED -> STARTED -> RESUMED 这样的顺序变迁。
