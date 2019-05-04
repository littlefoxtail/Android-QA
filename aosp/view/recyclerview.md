# RecyclerView

RecyclerView是Android5.0提出的新UI控件，在限定的视图内展示大量的数据
A flexible view for providing a limited window into a large data set

## RecyclerView的设计结构

- RecyclerView创建流程
- RecyclerView布局策略管理器LayoutManager
- RecyclerView视图描述者ViewHolder
- RecyclerView视图复用器RecyclerView
- RecyclerView视图适配器Adapter
- RecyclerView视图动画ItemAdapter
- RecyclerView视图分隔条ItemDecoration

RecyclerView继承于ViewGroup，实现ScrollingView与NestedScrollingChild接口

对比ListView：
ListView相比RecyclerView，有一些优点：

- addHeaderView()，addFootderView()添加头视图和尾视图
- 通过"android:divider"设置自定义分割线
- setOnItemClickListener和setOnItemLongClickListener设置点击事件和长按事件

这些功能在RecyclerView中都没有直接的接口，要自己实现，因此如果只是简单的显示功能，ListView无疑更简单

||Item复用方面|样式丰富方面|效果增强|代码内聚|局部刷新|
|:---:|:---:|:---:|:---:|:---:|:---:|
|RecyclerView|内置了RecyclerViewPool、多级缓存、ViewHolder|通过支持水平、垂直和表格列表及其他更复杂形势|内置ItemDecoration和ItemAnimator，可以自定义绘制ItemView之间的一些特殊UI或Item项数据变化时的动画效果|将功能密切相关的类写成内部类|有|
|ListView|需要手动添加ViewHolder|只支持具体某一种|将这些特殊UI作为ItemView的一部分|无|无|

![recyclerview_structure](/img/recyclerview_structure.png)

### 设计模式

- 通过桥接模式，使得RecyclerView将布局方式独立成LayoutManager，实现对布局的定制化
- 通过组合模式，使RecycleView通过dispatchLayout对item view进行布局绘制
- 通过适配器模式，ViewHolder将RecyclerView与ItemView联系起来，使得RecycleView方便操作itemView
- 通过观察者模式，给ViewHolder注册观察者，当调用notifyDataSetChanged时，就能重新绘制

## ViewHolder

对于Adapter来说，一个ViewHolder就对应一个data。它也是Recycler缓存池的基本单元。

```java
class ViewHolder {
    public final View itemView; //会被当做child view来add到RecyclerView中
    int mPosition = NO_POSITION;//标记当前的ViewHolder在Adapter中所处的位置
    int mItemViewType = INVALID_TYPE;//这个ViewHolder的Type，在ViewHolder保存到RecyclerPool时，主要靠这个类型来对ViewHolder做复用
    int mFlags;//标记ViewHolder的状态，比如FLAG_BOUND(显示在屏幕上)、FLAG_INVALID(无效，想要使用必须rebound)
}
```

## Adapter

它的工作把data和View绑定，即上面说的一个data对应一个ViewHolder。主要负责ViewHolder的创建以及数据变化时通过RecyclerView。

1. 它引用着一个数据源集合dataSource
2. getItemCount()用来告诉RecyclerView展示的总条目
3. 它并不是直接隐射data->ViewHolder，而是data postion->data type->viewholder。所以对于ViewHolder来说，它知道的只是它的view type

## AdapterDataObservable

Adapter是数据源的直接接触者，当数据源发生变化时，它需要通知给RecyclerView。这里使用的模式是观察者模式。AdapterDataObservable是数据源变化时的被观察者。

RecyclerViewDataObserver是观察者。开发中我们通常使用adapter.notifyXXX()来刷新UI。

## RecyclerViewDataObserver

它是`RecyclerView`用来监听`Adapter`数据变化的观察者

## LayoutManager

它是`RecyclerView`的布局管理者，`RecyclerView`在`onLayout`时，会利用它来layoutChildren，它决定RecyclerView中的子View的摆放规则。但不知如此，它做的工作还有：

1. 测量子View
2. 对子View进行布局
3. 对子View进行回收
4. 子View动画的调度
5. 负责RecyclerView滚动的实现

## Recycler

[RecyclerView的缓存机制](recyclerview的缓存机制.md)

对于LayoutManager来说，它是ViewHolder的提供者。对于RecyclerView来说，它是ViewHolder的管理者，是RecyclerView最核心的实现。

### scrap list

```java
final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
ArrayList<ViewHolder> mChangedScrap = null;
```

`mAttachedScrap`和`mChangedScrap`中的View复用主要作用在`adapter.notifyXXX`时。这时候就会产生很多scrap状态的view。也可以把它理解为一个ViewHolder的缓存。不过这里获取ViewHolder时完全根据ViewHolder的position而不是item type。

### mCacheViews

可以把它理解为RecyclerView的一级缓存。它的默认大小为3，从中可以根据item type

## 创建流程

### 构造

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

### setLayoutManager()

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





Android默认提供的RecyclerView支持线性布局、网格布局、瀑布流布局三种，同时能够控制横向还是纵向滚动。

- ViewHolder的编写规范
- RecyclerView复用Item的工作已经完成
- RecyclerView需要多出一步LayoutManager的设置工作

# LayoutManager

LayoutManager是RecyclerView用来管理子View布局的一个组件

1. 布局子视图
2. 在滚动过程中根据子视图在布局中所处位置，决定何时添加子视图和回收视图
3. 滚动子视图

其中，只有滚动子视图，才会需要对子视图回收或者添加，而添加
