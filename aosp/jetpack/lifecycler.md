
# Lifecycler
基本思路：把Activity和Fragment当作观察者，把使用Lifecycle的当作观察者，在通过A/F的生命周期通知Lifecycle即可。
## 增加观察者
### addObserver 初始化
```java
//LifecycleRegistry****
public void addObserver(@NonNull LifecycleObserver observer) {  
    enforceMainThreadIfNeeded("addObserver");  
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;  
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);  
//mObserverMap是一个保存LifecycleObserver观察者对象的map，负责保存该被观察者当前所有的观察者。
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);  
  
    if (previous != null) {  
        return;  
    }  
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();  
    if (lifecycleOwner == null) {  
        // it is null we should be destroyed. Fallback quickly  
        return;  
    }  
  
    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;  
    State targetState = calculateTargetState(observer);  
    mAddingObserverCounter++;  
    while ((statefulObserver.mState.compareTo(targetState) < 0  
            && mObserverMap.contains(observer))) {  
        pushParentState(statefulObserver.mState);  
        final Event event = Event.upFrom(statefulObserver.mState);  
        if (event == null) {  
            throw new IllegalStateException("no event up from " + statefulObserver.mState);  
        }  
        statefulObserver.dispatchEvent(lifecycleOwner, event);  
        popParentState();  
        // mState / subling may have been changed recalculate  
        targetState = calculateTargetState(observer);  
    }  
  
    if (!isReentrance) {  
        // we do sync only on the top level.  
        sync();  
    }  
    mAddingObserverCounter--;  
}
```
#### 观察者集合中存储的value
```java
static class ObserverWithState {  
    State mState;  
    LifecycleEventObserver mLifecycleObserver;  
  
    ObserverWithState(LifecycleObserver observer, State initialState) {  
//observer的装饰类，生成新的观察者
        mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);  
        mState = initialState;  
    }  
  
    void dispatchEvent(LifecycleOwner owner, Event event) {  
        State newState = event.getTargetState();  
        mState = min(mState, newState);  
        mLifecycleObserver.onStateChanged(owner, event);  
        mState = newState;  
    }  
}
```

```java
@NonNull  
//Lifecycling
static LifecycleEventObserver lifecycleEventObserver(Object object) {  
	boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;  
boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;  
if (isLifecycleEventObserver && isFullLifecycleObserver) {  
    return new FullLifecycleObserverAdapter((FullLifecycleObserver) object,  
            (LifecycleEventObserver) object);  
}  
if (isFullLifecycleObserver) {  
    return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);  
}
    
    return new ReflectiveGenericLifecycleObserver(object);  
}
```
```java
class ReflectiveGenericLifecycleObserver implements LifecycleEventObserver {  
    private final Object mWrapped;  
    private final CallbackInfo mInfo;  
  
    ReflectiveGenericLifecycleObserver(Object wrapped) {  
        mWrapped = wrapped;  
//保存了方法反射内容和注解信息到info中，
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());  
    }  
  
    @Override  
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Event event) {  
        mInfo.invokeCallbacks(source, event, mWrapped);  
    }  
}
```
当观察者通过实现LifecycleEventObserver、FullLifecycleObserver(DefaultLifecycleObserver extends FullLifecycleObserver)或者两者的共同子类来实现的时候。直接向下转型的结果，得到对应的观察者对象。不再需要通过反射来获得。
## 关联生命周期的源码
AppCompatActivity的超类ComponentActivity实现了LifecycleOwner接口。完成了观察者模式的构建。
```java
//ComponentActivity#onCreate
ReportFragment.injectIfNeededIn(this);
```
### API<29 ReportFragment
```java
//ReportFragment
@Override  
public void onResume() {  
    super.onResume();  
    dispatchResume(mProcessListener);  
    dispatch(Lifecycle.Event.ON_RESUME);  
}

private void dispatch(@NonNull Lifecycle.Event event) {  
    if (Build.VERSION.SDK_INT < 29) {  
        // Only dispatch events from ReportFragment on API levels prior  
        // to API 29. On API 29+, this is handled by the ActivityLifecycleCallbacks        // added in ReportFragment.injectIfNeededIn
		dispatch(getActivity(), event);  
    }  
}

@SuppressWarnings("deprecation")  
static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {  
    if (activity instanceof LifecycleRegistryOwner) {  
//最终都会执行handleLifecycleEvent(event)方法
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);  
        return;  
    }  
  
    if (activity instanceof LifecycleOwner) {  
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();  
        if (lifecycle instanceof LifecycleRegistry) {  
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);  
        }  
    }  
}
```
api<29，通过Activity通过绑定一个空的Fragment，从而执行Fragment的生命周期的时候执行所有标记过当前Activity生命周期的方法。通过状态机进行分发之后，执行onStateChanged()方法。在api<29的情况下，通过反射执行invokeCallback->mMethod.invoke
api>29，分发原理进行了改变。通过Application的ActivityLifecyclecallbacks接口进行的分发。第二是执行onStateChange方法，不通过反射执行，直接调用LifecycleEventObserver的onStateChanged。
## Lifecycle中的状态机管理生命周期
![[Pasted image 20220528214736.png]]
通过两个枚举构建起状态机模型。
状态机的好处就是可以简化条件语句的执行。
在状态机的模型之下，通过mState、mActive等变量判断当前的生命周期状态。
