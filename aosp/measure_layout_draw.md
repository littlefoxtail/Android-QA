任何一控件其实都是直接或间接继承自View实现的，所以说View应该具有相同的绘制流程与机制才能显示到屏幕上
每一个View的绘制过程都必须经历三个最主要的过程，也就是measure、layout和draw。

```
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

```
ViewRootImpl->performTraversals->performMeasure->performLayout->
performDraw

performMeasure->view.measure->view.onMeasure
performLayout->view.layout->view.onLayout
performDraw->view.draw->view.onDraw
```
ViewRootImpl，他是链接WindowManager和DecoreView的纽带。更广阔可以说是Window和View之间的纽带
完成View的绘制
向DecoreView分发收到的用户发起的event事件，如按键，触屏等事件

Window是一个抽象概念，每一个Window都对应一个View和一个ViewRootImple,Window又通过ViewRootImpl与View建立联系


## 开始
整个View树的绘图流程是在ViewRootImpl类的performTraversals()方法，该函数做的执行过程主要是根据之前设置的状态，判断是否重新计算视图大小
(measure)、是否重新放置视图的位置(layout)、以及是否重绘(draw)，其核心也就是判断来选择顺序执行这三个方法中的哪个。

MeasureSpec并不是指View的测量宽高，MeasureSpec作用在于：在Measure流程中，系统会将
View的LayoutParams根据父容器所施加的规则转换成对应的MeasureSpec，
然后在onMeasure方法中根据这个MeasureSpec来确定View的测量宽高。
* MeasureSpec数值（数值1080(二进制为: 1111011000)为例）

|模式名称|模式数值(高2位)|实际数值(低30位)|
|---|---|----|
|UPSPECIFIED|00|000000000000000000001111011000|
|EXACTLY|01|000000000000000000001111011000|
|AT_MOST|10|000000000000000000001111011000|
* View.MeasureSpec:

|模式|二进制数值|描述|
|---|---|----|
|UPSPECIFIED|00|默认值，父控件没有给子view任何限制，子View可以设置为任意大小|
|EXACTLY|01|表示父控件已经确切的指定了子View的大小|
|AT_MOST|10|表示子View具体大小没有尺寸限制，但是存在上限，上限一般为父View大小|

* 上面的测量模式跟`wrap_content`、`match_parent`以及写成固定的尺寸关系

|对应关系|描述|
|---|---|
|`match_parent->EXACTLY`|怎么理解呢？match_parent就是要利用父View给我们提供的所有剩余空间，而父View剩余空间是确定的，也就是这个测量模式的整数里面存放的尺寸。  |
|`wrap_content->AT_MOST`|怎么理解：就是我们想要将大小设置为包裹我们的view内容，那么尺寸大小就是父View给我们作为参考的尺寸，只要不超过这个尺寸就可以啦，具体尺寸就根据我们的需求去设定| 
|`固定尺寸（如100dp）--->EXACTLY`|用户自己指定了尺寸大小，我们就不用再去干涉了，当然是以指定的大小为主啦|

```java
int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
 // Ask host how big it wants to be
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```

## measure测量流程
### ViewGroup的测量过程
由于DecodeView继承自FrameLayout，是PhoneWindow的一个内部类，而FrameLayout没有measure方法，因此调用的是父类View
的measure方法，
```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        //首先判断当前view的layoutMode是不是特例LAYOUT_MODE_OPTICAL_BOUNDS
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
                Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }
        //根据widthMeasureSpec和heightMeasureSpec计算key值
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        //mMeasureCache是LongSparseLongArray类型的成员变量
        //其缓存着view在不同widthMeasureSpec、heightMeasureSpec下量算的结果
        
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
        //mOldWidthMeasureSpec和mOldHeightMeasureSpec分别表示上次对view测量的值
        //mPrivateFlags是一个Int类型的值，其记录了View的各种状态位
        //forceLayout需要强制执行layout，所以这种情况下要尝试进行量算
        //如果新传入的widthMeasureSpec/heightMeasureSpec与上次量算时的mOldWidthMeasureSpec/mOldHeightMeasureSpec不等，
        //那么也就是说该View的父ViewGroup对该View的尺寸的限制情况有变化，这种情况下要尝试进行量算


        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec || heightMeasureSpec != mOldHeightMeasureSpec;
        ...
        if ((forceLayout || neddsLayout) {
                // 通过按位操作，重置View的状态mPrivateFlags，将其标记为未量算状态
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
            //在真正进行量算之前，View还想进一步确认能不能从已有缓存mMeasureCache中读取缓存过的量算结果
            //如果是强制layout导致的量算，那么将cacheIndex设置为-1，即不从缓存中读取量算结果
            //如果不是强制layout导致的量算，那么就用key作为缓存索引cacheIndex
            int cahceIndex = fourceLayout ? -1 : mMeasureCache.indexOfKey(key);
        //sIgnoreMeasureCache是一个boolean类型的成员变量，其值是在View的构造函数中计算的，而且只计算一次
        //一些老的App希望在一次layou过程中，onMeasure方法总是被调用，
        //具体来说其值是通过如下计算的: sIgnoreMeasureCache = targetSdkVersion < KITKAT;
        //也就是说如果targetSdkVersion的API版本低于KITKAT，即API level小于19，那么sIgnoreMeasureCache为true
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                // 如果运行到此处，表示没有从缓存中取到
                // onMeasure方法将会进行实际的量算工作，并把量算的结果保存到成员变量
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                //如果运行到此，表示当前的条件允许View从缓存成员变量mMeasureCache中读取量算过的结果
                long value = mMeasureCache.valueAt(cacheIndex);

            }
        ...
}
```

最后会调用super.onMeasure， onMeasure不同的ViewGroup有着不同的实现，但大体是对每个子View进行遍历，根据ViewGroup的
MeasureSpec及子View的layoutParams来确定自身的测量宽高，然后根据所有子View的测量宽高信息再确定父容器的测量宽高。

### View的测量过程
ViewGroup提到measureChildWithMargin方法，它接收的主要参数是子View以及父容器的MeasureSpec，所以它的作用就是对子View进行测量，
ViewGroup#measureChildWithMargins:

|子View的LayoutParams\父容器SpecMode|EXACTLY|AT_MOST|UNSPECIFIED|
| ---- | ---- | ---- | ---- |
|精确值(dp)|EXACTLY childSize|EXACTLY childSize|EXACTLY childSize|
|match_parent|EXACTLY parentSize|AT\_MOST parentSize|UNSPECIFIED 0|
|wrap_content|AT_MOST parentSize|AT\_MOST parentSize|UNSPECIFIED 0|

通过获取子View的MeasureSpec获得后，执行child.measure，绘制流程已经从ViewGroup转移到子View中了，
在measure方法中会调用onMeasure方法，
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
        getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

### 总结
测量始于DecorView，通过不断的遍历子View的measure方法，根据ViewGroup的MeasureSpec及子View的LayoutParams来决定
子View的MeasureSpec，进一步获取子View的测量宽高，然后逐层返回，不断保存ViewGroup的测量高度。

## layout流程
### ViewGroup的布局流程
performLayout方法进行layout流程
通过对mLeft、mTop、mRight、mBottom这四个值进行了初始化，对于每一个View，包括ViewGroup来说，以上四个值保存了
View的位置信息，所以这四个值是最终宽高。也即是说，如果要得到View的位置信息，那么就应该在layout方法完成后调用
getLeft(),getTop()等方法来取得最终宽高，如果是在此之前调用相应的方法，只能得到0的结果，所以一般我们是在onLayout
方法中获取View的宽高信息。

首先先获取父容器的padding值，然后遍历其每一个子View，  
根据子View的layout_gravity属性、子View的测量宽高、父容器的padding值、来确定子View的布局参数，  
然后调用child.layout方法，把布局流程从父容器传递到子元素

View#layout()
```java
public void layout(int l, int t, int r, int b) {
        //成员变量mPrivateFlags3中的一些比特位存储和laout相关的信息
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
                //如果在mPrivateFlag3的低位字节的第4位（从最右4向左第4位）的值为1，
                //那么久表示在layout布局的需要对View进行测量

                onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
                //量完了需要移除标签
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
                }

                int oldL = mLeft;
                int oldT = mTop;
                int oldB = mBottom;
                int oldR = mRight;
                //这里setOpticalFrame还是会调用setFrame
                //setFrame方法会将新的left,top,right,bottom存储到view的成员变量
                //返回true代表发生了view的位置和尺寸变化
                boolean changed = isLayoutModeOptical(mParent) ?
                        setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

                if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
                //如果view的布局发生了变化，或者mPrivateFlags有需要LAYOUT的标签PFLAG_LAYOUT_REQUIRED,那么就会执行
                //首先触发onLayout方法执行，View中默认的onLayout是个空方法
                //不过继承自ViewGroup的类都要实现onLayout，从而在onLayout中依次循环子View
                //并调用view#layout
                onLayout(changed, l, t, r, b);

                if (shouldDrawRoundScrollbar()) {
                        if(mRoundScrollbarRenderer == null) {
                        mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
                        }
                } else {
                        mRoundScrollbarRenderer = null;
                }
                //移除flag
                mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
                //可以通过View的addOnLayoutChangeListener(View.onLayoutChangeListener)方法
                //这些事件都存储在ListenerInfo.mOnLayoutChangeListeners
                ListenerInfo li = mListenerInfo;
                if (li != null && li.mOnLayoutChangeListeners != null) {
                        // 对mOnlayoutChangeListeners中的事件监听器进行拷贝
                        ArrayList<OnLayoutChangeListener> listenersCopy =
                                (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                        int numListeners = listenersCopy.size();
                        for (int i = 0; i < numListeners; ++i) {
                                // 遍历注册器，依次调用onLayoutChange方法，这样Layout事件监听器就得到了响应
                        listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                        }
                }
                }
                // 移除强制layout标签
                mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
                // 加入layout完成标签
                mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

                if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
                mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
                notifyEnterOrExitForAutoFillIfNeeded(true);
                }
}

```

```java
protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;

        if (DBG) {
            Log.d("View", this + " View.setFrame(" + left + "," + top + ","
                    + right + "," + bottom + ")");
        }

        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            //将新旧left、right、top、bottom进行对比，只要不完全相对就说明View的局部发生了变化，则将changed变量设置为true 
            changed = true;

            // Remember our drawn bit
            // 先保存一下mPrivateFlags中的PFLAG_DRAW标签信息
            int drawn = mPrivateFlags & PFLAG_DRAWN;
            // 计算View的新旧尺寸
            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            // 比较View的新旧尺寸是否相同，如果尺寸发生了变化，那么sizeChanged的值为true
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // Invalidate our old position
            invalidate(sizeChanged);
            //将新的left、top、right、bottom存储到View的成员变量中 
            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
            //mRenderNode.setLeftTopRightBottom()方法会调用RenderNode中原生方法的nSetLeftTopRightBottom()方法，
            //该方法会根据left、top、right、bottom更新用于渲染的显示列表
            mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
            //向mPrivateFlags中增加标签PFLAG_HAS_BOUNDS,表示当前View具有了明确的边界范围
            mPrivateFlags |= PFLAG_HAS_BOUNDS;


            if (sizeChanged) {
                //如果View的尺寸和之前发生了变化，那么就执行sizeChange()方法
                //该方法中又会调用onSizeChanged方法，并将View的新旧尺寸传递进去
                sizeChange(newWidth, newHeight, oldWidth, oldHeight);
            }

            if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {
                // If we are visible, force the DRAWN bit to on so that
                // this invalidate will go through (at least to our parent).
                // This is because someone may have invalidated this view
                // before this call to setFrame came in, thereby clearing
                // the DRAWN bit.
                // 有可能在调用setFrame方法之前，invalidate方法就被调用了
                // 这会导致mPrivateFlag移除了PFLAG_DRAWN标签
                // 如果当前View处于可见状态就将mPrivateFlags强制添加PFLAG_DRAW状态位，
                // 这样会确保下面的invalidate()方法会执行到其父控件级别
                mPrivateFlags |= PFLAG_DRAWN;
                invalidate(sizeChanged);
                // parent display list may need to be recreated based on a change in the bounds
                // of any child
                // 父控件会重建用渲染的显示列表
                invalidateParentCaches();
            }

            // Reset drawn bit to original value (invalidate turns it off)
            // 重新恢复mPrivateFlags中原有的PFLAG_DRAWN标签信息
            mPrivateFlags |= drawn;

            mBackgroundSizeChanged = true;
            mDefaultFocusHighlightSizeChanged = true;
            if (mForegroundInfo != null) {
                mForegroundInfo.mBoundsChanged = true;
            }

            notifySubtreeAccessibilityStateChangedIfNeeded();
        }
        return changed;
    }
```
```java
private void sizeChange(int newWidth, int newHeight, int oldWidth, int oldHeight) {
        //将View的新旧尺寸传递给onSizeChange()方法
        if (mOverlay != null) {
                mOverlay.getOverlayView().setRight(newWidth);
                mOverlay.getOverlayView().setBottom(newHeight);
        }
        rebuidOutline();

}
```

### 子View的布局流程
子View的布局流程也很简单。如果子View是一个ViewGroup,那么会重复以上步骤，如果是一个View，那么会直接调用  
View#layout方法，根据以上分析，在该方法内部会设置view的四个布局参数，接着调用onLayout方法，
View#onLayout方法是一个空实现，主要作用是在我们自定义View中重写该方法，实现自定义的布局逻辑。

### 总结
View的布局流程就已经全部分析完了。可以看出，布局流程的逻辑相比测量流程来说，简单许多，  
获取一个View的测量宽高是比较复杂的，而布局流程则是根据已经获得的测量宽高进而确定一个View的四个位置参数

## draw流程
![Draw](../img/draw.png)
绘制流程将决定View的样子，一个View该显示什么由绘制流程完成。
### ViewRootImpl#performDraw

里面又调用了ViewRootImpl#draw方法，并传递了fullRedrawNeeded参数。
该参数由mFullRedrawNeeded成员变量获取，它的作用是判断是否重新绘制全部视图，如果是第一次绘制视图，
那么显示应该绘绘制所有的视图，如果由于某些原因，导致了视图重绘，那么就没有必要绘制所有视图。

### ViewRootImpl#draw
首先获取了mDirty值，该值保存了需要重绘区域的信息，关于视图绘制。根据fullRedrawNeeded来判断是否需要
重置dirty区域，最后调用了ViewRootImpl#drawSoftware方法，并把相关参数传递进去，包括dirty区域

首先实例化了Canvas对象，然后锁定该canvas的区域，由dirty区域决定，接着对canvas进行一系列的属性赋值，
最后调用了mView.draw方法，前面分析过mView就是DecorView，也就是说从DecorView开始绘制。

### View的绘制流程
由于ViewGroup没有重写draw方法，因此所有的View都是调用View#draw
绘制流程的六个步骤：
1. 对View的背景进行绘制
2. 保存当前的图层信息
3. 绘制View的内容
4. 对View的子View进行绘制
5. 绘制View的褪色的边缘，类似于阴影效果。
6. 绘制View的装饰


### 补充章节
#### 确定View大小（onSizeChanged）
**Q: 在测量完View并使用setMeasuredDimension函数之后View的大小基本上已经确定了，那么为什么还要再次确定View的大小呢？**

**A: 这是因为View的大小不仅由View本身控制，而且受父控件的影响，所以我们在确定View大小的时候最好使用系统提供的onSizeChanged回调函数。**


### 重点知识梳理


### 自定义View分类

> PS ：实际上ViewGroup是View的一个子类。

| 类别        | 继承自                 | 特点      |
| --------- | ------------------- | ------- |
| View      | View SurfaceView 等  | 不含子View |
| ViewGroup | ViewGroup xxLayout等 | 包含子View |

### 自定义View流程：

| 步骤   | 关键字           | 作用                           |
| ---- | ------------- | ---------------------------- |
| 1    | 构造函数          | View初始化                      |
| 2    | onMeasure     | 测量View大小                     |
| 3    | onSizeChanged | 确定View大小                     |
| 4    | onLayout      | 确定子View布局(自定义View包含子View时有用) |
| 5    | onDraw        | 实际绘制内容                       |
| 6    | 提供接口          | 控制View或监听View某些状态。           |

### rquestLayout
当我们动态移动一个View的位置或者View的大小、形状发生了变换的时候，我们可以在View中调用这个方法。

子View调用requestLayout方法，会标记当前View及父容器，同时逐层向上提交，直到ViewRootImpl处理该事件，ViewRootImpl会调用三大流程，从measure开始，对于每一个含有标记位的view及其子View都会进行测量、布局、绘制。


### invalidate
该方法调用会引起View树的重绘，常用于内部调用或者需要刷新界面的时候，需要在主线程中调用该方法。当子View调用了invalidate方法后，会为该View添加一个标记位，同时不断向父容器请求刷新，父容器通过计算得出自身需要重绘的区域，知道传递到ViewRootImpl中，最终触发performTraversals方法，进行开始View树重绘流程。

### postInvalidate
postInvalidate是在非UI线程中调用，invalidate是在UI线程中调用

一般来说，如果View确定自身不再适合当前区域，比如说它的LayoutParams发生了改变，需要父布局对其进行重新测量、布局、绘制这三个流程，往往使用requestLayout。而invalidate则是刷新当前View，使当前View进行重绘，不会进行测量、布局流程，因此如果View只需要重绘而不需要测量，布局的时候，使用invalidate方法往往比requestLayout方法更高效


# 下面是真正的流程图

![img](../img/view_measure_layout_draw.jpg)



