# onTouchEvent
## 跟随手指拖动
```java
//实际上的滚动方法
if (overScrollBy(deltaX, 0, mScrollX, 0, range, 0,  
        mOverscrollDistance, 0, true)) {
```

```java
//为那些滑动会超过边界的View提供滑动处理
protected boolean overScrollBy(int deltaX, int deltaY,
            int scrollX, int scrollY,//当前滑动x/y方向上的滑动值
            int scrollRangeX, int scrollRangeY,//最大的滑动距离
            int maxOverScrollX, int maxOverScrollY,//可以超过滑动距离多少
            boolean isTouchEvent) {
        final int overScrollMode = mOverScrollMode;
        final boolean canScrollHorizontal =
                computeHorizontalScrollRange() > computeHorizontalScrollExtent();
        final boolean canScrollVertical =
                computeVerticalScrollRange() > computeVerticalScrollExtent();
        final boolean overScrollHorizontal = overScrollMode == OVER_SCROLL_ALWAYS ||
                (overScrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && canScrollHorizontal);
        final boolean overScrollVertical = overScrollMode == OVER_SCROLL_ALWAYS ||
                (overScrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && canScrollVertical);

        int newScrollX = scrollX + deltaX;
        if (!overScrollHorizontal) {
            maxOverScrollX = 0;
        }

        int newScrollY = scrollY + deltaY;
        if (!overScrollVertical) {
            maxOverScrollY = 0;
        }

        // Clamp values if at the limits and record
        final int left = -maxOverScrollX;
        final int right = maxOverScrollX + scrollRangeX;
        final int top = -maxOverScrollY;
        final int bottom = maxOverScrollY + scrollRangeY;

        boolean clampedX = false;
        if (newScrollX > right) {
            newScrollX = right;
            clampedX = true;
        } else if (newScrollX < left) {
            newScrollX = left;
            clampedX = true;
        }

        boolean clampedY = false;
        if (newScrollY > bottom) {
            newScrollY = bottom;
            clampedY = true;
        } else if (newScrollY < top) {
            newScrollY = top;
            clampedY = true;
        }
//需要重写的方法
        onOverScrolled(newScrollX, newScrollY, clampedX, clampedY);

        return clampedX || clampedY;
    }
```

HorizontalScrollView重写了*onOverScrolled*
```java
protected void onOverScrolled(int scrollX, int scrollY,
            boolean clampedX, boolean clampedY) {
        // Treat animating scrolls differently; see #computeScroll() for why.
//处于fling状态
        if (!mScroller.isFinished()) {
            final int oldX = mScrollX;
            final int oldY = mScrollY;
            mScrollX = scrollX;
            mScrollY = scrollY;
            invalidateParentIfNeeded();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (clampedX) {
                mScroller.springBack(mScrollX, mScrollY, 0, getScrollRange(), 0, 0);
            }
        } else {
            super.scrollTo(scrollX, scrollY);
        }

        awakenScrollBars();
    }

```
## 开启惯性滑动
