# LiveData

```java
public abstract class LiveData<T> {
     @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        // 每次更新 value，都会使 mVersion + 1
        // ObserverWrapper 也有一个字段，叫 mLastVersion
        // 通过比较这两个字段，可以避免重复通知客户（具体在后面会看到）
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }

     // 如果 initiator == null，表示要通知所有的 observer
    // 不等于 null 则只通知 initiator
    private void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            // 在 observer 的回调里面又触发了数据的修改
            // 设置 mDispatchInvalidated 为 true 后，可以让下面的循环知道
            // 数据被修改了，从而开始一轮新的迭代。
            //
            // 比方说，dispatchingValue -> observer.onChanged -> setValue
            //            -> dispatchingValue
            // 这里 return 的是后面那个 dispatchingValue，然后在第一个
            // dispatchingValue 会重新遍历所有的 observer，并调用他们的
            // onChanged。
            //
            // 如果想避免这种情况，可以在回调里面使用 postValue 来更新数据
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                // 调用 observer.onChanged()
                considerNotify(initiator);
                initiator = null;
            } else {
                for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        // 某个客户在回调里面更新了数据，break 后，这个 for 循环会
                        // 重新开始
                        break;
                    }
                }
            }
        // 当某个客户在回调里面更新了数据，mDispatchInvalidated == true
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }
}
```

看过我那篇 lifecycle 源码分析的读者应该对 dispatchingValue 处理循环调用的方式很熟悉了。以这里为例，为了防止循环调用，我们在调用客户代码前先置位一个标志（mDispatchingValue），结束后再设为 false。如果在回调里面又触发了这个方法，可以通过 mDispatchingValue 来检测。

检测到循环调用后，再设置第二个标志（mDispatchInvalidated），然后返回。返回又会回到之前的调用，前一个调用通过检查 mDispatchInvalidated，知道数据被修改，于是开始一轮新的迭代。