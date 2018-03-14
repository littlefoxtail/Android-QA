# RecyclerView的设计结构
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

* ViewHolder的编写规范
* RecyclerView复用Item的工作已经完成
* RecyclerView需要多出一步LayoutManager的设置工作

# 缓存机制对比
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
1. mActiveViews和mAttachedScrap功能类似，意义在于快速重用屏幕上可见的列表项目itemView，而不需要重新
createView和bindView
2. mScrapView和mCachedViews + mRecyclerViewPool功能相似，意义在于缓存离开屏幕的ItemView,目的是让即将进入屏幕的ItemView重用.
3. RecyclerView的优势在于
    * mCacheViews的使用，可以做到屏幕外的列表项ItemView进入屏幕内时也无须bindView快速重用；
    * mRecyclerPool可以供多个RecyclerView共同使用，在特定场景下，如viewpager+多个列表页下有优势。
    
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

