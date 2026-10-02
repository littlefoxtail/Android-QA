要点：
1. mContentParent：实际上是ActionBarOverlayLayout，是我们布局的直接父布局
2. Activity的展示过程大概就是atms回掉activity的启动方法，然后会进行初始化PhoneWindow、DecorView。初始化完成后会等待wms回调onResume的逻辑处理。UI关键类ViewRootImpl，onResume中会进行activity以下五个回调方法的处理：onNewIntent、onActivityResult、onRestart、onStart、onResume
# Window的初始化
## PhoneWindow的初始化
ActivitThread中的performLaunchActivity函数中，先创建了Activity，然后调用了Activity的attach函数。
同时也创建了WindowManagerService并绑定到PhoneWindow。

```java
//ActivityThread#performLaunchActivity
Activity activity = null;
try {
    java.lang.ClassLoader cl = appContext.getClassLoader();
    if (appContext.getApplicationContext() instanceof Application) {
        activity = ((Application) appContext.getApplicationContext())
            .instantiateActivity(cl, component.getClassName(), r.intent);
    }
    if (activity == null) {
        activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
    }
    ...

    if (activity != null) {
        activity.attach(...)
    }
}
```

attach函数，先创建了PhoneWindow对象给了mWindow，然后调用其setWindowManager设置其WindowManager，最后调用mWindow的getWindowManager方法作为Activity的mWindowManager成员变量

```java
//初始化 PhoneWindow
mWindow = new PhoneWindow(this, window, activityConfigCallback);
mWindow.setWindowControllerCallback(this);
mWindow.setCallback(this); 
mWindow.setOnWindowDismissedCallback(this);
mWindow.setWindowManager((WindowManager)context.getSystemService(Context.WINDOW_SERVICE), 
    mToken, mComponent.flattenToString(), (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
mWindowManager = mWindow.getWindowManager();
```
# setContentView流程
## DecorView的创建
PhoneWindow的setContentView函数会调用installDecor来创建DecorView对象。

```java
public void setContentView(int layoutResId) {
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestoryed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

在installDecor函数中调用了generateDecor函数来创建按DecorView对象

```java
private void installDecor() {
    if (mDecor == null) {
        mDecor = generateDecor();
        mDecor.setDescendantFousability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
    }
}
```

```java
protected DecorView generateDecor() {
    return new DecorVew(getContext, -1);
}
```

# ActivityThread.handleResumeActivity 
处理Activity可见生命周期
```java
//如果activity的window不为空，没有被finish，且判断到它应该被设置可见则走下面逻辑
if (r.window == null && !a.mFinished && willBeVisible) {  
    r.window = r.activity.getWindow();  
    View decor = r.window.getDecorView();  
    decor.setVisibility(View.INVISIBLE);  
    ViewManager wm = a.getWindowManager();  
    WindowManager.LayoutParams l = r.window.getAttributes();  
    a.mDecor = decor;  
    l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;  
    l.softInputMode |= forwardBit;  
    if (r.mPreserveWindow) {  
        a.mWindowAdded = true;  
        r.mPreserveWindow = false;  
        // Normally the ViewRoot sets up callbacks with the Activity  
        // in addView->ViewRootImpl#setView. If we are instead reusing        // the decor view we have to notify the view root that the        // callbacks may have changed.        ViewRootImpl impl = decor.getViewRootImpl();  
        if (impl != null) {  
            impl.notifyChildRebuilt();  
        }  
    }    if (a.mVisibleFromClient) {  
        if (!a.mWindowAdded) {  
            a.mWindowAdded = true;  
//把decor添加进window
            wm.addView(decor, l);  
        } else {  
            // The activity will get a callback for this {@link LayoutParams} change  
            // earlier. However, at that time the decor will not be set (this is set            // in this method), so no action will be taken. This call ensures the            // callback occurs with the decor set.            a.onWindowAttributesChanged(l);  
        }  
    }  
    // If the window has already been added, but during resume  
    // we started another activity, then don't yet make the    // window visible.} else if (!willBeVisible) {  
    if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");  
    r.hideForNow = true;  
}
```

```java
if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {  
    if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);  
//获取ViewRootImpl
    ViewRootImpl impl = r.window.getDecorView().getViewRootImpl();  
    WindowManager.LayoutParams l = impl != null  
            ? impl.mWindowAttributes : r.window.getAttributes();  
    if ((l.softInputMode  
            & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)  
            != forwardBit) {  
        l.softInputMode = (l.softInputMode  
                & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))  
                | forwardBit;  
        if (r.activity.mVisibleFromClient) {  
            ViewManager wm = a.getWindowManager();  
            View decor = r.window.getDecorView();  
//调用ams更新布局
            wm.updateViewLayout(decor, l);  
        }  
    }  
    r.activity.mVisibleFromServer = true;  
//可见activity数量加1
    mNumVisibleActivities++;  
//控制decor可见
    if (r.activity.mVisibleFromClient) {  
        r.activity.makeVisible();  
    }  
}
```
## performResume


# ViewRootImpl的初始化
## ViewRootImpl

在ActivityThread的handleResumeActivity函数中会调用WindowManager的addView函数

```java
if (r.window == null && !a.mFinished && willBeVisible) {
    r.window = r.activity.getWindow;
    View decor = r.window.getDecorView();
    decor.setVisibility(View.INVISIBLE);
    ViewManager wm = a.getWindowManager();
    WindowManager.LayoutParams l = r.window.getAttributes();
    a.mDecor = decor;
    l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
    l.softInputMode != forwardBit;
    if (a.mVisibleFromClient) {
        if (!a.mWindowAdded) {
            a.mWindowAdded = true;
//添加View
            wm.addView(decor, l);
        }
    }
}
```

最后addView函数在WindowManagerImpl中实现

```java
public void addView(View view, ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

在WindowManagerImpl最后是调用了WindowManagerGlobal的addView函数，在这个函数中新建ViewRootImpl对象，然后调用了ViewRootImpl的
setView函数，这里的View就是Activity的Window对象的DecorView。并且在这个函数中,把view root param这3个保存在mViews mRoots mParams这3个成员变量中了。

```java
//证明了ViewRootImpl是在onResume方法回调之前创建的
root = new ViewRootImpl(view.getContext(), display);
view.setLayoutParams(wparams);

mView.add(view);
mRoots.add(root);
mParams.add(wparams);
```
### viewRootImpl构造函数
```java
    public ViewRootImpl(@UiContext Context context, Display display, IWindowSession session,
            boolean useSfChoreographer) {
        mContext = context;
        mWindowSession = session;
        mDisplay = display;
        mBasePackageName = context.getBasePackageName();
//会初始化属性-mThread
        mThread = Thread.currentThread();

```
### setView
在ViewRootImpl的setView函数中，先调用了requestLayout来绘制view，然后调用了mWindowSession的addToDisplay函数和WMS

```java
requestLayout(); //绘制View

try {
    mWindowSession.addToDisplay()
}
```

mWindowSession，就是通过WMS获取到WindowSession的Binder，并且自己设置一个回调在WMS中

```java
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (mWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
                IWindowManager windowManager = getWindowManagerService();
                sWindowSession = windowManager.openSession(
                    new IWindowSessionCallback.Stub() {
                        public void onAnimatorScaleChanged(float scale) {
                            ValueAnimator.setDurationScale(scale);
                        }
                    },
                    imm.getClient, imm.getInputContext());
            }
        }
        return sWindowSession;
    }
}
```

```java
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
    int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
    Rect outOutsets, InputChannel outInputChannel) {
        return mService.addWindow(this, ...)
    }
```
### updateViewLayout
调用wm.addView之后执行了wm.updateViewLayout
调用ViewRootImpl的setLayoutParams，最终执行到了ViewRootImpl的requestLayout
```java
public void requestLayout() {
	if (!mHandlingLayoutInLayoutRequest) {
		checkThread();
		mLayoutRequested = true;
		scheduleTraversals();
	}
}
```

# 总结
1. handlerLaunchActivity->创建Activity的时候，会初始化PhoneWindow，PhoneWindow的子Viewe是DecorView。
2. onCreate->setContentView->添加view，最终调用PhoneWinow.setContej hjntView，初始化DecorView，用于管理我们的view。
3. onResume，会执行DecorView的显示逻辑，创建ViewRootImpl，将Window添加到IWindowManager服务中。
## 生命周期对线程影响
1. onCreate执行时，window已经被创建，成功绑定了WindowManagerService，此时还未创建DecorView
2. setContentView执行，decorView被创建 ，可以处理View的逻辑并且没有线程检查。
3. onResume执行时，ViewRootImpl也没有创建，也就没有线程检查。
4. onResume执行后，ViewRootImpl被创建了，所以如果在子线程中更新View就会有线程检查，必然报错。