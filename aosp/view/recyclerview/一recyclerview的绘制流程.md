# 绘制流程

RecyclerView继承了ViewGroup，所以是一个自定义ViewGroup，所以最重要步骤：

1. 重写onMeasure用于确定自定义ViewGroup的大小
2. 重写onLayout用于布局子View的位置

![RecyclerView的绘制流程](/img/RecyclerView的绘制流程.png)

## onMeasure

```java
@Override
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        // 1.没有LayoutManager的情况
        defaultOnMeasure(widthSpec, heightSpec)
    }
    if (mLayout.mAutoMeasure) {
        // 2. 有LayoutManager并开启了自动测量，这种情况下，有可能会测量两次
    } else {
        // 3. 有LayoutManager但没有开启自动测量，
        // 为了RecyclerView支持wrap_content属性，系统提供的LayoutManager都是开启自动测量的

    }
}

```

RecyclerView的自动测绘过程：

`mState.mLayoutStep`取值：

|取值|含义|
|:--:|:--:|
|STEP_START|默认值，RecyclerView还未经历`dispatchLayoutStep1`，因为`dispatchLayoutStep1`调用之后变为了`STEP_LAYOUT`|
|STEP_LAYOUT|当mState.mLayoutStep为State.STEP_LAYOUT时，表示此时处于layout阶段，这个阶段会调用`dispatchLayoutStep2`方法layout RecyclerView的children。调用`dispatchLayoutStep2`方法之后，此时mState.mLayoutStep变为了State.STEP_ANIMATIONS|
|State.STEP_ANIMATIONS|当`mState.mLayoutStep`为`State.STEP_ANIMATIONS`时，表示`RecyclerView`处于第三个阶段，也就是执行动画阶段，也就是调用`dispatchLayoutStep3`方法。当`dispatchLayoutStep3`方法执行完毕之后，`mState.mLayoutStep`又变为了`State.STEP_START`|

```java
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout.mAutoMeasure) {//默认值一般都是true
        // 1. 首先执行的LayoutManager的onMeasure方法
        // 2. 检查width和height都已经是精确值的嘶吼，就不用再跟进内容进行计算所需要的width和height
        final int widthMode = MeasureSpec.getMode(widthSpec);
        final int heightMode = MeausreSpec.getMode(heightSpec);
        final boolean skipMeasure = widthMode = MeasureSpec.EXACTLY
            && heightMode == MeasureSpec.EXACTLY;
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
        if (skipMeasure || mAdapter == null) {//如果测量值是绝对值，则跳过measure过程，直接走layout
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

|方法|作用|
|:--:|:--:|
|dispatchLayoutStep1|1.处理`Adapter`更新；2.决定是否执行`ItemAnimator`；3.保存`ItemView`的动画信息|
|dispatchLayoutStep2|真正进行`children`的测量和布局|
|dispatchLayoutStep3|三大dispatchLayoutStep方法第三步。这个方法的作用执行在dispatchLayoutStep1方法里面保存的动画信息。本方法不是本文的介绍重点，后面在介绍ItemAnimator时，会重点分析这个方法。|

```java
/**
 * 处理adapter更新
 * 决定哪些动画需要别执行
 * 保存当前View的信息
 * 如果必要的话，执行上一个Layout的操作并且保存他的信息
 */
private void dispatchLayoutStep1() {
    processAdapterUpdatesAndSetAnimationFlags();
}

private void processAdapterUpdatesAndSetAnimationFlags() {
    //RecyclerView第一次加载数据时，是不会执行的动画
    mState.mRunSimpleAnimations = mFirstLayoutComplete
            && mItemAnimator != null
            && ..;

    mState.mRunPredictiveAnimations = mState.mRunSimpleAnimations
            && animationTypeSupported
}


private void dispatchLayoutStep2() {
    mState.mItemCount = mAdapter.getItemCount();
    // 通过这个来完成对childred的测量和布局
    mState.mInPreLayout = false;
    mLayout.onLayoutChildren(mRecycler, mState);
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

![fil](/img/fill.png)

```java
public class LinearLayoutManager {
    public void onLayoutChildren(..) {
        resolveShouldLayoutReverse();
        final View focused = getFocusedChild();
        // 第一步
        // mAnchorInfo.mValid，这里mAnchorInfo就是我们要的锚点。mValid的默认值是false，一次测量之后设为true，onLayout完成后会回调执行reset方法，又变为false
// 再就是mAnchorInfo.mLayoutFromEnd。
        if (!mAnchorInfo.mValid || mPendingScrollPosition != RecyclerView.NO_POSITION
            || mPendingSavedState != null) {
                //mStackFromEnd、mShouldReverseLayout默认是false，除非手动调用setStackFromEnd()方法，两个都会false，异或则为false
                mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
                // 计算锚点位置和坐标
                updateAnchorInfoForLayout(recycler, state, mAnchoriInfo);
        }
        // 第二步
        detachAndScrapAttachedViews(recycler);
        mLayoutState.mIsPreLayout = state.isPreLayout();
        // 第三步
        {
            //正常绘制流程，先往下绘制，再往上绘制
            updateLayoutStateToFillEnd(mAnchorInfo);
            fill(..);

            updateLayoutStateToFillEnd(mAnchorInfo);
            fill(..);
        }
    }

    private void updateAnchorInfoForLayout(..) {
        // 第一种计算方式
        if (updateAnchorFromPendingData(state, anchorInfo)) {
            return;
        }
        // 第二种计算方式
        if (updateAnchorFromChildren(recycler, state, anchorInfo)) {
            return;
        }
        // 第三种计算方式
        anchorInfo.assignCoordinateFromPadding();
        anchorInfo.mPosition = mStackFromEnd ? state.getItemCount() - 1 : 0;
    }

    int fill(..) {
        layoutChunk(..);
        return start - layoutState.mAvailable;
    }

    void layoutChunk(..) {
        View view = layoutState.next(recycler);
        //测量ChildView
        measureChildWithMargins(view, 0, 0);
        layoutDecoratedWithMargins(view, left, top, right, bottom);
    }

    public class LayoutStae {
        View next(RecyclerView.Recycler recycler) {
            //默认mScrapList=null，但是执行layoutForPredictiveAnimations方法的时候不会为空
            if (mScrapList != null) {
                return nextViewFromScrapList();
            }
            //重要，从recycler获得View,mScrapList是被LayoutManager持有，recycler是被RecyclerView持有
            final View view = recycler.getViewForPosition(mCurrentPosition);
            mCurrentPosition += mItemDirection;
            return view;
        }
    }
}
```

`updateAnchorInfoForLayout`锚点计算：

1. 第一种计算方式，RecyclerView被重建，1. 期间回调了`onSaveInstanceState`方法，目的是为了恢复上次的布局。 2.调用了`scrollToPosition`之类的方法，所以目的是让`RecyclerView`滚到准确的位子
2. 第二种计算方法，从Children上面来计算锚点信息。这种计算方式也有两种情况：1. 如果当前有拥有焦点的Child，那么有当前有焦点的Child的位置来计算锚点；2. 如果没有child拥有焦点，那么根据布局方向(此时布局方向由mLayoutFromEnd来决定)获取可见的第一个ItemView或者最后一个ItemView。
3. 如果前面两种方式都计算失败了，那么采用第三种计算方式，也就是默认的计算方式。

`layoutChunk`简单概述：

1. 调用LayoutState的`next`方法获得一个`ItemView`。
2. 如果`RecyclerView`是第一次布局Childred的话（`layoutState.mScrapList == null`为true），会先调用addView，将`View`添加到`RecyclerView`里面去。
3. 调用`measureChildWithMargins`方法，测量每个`ItemView`的宽高。
4. 调用`layoutDecoratedWithMargins`方法，布局`ItemView`。

## layout

```java
// RecyclerView
protected void onLayout(..) {
    dipatchLayout();
}

//RecyclerView
void dispatchLayout() {

}
```

`dispatchLayout`方法保证`RecyclerView`必须经历一个过程--`dispatchLayoutStep1`、`dispatchLayoutStep2`、`dispatchLayoutStep3`。

`dispatchLayoutStep3`主要是做item的动画，也就是`ItemAnimator`的执行。

## draw

1. 调用`super.draw`方法。
2. 如果需要的话，调用`ItemDecoration`的`onDrawOver`方法
3. 如果`RecyclerView`调用了`setClipTopadding`，会实现一种特殊的滑动效果

```java
public void draw(Canvas c) {
    // 1
    super.draw(c);
// 2
    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDrawOver(c, this, mState);
    }
// 3
    // TODO If padding is not 0 and clipChildrenToPadding is false, to draw glows properly, we
        // need find children closest to edges. Not sure if it is worth the effort.

}

public void onDraw(Canvas c) {
    super.onDraw(c);
    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDraw(c, this, mState);
    }
}
```

## 总结

1. RecyclerView的measure过程分为三种情况，每种情况都有执行过程。通常来说，我们都会走自动测量的过程。
2. 自动测量里面需要分清楚mState.mLayoutStep状态值，因为根据不同的状态值调用不同的dispatchLayoutStep方法。
3. layout过程也根据mState.mLayoutStep状态来调用不同的dispatchLayoutStep方法。
4. draw过程主要做了四件事：1.绘制ItemDecoration的onDraw部分;2.绘制Children;3.绘制ItemDecoration的drawOver部分;4. 根据mClipToPadding的值来判断是否进行特殊绘制。
