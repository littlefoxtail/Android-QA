# WindowManagerService

WindowManagerService，是一个窗口管理系统服务，主要功能如下：

* 窗口管理，绘制
* 转场动画--Activity切换动画
* Z-ordered的维护，Activity窗口显示前后顺序
* 输入法管理
* Token管理
* 系统消息收集线程
* 系统消息分发线程

WindowManagerService是继ActivityManagerService与PackageManagerService之后又一个复杂却十分重要的系统服务

先了解下(Window)是什么？

Android系统中的窗口是屏幕上的一块用于绘制各种UI元素并可以响应用户输入的一快矩形区域。从原理上来讲，窗口的概念是独自占有一个
Surface实例的显示区域。例如Dialog、Activity的界面、壁纸、状态栏以及Toast等都是窗口。

![window_manager_service_class](/img/WindowManagerService_class.png)

主要角色：

* WindowManager：应用于窗口管理服务WindowManagerService交互的接口
* WindowManagerService：窗口管理服务，继承于IWindowManager.Stub交互接口，是Binder的服务端，该服务运行在一个单独的进程中，因此WindowManager与WindowManagerService的交互也是一个IPC的过程
* SurfaceFlinger：SurfaceFlinger服务运行在Android系统的System进程中，它负责管理Android系统的帧缓冲区(Frame Buffer)，Android设备的显示屏被抽象为一个帧缓冲区，而Android系统中的SurfaceFlinger服务就是向这个帧缓冲区写入内容来绘制应用程序的用户界面
* Surface：每一个显示界面的窗口都是一个Surface
* PhoneWindowManager：实现了窗口的各种策略
* Choreographer：用户控制窗口动画、屏幕旋转等操作
* DisplayContent：用于描述多屏输出相关信息
* WindowState：描述窗口的状态信息以及和WindowManagerService进行通信，一般一个窗口对应一个
* WindowToken：窗口Token，用来做Binder通信
* Session：通讯对象，App进程通过建立Session代理对象和Session对象通信，进而和WindowManagerService建立连接

![window_mansger_service_structure](/img/window_mansger_service_structure.png)

Activity持有一个Window对象，负责UI的展示与交互

* mWindow：PhoneWindow对象，继承与Window，是窗口对象
* mWindowManager：WindowManagerImpl对象，实现WindowManager
* mMainThread：ActivityThread对象，并非真正的线程，是运行在主线程里的对象
* mUIThread：Thread对象，主线程
* mHandler：Handler对象，主线程Handler
* mDecor：View对象，用来展示Activity里的视图

ViewRootImpl负责管理DecorView与WindowManagerService的交互，每次调用WindowManager.addView()添加窗口时，都会创建一个ViewRootImpl对象，它内部也保存了一些重要信息：

* mWindowSession：IWindowSession对象，Session的对象代理，用来和Session进行通信，同一进程里的所有ViewRootImpl对象只对应同一个Session代理对象
* mWindow：IWindow.Stub对象，每个窗口对应一个对象

## 一 Window的添加流程

```java
public final class WindowManagerGlobal {
    public void addView(View view, ViewGroup.LayoutParams params,
                Display display, Window parentWindow) {
           //校验参数的合法性

           //ViewRootImpl封装了View与WindowManager的交互
           ViewRootImpl root;
           View panelParentView = null;
           synchronized (mLock) {
                root = new ViewRootImpl(view.getContext(), display);
                view.setLayoutParams(wparams);
                // mViews存储着所有Window对应的View对象
                mViews.add(view);
            //    mRoots存储着所有Window对应的ViewRootImpl对象
                mRoots.add(root)
            //    mParams存储着所有Window对应的WindowManager.LayoutParams对象

                try {
                    root.setView(view. wparams, panelParentView);
                } catch (RuntimeException e) {
                    if (index >= 0) {
                        removeViewLocked(index, true)
                    }
                    throw e;
                }
           }
    }
}
```

ViewRootImpl就是一个封装类，封装了View与WindowManager的交互方式，它是View与WindowManagerService通信的桥梁
最后也是调用ViewRootImpl.setView()方法来完成Window的添加并完成Window的添加并完成更新界面

```java
public final class ViewRootImpl implements ViewParent {
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                // 参数校验与预处理

                // 1 requestLayout()完成界面异步绘制的请求
                requestLayout();
                // 2 创建WindowSession并通过WindowSession请求
                // WindowManagerService来完成Window添加的过程，这是一个IPC过程
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
            }
        }
    }
}
```

1. requestLayout()完成界面异步绘制的请求
2. 创建WindowSession并通过WindowSession请求WindowManagerService来完成Window添加的过程这个是一个IPC过程，WindowManagerService作为实际的窗口管理者，窗口的创建、删除和更新都是由它来完成的，它同时还负责了窗口的层叠排序和大小计算

WindowManager与WindowManagerService的跨进程通信。Android的各种服务都是基于C/S结构来设计的，系统层提供服务，应用层使用服务。
WindowManager也是一样，它与WindowManagerService的通信都是通过WindowSession来完成的。


## 二 Window的删除流程

```java
public final class WindowManagerGlobal {
    public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }
        }
    }

    private void removeVieLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoot.get(index);
        View view = root.getView();

        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();

            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }
}
```

ViewRootImpl.die()

```java
boolean die(boolean immediate) {
    //根据immediate参数来判断值执行异步删除还是同步删除
    if (immediate && !mIsInTraversal) {
        doDie();
        return false;
    }

    if (!mIsDrawing) {
        destoryHardwareRenderer();
    }

    //如果异步删除，则发送一个删除View的消息MSG_DIE就会直接返回
    mHandler.sendEmptyMessage(MSG_DIE);
    return true;
}

void doDie() {
    checkThread();

    synchronized(this) {
        if (mRemoved) {
            return
        }
        mRemoved = true;
        if (mAdded) {
            dispatchDetachedFromWindow();
        }
        if (mAdded && !mFirst) {
            destoryHardwareRenderer();

            if (mView != null) {
                int viwVisibility = mView.getVisibility();
                boolean viewVisibilityChanged = mViewVisibility != viewVisibility;
                if (mWindowAttributesChanged || viewVisibilityChanged) {
                    try {
                        if (relayoutWindow(mWindowAttributes, viewVisibility, false)
                                & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0 {
                                mWindowSession.finishDrawing(mWindow);
                        }
                    } catch (RmoteException e) {

                    }
                }

                mSurface.release();

            }
        }
        mAdded = false;
    }
    //刷新数据，将当前移除View的相关信息
    WindowManagerGlobal.getInstance().doRemoteView(this);
}

public void dispatchDetachedFromWindow() {
    // 回调View的dispatchDetachedFromWindow方法，通知该View已从Window中移除

    if (mView != null && mView.mAttachInfo != null) {
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
        mView.dispatchDetachedFromWindow();
    }

    //调用WindowSession.remove()方法，这同样是一个IPC过程，最终调用的是
    // WindowManagerService.removeWindow()方法来移除Window
    try {
        mWindowSession.remove(mWindow);
    } catch (RemoteException e) {

    }

    if (mInputChannel != null) {
        mInputChannel.dispose();
        mInputChaneel = null;
    }

    mDispalyManager.unregisterDisplayListener(mDisplayListener);

    unscheduleTraversals();
}
```

## 三 Window的更新数据

```java
public final class WindowManagerGloba {
    public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        //更新View的LayoutParams参数
        view.setLayoutParams(wparams);

        synchronized(mLock) {
            //查找View的索引，更新mParams里的参数
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoot.get(index);
            mParams.remove(index);
            mParams.add(index, wparams);
            root.setLayoutParams(wparams, false);
        }
    }
}
```

```java
public final class ViewRootImpl implements ViewParents {
    void setLayoutParams(WindowManager.LayoutParams attrs, boolean newView) {
        //参数预处理

        //如果是新View，调用requestLayout()进行重新绘制
        if (newView) {
            mSoftInputMode = attrs.softInputMode;
            requestLayout();
        }

        mWindowAttributesChanged = true;
        scheduleTraversals();
    }
}
```

> 事实上Window是一个抽象的概念，也就是说它并不是实际存在的，它以View的形式存在，每个Window都对应着一个View和一个ViewRootImpl，Window与View通过ViewRootImpl来建立联系。
WindowManagerService实际管理的也不是Window，而是View，管理当前状态下那个View应该在最上层显示，SurfaceFlinger绘制也同样是View