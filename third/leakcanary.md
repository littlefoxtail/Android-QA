# LeakCanary

1. LeakCanary.install()返回了一个RefWatcher。RefWatcher.queue是一个ReferenceQueue实例。
2. 通过继承WeakReference的KeyedWeakReference，增加了key+name字段从而支持对象的唯一起名（通过UUID）和名字获取。
3. 在RefWatcher.watch()时把UUID放入RefWatcher.retainedKeys，并把对象跟RefWatcher.queue关联。
4. 通过Runtime.gc()和System.runFinalization()进行GC。
5. 在RefWatcher.removeWeaklyReachableReferences()调用queue.poll()取出KeyedWeakReference，并从retainedKeys中删除。所以，如果retainedKeys中key仍存在，说明对象未被垃圾回收。反之则已经垃圾回收。
6. 如果对象未被垃圾回收，则执行分析。

## 业务层

1. Application LeakCanary.install(this)；
2. 基于Activity/Fragment onDestory()中 RefWatcher.watch(this)；

## Api层

LeakCanary、RefWatcher，组成了api层。

## LeakCanary2

不需要初始化了

```kt
internal class LeakSentryInstalledr : ContentProvider() {
    fun onCreate() : Boolean {
        CanaryLog.logger = DefaultCanaryLog()
        val application = context!!.applicationContext as Application 
        InternalLeakSentry.install(application)
        return true
    }
}
```

### 主线1 install()

```kt
internal object InternalLeakSentry {
    ActivityDestroyWatcher.install(
        application, refWatcher, configProvider
    )
    FragmentDestroyWatcher.install(
        application, refWatcher, configProvider
    )
}
```

```kt
internal class ActivityDestroyWatcher private constructor (..) {
    private val lifecycleCallbacks = object : ActivityLifecycleCallbacksAdapter() {
        override fun onActivityDestroyed(activity : Activity) {
            if (configProvider().watchActivities) {
                refWatcher.watch(activity)
            }
        }
    }
}
```

### 主线2 watch()

```kt
class RefWatcher constructor(..) {
    @Synchronized fun watch(..) {
        if (!isEnabled()) {
            return
        }
        //移除弱引用
        remoteWeaklyReachableReferences()
        //传入了referenceQueue，这表明，当对象呗标记GC时，会将对象加入到queue中
        val reference = KeyedWeakReference(watchedReference, key, referenceName, watchUptimeMillis, queue);

        watchedReferences[key] = reference
        checkRetainedExecutor.execute {
            moveToRetained(key)
        }
    }

    @Synchronized private fun moveToRetained(key: String) {
        removeWeaklyReachableReferences()
        val retainedRef = watchedReferences.remove(key)
        if (retainedRef != null) {
            //有泄露的可能
            retainedReferences[key] = retainedRef
            onReferenceRetained()
        }
    }

    private fun removeWeaklyReachableReferences() {
        val ref : KeyedWeakReference?
        do {
            ref = queue.poll() as KeyedWeakReference?
            if (ref != null) {
                val removedRef = watchedReferences.remove(ref.key)
                if (removedRef == null) {
                    retainedReferences.remote(ref.key)
                }
            }
        } while (ref != null)
    }
}
```

```kt
internal class HeapDumpTrigger(..) {
    private fun checkRetainedInstance(reason: String) {

    }
}

```

## 总结

LeakCanary对对象泄漏的监听，当对象需要销毁时，大致可以分为三步：

1. 通过弱引用队列，看对象是否被回收
2. 强制gc后，再次通过弱引用队列，看对象是否被回收
3. 通过haha库对内存进行快照分析，判断是否被回收