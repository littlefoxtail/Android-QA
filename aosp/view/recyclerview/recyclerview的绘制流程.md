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
        // 2. 有LayoutManager并开启了自动测量
    } else {
        // 3. 有LayoutManager但没有开启自动测量

    }
}
```

RecyclerView的自动测绘过程

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

/**
 * 处理adapter更新
 * 决定哪些动画需要别执行
 * 保存当前View的信息
 * 如果必要的话，执行上一个Layout的操作并且保存他的信息
 */
private void dispatchLayoutStep1() {

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
        // mAnchorInfo.mValid，这里mAnchorInfo就是我们要的锚点。mValid的默认值是false，一次测量之后设为true，onLayout完成后会回调执行reset方法，又变为false
// 再就是mAnchorInfo.mLayoutFromEnd。
        if (!mAnchorInfo.mValid || mPendingScrollPosition != RecyclerView.NO_POSITION
            || mPendingSavedState != null) {
                //mStackFromEnd、mShouldReverseLayout默认是false，除非手动调用setStackFromEnd()方法，两个都会false，异或则为false
                mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
                // 计算锚点位置和坐标
                updateAnchorInfoForLayout(recycler, state, mAnchoriInfo);
        }

        {
            //正常绘制流程，先往下绘制，再往上绘制
            updateLayoutStateToFillEnd(mAnchorInfo);
            fill(..);

            updateLayoutStateToFillEnd(mAnchorInfo);
            fill(..);
        }
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
