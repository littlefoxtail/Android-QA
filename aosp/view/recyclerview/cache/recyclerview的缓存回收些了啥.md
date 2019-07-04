# 回收场景

```java
public class RecyclerView {
    public boolean onTouchEvent(MotionEvent e) {
        case MotionEvent.ACTION_MOVE: {
            if (scrollByInternal(
                canScrollHorizontally ? dx : 0,
                canScrollVertically ? dx : 0,
                vtev)) {

            }
        }
    }

    boolean scrollByInternal(int x, int y, MotionEvent ev) {
        if (mAdapter != null) {
            scrollStep(x, y, mScrollStepConsumed);
        }
    }

    private void scrollStep() {
        if (dx != 0) {
            consumedX = mLayout.scrollHorizontallyBy(dx, mRecycler, mState);
        }
        if (dy != 0) {
            consumedY = mLayout.scrollVerticallyBy(dy, mRecycler, mState);
        }
    }
}
```

```java
public class LinearLayoutManager {
    static class LayoutState {
        /*
         *需要填充的像素点
         */
        int mAvailable;
        /*
         * 在不添加新的View情况，可以滚动的距离，
         */
        int scrollingOffset;
    }

    public int scrollVerticallyBy() {
        return scrollBy(...);
    }

    int scrllBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
        //dx > 0 列表向上滚动
        final layoutDirection = dy > 0 ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
        //更新LayoutState
        updateLayoutState(...)
        final int consumed = mLayoutState.mScrollingOffset
                    + fill(recycler, mLayoutState, state, false);
    }

    int fill(...) {

        //不断循环获取新的表项用于填充，直到没有填充空间
        while((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
            //填充新的表项
            layoutChunk(..)

            if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
                //在当前滚动偏移量基础上追加因新表项插入增加的像素
                layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
                if (layoutState.mAvailable < 0) {
                    layoutState.mScrollingOffset += layoutState.mAvailable;
                }
                //回收表项
                recycleByLayoutState(recycler, layoutState);
            }
        }
    }
}
```

在不断获取新表项用于填充的同时也在回收表项

```java
public class LinearLayoutManager {


    private void recycleByLayoutState(..) {
        if (!layoutState.mRecycle || layoutState.mInfinite) {
            return;
        }
        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            recycleViewsFromEnd(..);
        } else {
            //列表向上滚动的时候
            recycleViewsFromStart(..);
        }
    }
}
```

RecyclerView的回收分两个方向：1. 从列表头回收 2.从列表尾回收。就以“从列表头回收”为研究对象分析下RecyclerView在滚动时到底是怎么判断“哪些表项应该被回收？”。
（“从列表头回收表项”所对应的场景是：手指上滑，列表向下滚动，新的表项逐个插入到列表尾部，列表头部的表项逐个被回收。）

## 回收哪些表项

```java
class LinearLayoutManager {
    private void recycleViewsFromStart() {
        else {
            for in in childCount:
                View child = getChildAt(i);
                //当前表项的尾部相对于列表头部的坐标
                if (mOrientationHelper.getDecoratedEnd(child) > limit
                    || mOrientationHelper.getTransformedEndWithdecoration(child) > limit) {
                        recycleChildren(recycler, 0, i);
                        return;
                    }
        }
    }

    private void updateLayoutState(..) {
        if (layoutDirection == LayoutState.LAYOUT_END) {
            //最后一个孩子
            final View child = getChildClosestToEnd();
        }
    }
}
```

![recyclerview_offset](/img/recyclerview_offset)
