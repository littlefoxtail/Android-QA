# RecyclerView的设计结构

- RecyclerView创建流程
- RecyclerView布局策略管理器LayoutManager
- RecyclerView视图描述者ViewHolder
- RecyclerView视图复用器RecyclerView
- RecyclerView视图适配器Adapter
- RecyclerView视图动画ItemAdapter
- RecyclerView视图分隔条ItemDecoration

RecyclerView继承于ViewGroup，实现ScrollingView与NestedScrollingChild接口

对比ListView：
||Item复用方面|样式丰富方面|效果增强|代码内聚|
|:---:|:---:|:---:|:---:|:--:|
|RecyclerView|内置了RecyclerViewPool、多级缓存、ViewHolder|通过支持水平、垂直和表格列表及其他更复杂形势|内置ItemDecoration和ItemAnimator，可以自定义绘制ItemView之间的一些特殊UI或Item项数据变化时的动画效果|将功能密切相关的类写成内部类|
|ListView|需要手动添加ViewHolder|只支持具体某一种|将这些特殊UI作为ItemView的一部分|无|

![recyclerview_structure](../../img/recyclerview_structure.png)

# 创建流程

## 构造

```java
public RecyclerView(Contex context, AttributeSet attrs, int defStyle) {
    super(context, attrs, defStyle);
    //设置当前View可以滑动
    setScrollContainer(true);
    //设置当前View可以获取焦点
    setFocusableInTouchMode(true);

    final ViewConfiguration vc = ViewConfiguration.get(context);
    mTouchSlop = vc.getScaledTouchSlop();
    mMinFlingVelocity = vc.getScaledMininumFlingVelocity();
    mMaxFlingVelocity = vc.getScaleMaximumFlingVelocity();
    setWillNotDraw(getOverScrollMode() == View.OVER_SCROLL_NEVER);

    //设置动画结束时的监听器，这里实现了ItemAnimator.ItemAnimatorListener接口
    mItemAnimator.setListener(mItemAnimatorListener);

    //初始化AdapterHelper，它是一个工具类，用来对Adapter里的操作进行排队和处理
    initAdapterHelper();
    //初始化ChildHelper，它是一个工具类，用来向RecyclerView移除和添加子View
    initChildrenHelper();



}
```

RecyclerView初始化了两个重要的工具类

- AdapterHelper：用来对Adapter里的操作进行排队和处理
- ChildHelper：用来向RecyclerView移除

## setLayoutManager()

```java
public class RecyclerView extends ViewGroup {
    public void setLayoutManager(LayoutManager layout) {
        if (layout == mLayout) {
            return;
        }
        //停止滑动
        stopScroll();
        if (mLayout != null) {
            //停止所有动画
            if (mItemAnimator != null) {
                mItemAnimator.endAnimations();
            }
            //移除当前LayoutManager里的View，并稍后在mRecycler里复用
            mLayout.removeAndRecycleAllViews(mRecycler);
            //移除Scrap View，Scrap View指的
            mLayout.removeAndRecycleScrapInt(mRecycler);
            //清空mRecycler
            mRecycler.clear();
            if (mIsAttached) {
                mLayout.dispatchDetachedFromWindow(this, mRecycler);
            }
            mLayout.setRecyclerView(null);
            mLayout = null;
        } else {
            mRecycler.clear();
        }
        //移除所有的子View（包括处于hidden状态的View）
        mChildHelper.removeAllViewsUnfiltered();
        mLayout = layout;
        if (layout != null) {
            if (layout.mRecyclerView != null) {
                throw new IllegalArgumentException("LayoutManager " + layout
                        + " is already attached to a RecyclerView:"
                        + layout.mRecyclerView.exceptionLabel());
            }
            //从新给mLayout设置当前RecyclerView的引用
            mLayout.setRecyclerView(this);
            if (misAttached) {
                // 处理attach到Window
                mLayout.dispatchAttachToWindow(this);
            }
        }
        // 更新View的缓存数目
        mRecycler.updateViewCacheSize();
        //重新布局
        requestLayout();
    }
}
```

LayoutManager只负责对View进行布局，而承担管理View责任的是Recycler，RecyclerView实现了View复用的功能

- removeAndRecycleAllViews(Recycler recycler)：移除当前LayoutManager里的View，并稍后再mRecycler里复用

```java
public void removeAndReccyleAllViews(Recycler recycler) {
    for(int i = getChildCount() - 1; i >= 0; i--) {
        final View view = getChildAt(i);
        if (!getChildViewHolderInt(view).shouldIgnore()) {
            removeAndRecycleViewAt(i, recycler);
        }
    }
}
```

- removeAndRecycleScrapInt(Recycler recycler)：移除Scrap View，Scrap View指的是那些暂时处理分离状态的View后面可能还会重新使用

```java
void removeAndRecycleScrapInt(Recycler recycler) {
    finalt int scapCount = recycler.getScrapCount();
    for (int i = scrapCount - 1; i >= 0; i--) {
        final View scrap = recycler.getScrapViewAt(i);
        final ViewHolder vh = getChildViewHolderInt(scrap);
        if (vh.shouldIgnore()) {
            continue;
        }


        vh.setIsRecyclable(false);
        if (vh.isTmDetached()) {
            mRecyclerView.removeDetachedView(scrap, false);
        }
        if (mRecyclerView.mItemAnimator != null) {
            mRecyclerView.mItemAnimator.endAnimation(vh);
        }
        vh.setIsRecyclable(true);
        recycler.quickRecycleScapView(scrap);
    }
    recycler.cleanScrap();
    if (scapCount > 0) {
        mRecyclerView.invalidate();
    }
}
```

## setAdapter()

```java
public void setAdapter(Adapter adapter) {
    //停止当前的布局和滑动
    setLayoutFrozen(false);
    //设置Adapter，这里有两个boolean ，第一个表示是否使用当前相同的ViewHolder
    // 第二个表示是否移除和复用当前所有的View，当第一个 booleana为false时候，第二个boolean会被忽略
    setAdapterInternal(adapter, false, true);
    // 请求重新进行布局
    requestLayout();
}

private void setAdapterInternal(Adapter adapter, boolean compatibleWithPrevious, boolean removeAndRecycleViews) {
    if (mAdapter != null) {
        // 先释放掉原来数据和View
        mAdapter.unregisterAdapterDataObserver(mObserver);
        mAdapter.onDetachedFromRecyclerView(this);
    }
    //停止动画以及做一些清空操作，准备重新设置Adapter
    if (!compatibleWithPrevious || removeAndRecyclerViews) {
        removeAndRecyclerViews();
    }
    mAdapterHelper.reset();
    final Adapter oldAdapter = mAdapter;
    mAdapter = adapter;
    if (adapter != null) {
        // 重新注册数据监测DataObserver以及attach到RecyclerView
        adapter.registerAdapterDataObserver(mObserver);
        adapter.onAttachedToRecyclerView(this);
    }
    if (mLayout != null) {
        mLayout.onAdapterChanged(oldAdapter, mAdapter);
    }
    //触发Recycler.onAdapterChanged
    mRecycler.onAdapterChanged(oldAdapter, mAdapter, compatibleWithPrevious);
    mState.mStructrueChanged = true;
    markKnownViewsInvalid();
}
```

- 对以前的Adapter做一些清空操作，停止滑动和动画，准备好重新设置Adapter
- 重新设置Adapter，重新注册AdapterDataObserver以及关联到RecyclerView，并触发Recycler的onAdapterChanged()方法。通知Adapter已经发生改变，最终触发重新布局的操作

## RecyclerView布局策略管理器LayoutManager

>LayoutManager是一个抽象类。它主要用来测量和布局子View，滚动子View以及决定何时回收用户不可见的子View

关键方法：

- onLayoutChildren(Recycler recycler, State state)：对子View进行布局
- scrollHorizontallyBy(int dx, Recycler recycler, State state)：处理水平方向的滑动
- scrollVerticallyBy(int dy, Recycler recycler, State state)：处理竖直方向的滑动

RecyclerView已经处理了和触摸相关的事件，当RecyclerView上下滑动的时候，滑动的偏移量会传入这两个方法，dx/dy

LayoutManager的子类完成以下三件事：

- 将子View移动到适当的位置
- 处理子View移动后的添加/删除逻辑
- 返回移动后的实际距离，RecyclerView会根据这个距离判断是否触屏到边界

## LinearLayoutManager

### onLayoutChildren

```java
public void onLayoutChild(RecyclerView.Recycler recycler, RecyclerView.State state) {
    // 1. 从子View中查找出一个锚点坐标和锚点位置
    // 2. 从底部开始堆积，直到填充到首端
    // 3. 从顶部开始堆积，直到填充到微博
}

```

### scrollHorizontallyBy/scrollVerticallyBy

- 旧View的回收与新View的添加
- 其他View重新布局，偏移到合适的位置。

|||
|:---:|:---:|
|RecyclerViewDataObserver|数据观察器|
| Recycler|View循环复用系统，核心部件|
| SaveState|RecyclerView状态|
| AdapterHelper|适配器更新|
| ChildHelper|管理子View|
| ViewInfoStore|存储子VIEW的动画信息|
| Adapter|数据适配器|
|LayoutManager|负责子VIEW的布局，核心部件|
|ItemAnimator|Item动画|
|ViewFlinger|快速滑动管理|
|NestedScrollingChildHelper|管理子View嵌套滑动|

RecyclerView的Measure过程

```java
@Override
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        // 1.没有LayoutManager的情况
        defaultOnMeasure(widthSpec, heightSpec)
    }
    if (mLayout.mAutoMeasure) {
        // 2. 有LayoutManager并开启了自动测量
    } else {
        // 3. 有LayoutManager但没有开启自动测量

    }
}
```

RecyclerView的自动测绘过程

```java
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout.mAutoMeasure) {
        // 1. 首先执行的LayoutManager的onMeasure方法
        // 2. 检查width和height都已经是精确值的嘶吼，就不用再跟进内容进行计算所需要的width和height
        final int widthMode = MeasureSpec.getMode(widthSpec);
        final int heightMode = MeausreSpec.getMode(heightSpec);
        final boolean skipMeasure = widthMode = MeasureSpec.EXACTLY 
            && heightMode == MeasureSpec.EXACTLY;
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
        if (skipMeasure || mAdapter == null) {
            return;
        }
        // 开启布局流程计算出所有Child的边界
        // 然后根据计算出的child的边界计算出RecyclerView的所需width和heght
        // 检查是否需要再次测量
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
        }
        mLayout.setMeasureSpecs(widthSpec, heightSpec);
        mState.mIsMeasuring = true;
        dispatchLayoutStep2();
        // 布局过程结束，根据Childrend中的边界信息计算并设置RecyclerView长宽的测量值
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

        // 检查是否需要再次测量。
        if (mLayout.shouldMeasureTwice()) {
            mLayout.setMeasureSpecs(MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
            MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
            mState.mIsMeasuring = true;
            dispatchLayoutStep2();
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
        } else {

        }
    }
}
```

RecyclerView的非自动测绘流程

```java
if (mHasFixedSize) {
    // 如果RecyclerView已经设置Size固定，则执行LayoutManager onMeasure
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    return;
// 1. 如果过在测量的过程中有数据更新，则先处理更新的数据
// 2.执行自定义测量流程，需要自定义LayoutManager实现onMeasure方法
    if (mAdapterUpdateDuringMeasure) {
        eatRequestLayout();
        onEnterLayoutOrScroll();
        processAdapterUpdatesAndSetAnimationFlags();

        if (mState.mRunPredictiveAnimations) {
            mState.mInPreLayout = true;
        } else {
            mAdapterHelper.consumeUpdatesInOnePass();
            mState.mInPreLayout = false;
        }
        mAdapterUpdateDuringMeasure = false;
        resumeRequestLayout(false);
    } else if (mState.mRunPredictiveAnimation) {
        setMeasuredDimension(getMeasureWidth(), getMeasureHeight());
        return;
    }
    // 处理完新更新的数据，执行自定义测量操作
    if (mAdapter != null) {
        mStae.mItemCount = mAdapter.getItemCount();
    } else {
        mState.mItemCount = 0;
    }
    eatRquestLayout();
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    resumeRequestLayout(false);
    mState.mInPreLayout = false;
}

```

Android默认提供的RecyclerView支持线性布局、网格布局、瀑布流布局三种，同时能够控制横向还是纵向滚动。

- ViewHolder的编写规范
- RecyclerView复用Item的工作已经完成
- RecyclerView需要多出一步LayoutManager的设置工作

## 缓存机制对比

1. 层级不同

RecyclerView比ListView多两级缓存，支持多个ItemView缓存，支持开发者自定义缓存处理逻辑，支持所有RecyclerView共用同一个RecyclerViewPool

具体来说：

ListView(两级缓存)

|   ListView   | 是否需要回调createView | 是否需要回调bindView |生命周期|备注|
| :----------: | :--------------: | :------------: | :--------------------------------------: | :---------------: |
| mActiveViews |        否         |       否        |              onLayout函数周期内               | 用于屏幕内ItemView快速重用 |
| mScrapViews  |        否         |       是        | 与mAdapter一致，当mAdapter被更换时，mScrapViews即被清空 |                   |


RecyclerView(四级缓存)

|    RecyclerView     | 是否需要回调createView | 是否需要回调bindView |                   生命周期                   |                  备注                   |
| :-----------------: | :--------------: | :------------: | :--------------------------------------: | :-----------------------------------: |
|   mAttachedScrap    |        否         |       否        |              onLayout函数周期内               |           用于屏幕内ItemView快速重用           |
|     mCacheViews     |        否         |       是        | 与mAdapter一致，当mAdapter被更换时，mScrapViews即被清空 |        默认上限为2，即缓存屏幕外2个ItemView        |
| mViewCacheExtension |                  |                |                                          |          不直接使用，需要用户在定制，默认不实现          |
|    mRecyclerPool    |        否         |       是        |           与自身生命周期一致，不再被引用时即被释放           | 默认上限为5，技术上可以实现所有RecyclerViewPool共用同一个 |

基本上一致

1. mActiveViews和mAttachedScrap功能类似，意义在于快速重用屏幕上可见的列表项目itemView，而不需要重新createView和bindView
2. mScrapView和mCachedViews + mRecyclerViewPool功能相似，意义在于缓存离开屏幕的ItemView,目的是让即将进入屏幕的ItemView重用.
3. RecyclerView的优势在于：
    - mCacheViews的使用，可以做到屏幕外的列表项ItemView进入屏幕内时也无须bindView快速重用；
    - mRecyclerPool可以供多个RecyclerView共同使用，在特定场景下，如viewpager+多个列表页下有优势。

## RecyclerView条目缓存机制

RecyclerView缓存基本上是通过三个内部类管理的，Recycler、RecycledViewPool和ViewCacheExtension

- Recycler:一个Recycler是负责管理称为碎片的视图或者已经detached的视图，从而实现View的复用。
- RecycledViewPool:可以让你在多个RecyclerView之间分享视图


|变量|作用|
|:---:|:---:|
|mChangedScrap|mRecyclerView分离的ViewHolder列表|
|mAttachedScrap|未与RecyclerView分离的ViewHolder列表|
|mCachedViews|ViewHolder缓存列表|
|mViewCacheExtension|开发者可以控制的ViewHolder缓存的帮助类|
|mRecyclerPool|ViewHolder缓存池|

### RecycledViewPool

RecycledViewPool类是用来缓存Item用的，是一个ViewHolder的缓存池，如果多个RecyclerView之间用
`setRecycledViewPool(RecycleViewPool)`设置同一个RecycledViewPool,他们就可以共享Item。其实
RecyclerViewPool的内部维护了一个Map，里面以不同的viewType为key存储了各自对应的ViewHolder集合，可以通过
提供的方法来修改内部缓存的viewHolder。

### ViewCacheExtension

这是一个需要开发者重写的类。在`Recycler.getScrapOrHiddenOrCachedHolderForPosition`方法获取View
Recycler先检查自己内部的`attached scrap`和一级缓存，再检查
`ViewCacheExtensien`最后检查`RecyclerViewPool`，从上面三个任何一个只要拿到View就不会调用下一个方法。

# LayoutManager

LayoutManager是RecyclerView用来管理子View布局的一个组件

1. 布局子视图
2. 在滚动过程中根据子视图在布局中所处位置，决定何时添加子视图和回收视图
3. 滚动子视图

其中，只有滚动子视图，才会需要对子视图回收或者添加，而添加
