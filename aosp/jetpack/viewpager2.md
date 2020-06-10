# ViewPager2

原由：

1. 了解`ViewPager2`是怎么讲`RecyclerView`的滑动事件转变为`ViewPager`的页面滑动事件。
2. 了解怎么使用`RecyclerView`来加载Fragment

方面：

1. `PagerSnapHelper`的源码分析，了解它内部的原理，是如何实现ViewPager的效果。
2. 各种组件的分析，包含`ScrollEventAdapter`、`PageTransformerAdapter`。
3. `FragmentStateAdapter`的源码分析，主要是了解`Adapter`是怎么加载`Fragment`

## 基本分析

```java
// ViewPager2
private void initialize(..) {
    // 创建滑动事件转换器的对象
    mScrollEventAdapter = new ScrollEventAdapter(this);
    // 用来实现页面切换的基本效果
    mPagerSnapHelper = new PagerSnapHelperImpl();

}
```

## PagerSnapHelper

```java
// PagerSnapHelper
public int findTargetSnapPosition(..) {
    // 找到当前View相邻的View，包括左相邻和右相邻，并且计算滑动的距离
    for (int i = 0; i < childCount; i++) {
        final View child = layoutManager.getChildAt(i);
        if (child == null) {
            continue;
        }
        final int distance = distanceToCenter(..);
    }
    //根据滑动的方向来返回的相应位置
    final boolean forwardDirection = isForwardFling(..);
    if (forwardDirection && closestChildAfterCenter != null) {
        return layoutManager.getPosition(closestChildAfterCenter);
    } else if (!forwardDirection && closestChildAfterCenter != null) {
        return layoutManager.getPosition(closestChildAfterCenter);
    }

    View visibleView = forwardDirection ? closestChildBeforeCenter : closestChildBeforeCenter;
    if (visibleView == null) {
        return RecyclerView.NO_POSITION;
    }
    int visiblePostion = layoutManager.getPosition(visibleView);
    int snapToPosition = visiblePosition
                + (isReverseLayout(layoutManager) == forwardDirection ? -1 : +1);

    if (snapToPosition < 0 || snapToPosition >= itemCount) {
        return RecyclerView.NO_POSITION;
    }
    return snapToPosition;
}
```

## ScrollEventAdapter

`ScrollEventAdapter`的作用将`RecyclerView`的滑动事件转为`ViewPager2`的页面滑动事件。

### onScrollStateChanged方法

滑动状态发生变化，这个方法就会被调用

主要分为3个阶段：

1. 开始拖动，会调用`startDrag`表示拖动开始。
2. 拖动手势的释放，此时`ViewPager2`会准备滑动到正确的位置。
3. 滑动结束，此时`ScrollEventAdapter`会调用相关的方法更新状态。

```java
// ScrollEventAdaper
public void onScrollStateChanged(..) {
    // 1.开始拖动
    if ((mAdapterState != STATE_IN_PROGRESS_MANUAL_DRAG
            || mScrollState != SCROLL_STATE_DRAGGING)
            && newState == RecyclerView.SCROLL_STATE_DRAGGING) {
        startDrag(false);
        return;
    }
    // 2.拖动手势的释放
    if (isInAnyDraggingState() && newState == RecyclerView.SCROLL_STATE_SETTLING) {
        // Only go through the settling phase if the drag actually moved the page
        if (mScrollHappened) {
            dispatchStateChanged(SCROLL_STATE_SETTLING);
            // Determine target page and dispatch onPageSelected on next scroll event
            mDispatchSelected = true;
        }
        return;
    }
    // 3. 滑动结束
    if (isInAnyDraggingState() && newState == RecyclerView.SCROLL_STATE_IDLE) {
        boolean dispatchIdle = false;
        updateScrollEventValues();
        // 如果在拖动期间为产生移动距离
        if (!mScrollHappened) {
            if (mScrollValues.mPosition != RecyclerView.NO_POSITION) {
                dispatchScrolled(mScrollValues.mPosition, 0f, 0);
            }
            dispatchIdle = true;
        } else if (mScrollValues.mOffsetPx == 0) {
            dispatchIdle = true;
            if (mDragStartPosition != mScrollValues.mPosition) {
                dispatchSelected(mScrollValues.mPosition);
            }
        }
        if (dispatchIdle) {
            dispatchStateChanged(SCROLL_STATE_IDLE);
            resetState();
        }
    }

    if (mAdapterState == STATE_IN_PROGRESS_SMOOTH_SCROLL
            && newState == RecyclerView.SCROLL_STATE_IDLE && mDataSetChangeHappened) {
        updateScrollEventValues();
        if (mScrollValues.mOffsetPx == 0) {
            if (mTarget != mScrollValues.mPosition) {
                dispatchSelected(
                        mScrollValues.mPosition == NO_POSITION ? 0 : mScrollValues.mPosition);
            }
            dispatchStateChanged(SCROLL_STATE_IDLE);
            resetState();
        }
    }
}
```

### onScrolled方法

```java
// ScrollEventAdapter
public void onScrolled(..) {
    updateScrollEventValues();

    if (mDispatchSelected) {
        //拖动手势释放，ViewPager2正在滑动到正确的位置
        mDispatchSelected = false;

    } else if (mAdapterState == STATE_IDLE) {
        //调用了setAdapter
        dispatchSelected(mScrollValues.mPosition);
    }

}
```

1. 调用`updateScrollEventValues`方法更新`ScrollEventValue`里面的值。
2. 调用相关方法，更新状态。

## FragmentStateAdapter

1. onCreateViewHolder
2. onBindViewHolder
3. onViewAttachedToWindow
4. onViewRecycled
5. onFailedToRecyclerView

### onBindViewHolder

```java
public final void onBindViewHolder() {
    // position
    final long itemId = holder.getItemId();
    // ViewHolder中的FrameLayout的viewId，
    final int viewHolderId = holder.getContainer().getId();
    // 用于将viewHolderId保存在Adapter的LongSparseArray中所使用的Key值（位置）
    final Long boundItemId = itemForViewHolder(viewHolderId);//1
    mItemIdToViewHolder.put(itemId, viewHolderId);
}
```
