# 二RecyclerView的滑动机制

## Move事件

```java
// 1. 根据Move事件产生的x、y坐标来计算dx、dy
final int index = e.findPointerIndex(mScrollPointerId);
if (index < 0) {
    Log.e(TAG, "Error processing scroll; pointer index for id "
            + mScrollPointerId + " not found. Did any MotionEvents get skipped?");
    return false;
}

final int x = (int) (e.getX(index) + 0.5f);
final int y = (int) (e.getY(index) + 0.5f);
int dx = mLastTouchX - x;
int dy = mLastTouchY - y;

if (mScrollState != SCROLL_STATE_DRAGGING) {
    boolean startScroll = false;
    if (canScrollHorizontally) {
        if (dx > 0) {
            dx = Math.max(0, dx - mTouchSlop);
        } else {
            dx = Math.min(0, dx + mTouchSlop);
        }
        if (dx != 0) {
            startScroll = true;
        }
    }
    if (canScrollVertically) {
        if (dy > 0) {
            dy = Math.max(0, dy - mTouchSlop);
        } else {
            dy = Math.min(0, dy + mTouchSlop);
        }
        if (dy != 0) {
            startScroll = true;
        }
    }
    if (startScroll) {
        setScrollState(SCROLL_STATE_DRAGGING);
    }
}

if (mScrollState == SCROLL_STATE_DRAGGING) {
    mReusableIntPair[0] = 0;
    mReusableIntPair[1] = 0;
    // 2 根据dispatchNestedPreScroll询问父View是否优先处理滑动事件，如果要消耗，
    // dx和dy分别会减去父View消耗的那部分距离
    if (dispatchNestedPreScroll(
            canScrollHorizontally ? dx : 0,
            canScrollVertically ? dy : 0,
            mReusableIntPair, mScrollOffset, TYPE_TOUCH
    )) {
        dx -= mReusableIntPair[0];
        dy -= mReusableIntPair[1];
        // Updated the nested offsets
        mNestedOffsets[0] += mScrollOffset[0];
        mNestedOffsets[1] += mScrollOffset[1];
        // Scroll has initiated, prevent parents from intercepting
        getParent().requestDisallowInterceptTouchEvent(true);
    }

    mLastTouchX = x - mScrollOffset[0];
    mLastTouchY = y - mScrollOffset[1];
    // 3.根据情况来判断RecyclerView是垂直滑动还是水平滑动
    // 最终是调用scrollByInternal方法来实现滑动的效果的
    if (scrollByInternal(
            canScrollHorizontally ? dx : 0,
            canScrollVertically ? dy : 0,
            e)) {
        getParent().requestDisallowInterceptTouchEvent(true);
    }
    // 4. 调用GapWorker的postFromTraversal来预取ViewHolder
    if (mGapWorker != null && (dx != 0 || dy != 0)) {
        mGapWorker.postFromTraversal(this, dx, dy);
    }
}
```

```java
// GapWorker
public void run() {
    long nextFrameNs = TimeUnit.MILLISECONDS.toNanos(latestFrameVsyncMs) + mFrameIntervalNs;
    prefetch(nextFrameNs);
}
```