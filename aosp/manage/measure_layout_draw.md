
# 测量、定位和绘制

任何一控件其实都是直接或间接继承自View实现的，所以说View应该具有相同的绘制流程与机制才能显示到屏幕上
每一个View的绘制过程都必须经历三个最主要的过程，也就是measure、layout和draw。

```text
├── View
│   ├── measure(int, int) 测量view多大，父提供参数，该方法会调用onMeasure(int,int)
|   ├── onMeasure(int, int) 此方法可以被覆盖，子类有责任测量宽高
|   ├── layout(int, int, int, int) 布局定位，除了ViewGroup实现了,其他并不需要
|   ├── onLayout(boolean, int, int, int, int) 包含子的派生类需要实现，并且每个子要调用layout方法
|   ├── onDraw(Canvas) 绘制需要重写的方法
|   ├── draw(Canvas) 绘制流程的主方法，一般不重写
|   ├── draw(Canvas, ViewGroup, long) 该方法是ViewGroup.drawChild调用的，用于每个子类进行绘制自身
|   ├── dispatchDraw(Canvas) 绘制流程用于绘制子views，会被派生类重载，在children绘制之前。

├── ViewGroup
│   ├── layout(int, int, int, int) //ViewGroup实现了
│   ├── onLayout(boolean, int, int, int, int) //viewGroup是一个抽象类 需要实现
```

## 初始化工作

Activity方法onCreate里执行了setContentView只有View如何显示到屏幕上的。

1. Activity.setContentView->PhoneWindow.setContentView最终会生成一个DecorView对象
2. DecoreView是PhoneWindow类的内部类，继承自FrameLayout，所以调用Activity方法

```text
ViewRootImpl->performTraversals->performMeasure->performLayout->performDraw
performMeasure->view.measure->view.onMeasure
performLayout->view.layout->view.onLayout
performDraw->view.draw->view.onDraw
```

ViewRootImpl，他是链接WindowManager和DecoreView的纽带。更广阔可以说是Window和View之间的纽带
完成View的绘制，向DecoreView分发收到的用户发起的event事件，如按键，触屏等事件

Window是一个抽象概念，每一个Window都对应一个View和一个ViewRootImple,Window又通过ViewRootImpl与View建立联系
![image](/img/hierarchy.png)

## 开始

整个View树的绘图流程是在ViewRootImpl类的performTraversals()方法，该函数做的执行过程主要是根据之前设置的状态，判断是否重新计算视图大小(measure)、是否重新放置视图的位置(layout)、以及是否重绘(draw)，其核心也就是判断来选择顺序执行这三个方法中的哪个。

1. Android自上而下对所有View进行量算，这样Android就知道了每个View想要的大小，即宽高信息
2. 在完成了所有的View的量算工作后，Android会自上而下对所有View进行布局，Android就知道了每个View在其父控件中的位置，即View到其父控件四边
3. 在完成了对所有View的布局工作后，Android会自上而下对所有View进行绘图，这样Android就将所有的View渲染到屏幕上。

### performTraversals实现

#### 计算窗口期望尺寸

##### 开始的条件

```java
final View host = mView;
// mAdded指DecorView是否被成功加入到window中，在setView中被赋值为true
if (host == null || !mAdded)
    return;
```

##### 初始化窗口的期望宽高，并初始化几个标记

- 如果是第一次traversals
  - 设置需要重新draw和layout
  - 根据layoutParams的type去判断是否需要使用屏幕完全尺寸
    - Y：设置窗口尺寸为屏幕真实尺寸，不剪去任何装饰的尺寸
    - N：设置窗口尺寸为可用的最大尺寸

```java
// mFirst在构造器中被赋值true，表示第一次traversals
// 在后面代码中被赋值false
if (mFrist) {
    //设置需要重新draw并且重新layout
    mFullRedrawNeeded = true;
    mLayoutRequested = true;

    final Configuration config = mContext.getResources().getConfiguration();
    if (shouldUseDisplaySize(lp)) {
        Point size = new Point();
        mDisplay.getRealSize(size);
        desiredWindowWidth = size.x;
            desiredWindowHeight = size.y;
        } else {
            //设置窗口尺寸为可用的最大尺寸
            desiredWindowWidth = mWinFrame.width();
            desiredWindowHeight = mWinFrame.height();
        }
}
```

- 如果不是第一次traversals
  - 设置窗口宽高为全局变量存储的宽高
    - 如果这个值和上次traversals时的值不同了
      - Y：设置需要重新layout和draw
      - N：默认不需要重新进行

```java
if () {
    //...
} else {
    //如果不是第一次traversals，就直接使用之前存储的mWinFrame的宽高
    desiredWindowWidth = frame.width();
    desiredWindowHeight = frame.height();
    // mWidth和mHeight是上一次traversals时赋frame的值的
    // 如果现在的值不一样了，那么就需要重新draw和layout
    if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
        mFullRedrawNeeded = true;
        mLayoutRequested = true;
        windowSizeMayChange = true;
    }
}
```

> mWidth和mHeight也是用来描述Activity窗口当前宽度和高度的，他们的值由应用程序进程上一次主动WindowManagerService计算得到的，并且会一直保持不变到应用程序进程下一次再请求WindowManagerService重新计算为止

##### 赋值窗口期望大小并开始测量流程

要求重新测量时

- 如果是第一次traversals
  - 确保窗口的触摸模式已经打开

```java
boolean insetsChanged = false;
boolean layoutRequested = mLayoutRequested & (!mStopped || mReportNextDraw);
if (layoutRequested) {
    final Resources res = mView.getContext().getResources();
    //如果是第一次
    if (mFirst) {
        mAttachInfo.mInTouchMode = !mAddedTouchMode;
        //确保window的触摸模式已经打开
        // 内部mAttachInfo.mInTouchMode = inTouchMode
        ensureTouchModeLocally(mAddedTouchMode);
    } else {

    }
}

```

- 如果不是第一次traversals
  - 比较mPending..Insets和上次存在mAttachInfo中的是否改变
    - 如果改变：insetsChanged置为true。
    - 未改变：insetsChanged默认false。
  - 如果窗口宽高有设置为wrap_content
    - 设置windowSizeMayChange为true
    - 重新计算窗口期望宽高
  - 调用measureHierarchy()去测量窗口宽高，赋值窗口尺寸是否改变

##### 判断是否重置了窗口尺寸，是否需要重新计算insets

- 清空mLayoutRequested = false;
- 设置windowShouldResize
- 设置computesInternalInsets

```java
if (layoutRequested) {
    //暂时清空这个标记，在下面再次需要的时候再重新赋值
    mLayoutRequested = false;
}
//同时满足三个条件
// 1. layoutRequested为true，已经发起了一次新的layout
// 2. 上面measureHierarchy()中测量的值和上一次保存的值不同
// 或
// 3. 宽或高设置为wrap_content并且这次请求WMS的值和期望值、上次的值都不同
boolean windowShouldResize = layoutRequested && windowSizeMayChange
    && ((mWidth != host.getMeasureWidth() || mHeight != host.getMeasuredHeight())
        || (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT &&
            frame.width() < desiredWindowWidth && frame.width() != mWidth)
        || (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT &&
            frame.height() < desiredWindowHeight && frame.height() != mHeight));

windowShouldResize |= mDragResizing && mResizeMode == RESIZE_MODE_FREEFORM;

//如果activity重新启动
windowShouldResize |= mActivityRelaunched;

// 设置是否需要计算insets，设置了监听或存在需要设置的空insets
final boolean computesInternalInsets =
        mAttachInfo.mTreeObserver.hasComputeInternalInsetsListeners()
        || mAttachInfo.mHasNonEmptyGivenInternalInsets;
```

#### 确定窗口尺寸，调用WMS计算并保存

```java
// 第一次traversals 或
// 窗口尺寸有变化或insets有变化 或
// 
if (mFirst || windowShouldResize || insetsChanged ||
        viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
    //清空这个标记位
    mForceNextWindowRelalyout = false;
}
```

##### 进入if代码块

进入条件（或）

- 第一次traversals
- 窗口尺寸有变化
- insets有变化
- 窗口visibility有变化
- 窗口属性有变化
- 设置了强迫下一次必须重新layout

1. 设置insetsPending

    ```java
    if (isViewVisible) {
        //如果insets发生改变 并且是第一次traversals或窗口从不可见变为可见
        // 就设置insetsPending为true
        insetsPending = computesInternalInsets && (mFirst || viewVisibilityChanged);
    }
    ```

2. 调用WMS重新计算并保存

    ```java
    //调用relayoutWindow重新计算窗口尺寸以及insets大小
    //会使用IPC去请求 WMS
    relayoutResult = relayoutWindow(..);

    //frame指向的是mWinFrame，此时已经是上面重新请求WMS计算后的值了
    //将值保存在mAtachInfo
    mAttachInfo.mWindowLeft = frame.left;
    mAttachInfo.mWidnowTop = frame.top;

    if (mWidth != frame.width() || mHeight != frame.height()) {
        mWidth = frame.width();
        mHeight = frame.height();
    }
    ```

3. 是否需要重新测量

- 如果窗口不处于停滞状态或提交了下一次的绘制
  - 如果需要重新测量窗口尺寸
    - 执行测量操作
    - 如果有额外分配空间
      - 计算宽高上的额外空间加到上次计算的宽高尚
      - 重新测量
    - layoutRequested赋值true


    ```java
    //如果窗口不处于停止状态或者提交了下一次的绘制
    if (!mStopped || mReportNextDraw) {
        //判断是否需要重新测量窗口尺寸
        //窗口触摸模式发生改变，焦点发生改变
        //或 测量宽高与WMS计算宽高不相等
        //或 insets改变了
        //或 配置发生了改变，mPendingMergedConfigutation有变化
        if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidht()
                || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                updatedConfiguration) {
            //重新计算decorView的MeasureSpec
            int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
            int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

            //执行测量操作
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

            int width = host.getMeausredWidht();
            int height = host.getMeasuredHeight();
            boolean measureAgain = false;
            //判断是否需要重新测量
            //如果需要在水平方向上分配额外的像素
            if (lp.horizontalWeight > 0.0f) {
                //测量宽度加上额外的宽度
                width += (int) ((mWidth - width) * lp.horizontalWeight);
                //重新计算MeasureSpec
                childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width, MeasureSpec.EXAXTLY);
                //设置需要重新测量
                measureAgain = true;
            }

            if (lp.verticalWeight > 0.0f) {
                height += (int)((mHeight - height) * lp.verticalWeight);
                childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height, MeasureSpec.EXAXTLY);
                //设置需要重新测量
                measureAgain = true;
            }
            //如果需要重新测量了，就再调用一次performMeasure
            if (measureAgain) {
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
            layoutRequested = true;
        }

    }
    ```

##### 进入else代码块

- 不是第一次traversals
- 各种参数都没有改变

```java
//检查窗口发生移动的情况
maybeHandleWindowMove(frame);
```

- 判断窗口是否发生移动
  - 判断是否有动画
    - 执行动画
  - 改变mAttachInfo中的参数

```java
private void maybeHandleWindowMove(Rect frame) {
    //frame和窗口做了同样的移动，这样就会导致很多参数没有改变，但是我们还是需要做一些工作
    // 比较检查窗口是否发生移动
    final boolean windowMoved = mAttachInfo.mWindowLeft != frame.left || mAttachInfo.mWindowTop != frame.top
    if (windowMoved) {
        if (mTranslator != null) {
            //如果窗口发生了移动，并且存在动画，就执行动画
            mTranslator.translateRectInScreenToAppWinFrame(frame);
        }
        //改变位置
        mAttachInfo.mWindowLeft = frame.left;
        mAttachInfo.mWindowRight = frame.top;
    }
}
```

#### 调用layout和draw

##### layout

- 如果需要layout，并且窗口不是停止状态或提交了下一次draw
  - 调用performLayout()开始layout流程
  - 处理透明区域

```java
final boolean didiLayout = layoutRequested && (!mStopped || mReportNextDraw);
boolean triggerGlobalLayoutListener = didLayout || mAttachInfo.mRecomputeGlobalAttributes;
if (didLayout) {
    //开始layout流程
    performLayout(lp, mWidth, mHeight);
    //处理透明区域
    if (host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS != 0) {

    }

}
```

##### 处理insets

```java
if (computesInternalInsets) {
    final ViewTreeObserver.InternalInsetsInfo insets = mAttachInfo.mGivenInternalInsets;
    //重置insets树，清空状态
    insets.reset();
    //遍历计算
    mAttachInfo.mTreeObserver.dispatchOnComputeInternalInsets(insets);
    mAttachInfo.mHasNonEmptyGivenInternalInsets = !insets.isEmpty();


    try {
        //远程调用WMS去设置insets
        mWindowSession.setInsets(..);
    } catch(RemoteException e) {

    }
}
```

##### 收尾工作

```java
//设置不是第一次，之后再次调用traversals
mFrist = false;

```

##### draw

在第一次traversals时并不会调用performDraw()，因为创建了新的平台。scheduleTraversals()中会post一个runnable，会再次调用performTraversals()，等到那次调用时才会去执行draw流程。

```java
boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
//没有取消draw也没有创建新的平面，第一次traversals时newSurface为true
if (!cancelDraw && !newSurface) {
    //如果还存在等待执行的动画，就遍历执行它们
    if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
        for (int i = 0; i < mPendingTransitions.size(); ++i) {
            mPendingTransitions.get(i).startChangingAnimations();
        }
        mPendingTransitions.clear();
    }
    //开始draw流程
    performDraw();
} else {
    if (isViewVisible) {
        scheduleTraversals();
    } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
        //执行等待执行的动画
        for (int i = 0; i < mPendingTransitions.size(); ++i) {
            mPendingTransitions.get(i).endChangingAnimations();
        }
        mPendingTransitions.clear();
    }
    mIsInTraversal = false;
}

```

## 总结

### 详细流程图

![详细流程图](/img/view_measure_layout_draw.png)

### 总结信息

1. 渲染Android应用视图的渲染流程，测量流程用来确定视图的大小、布局流程用来确定视图的位置、绘图流程最终将视图绘制在应用窗口上
2. Android应用程序窗口UI首先是使用Skia图形库API来绘制在一块画布上，实际地是绘制在这块画布里面的一个图形缓冲区中，这个图形缓冲区最终会被交给SurfaceFlinger服 务，而SurfaceFlinger服务再使用OpenGL图形库API来将这个图形缓冲区渲染到硬件帧缓冲区中。

### 补充章节

#### 确定View大小（onSizeChanged

**Q: 在测量完View并使用setMeasuredDimension函数之后View的大小基本上已经确定了，那么为什么还要再次确定View的大小呢？**
**A: 这是因为View的大小不仅由View本身控制，而且受父控件的影响，所以我们在确定View大小的时候最好使用系统提供的onSizeChanged回调函数。**

### 重点知识梳理

### 自定义View分类

> PS ：实际上ViewGroup是View的一个子类。

| 类别        | 继承自                 | 特点      |
| --------- | ------------------- | ------- |
| View      | View SurfaceView 等  | 不含子View |
| ViewGroup | ViewGroup xxLayout等 | 包含子View |

### 自定义View流程

| 步骤   | 关键字           | 作用                           |
| ---- | ------------- | ---------------------------- |
| 1    | 构造函数          | View初始化                      |
| 2    | onMeasure     | 测量View大小                     |
| 3    | onSizeChanged | 确定View大小                     |
| 4    | onLayout      | 确定子View布局(自定义View包含子View时有用) |
| 5    | onDraw        | 实际绘制内容                       |
| 6    | 提供接口          | 控制View或监听View某些状态。           |
