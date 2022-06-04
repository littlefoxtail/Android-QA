NestedScrollingParent、NestedScrollingChild逻辑上分别对应外部和内部角色

# NestedScrollView#onTouch#ACTION_DOWN
```java
public boolean onTouchEvent(MotionEvent ev) {
	case MotionEvent.ACTION_DOWN: {
//子View接收滑动事件，通知父控件
		startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH);
```

## NestedScrollingChildHelper
## startNestedScroll
在子View准备滑动时，会调用该方法；屏幕触摸事件中，相当于ACTION_DOWN中调用该方法。
对NestedScrollingChild接口作了代理
```java
public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {  
//判断是否找到配合处理滑动的NestedScrollingParent
    if (hasNestedScrollingParent(type)) {  
        // Already in progress  
        return true;  
    }  
    if (isNestedScrollingEnabled()) {  
        ViewParent p = mView.getParent();  
        View child = mView;  
        while (p != null) {  
//循环往上找配合处理滑动的NestedScrollingParent
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {  
                setNestedScrollingParentForType(type, p); 
//调用p的onNestedScrollAccepted方法
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);  
                return true;  
            }  
            if (p instanceof View) {  
                child = (View) p;  
            }  
            p = p.getParent();  
        }  
    }  
    return false;  
}
```
# NestedScrollView#onTouch#ACTION_MOVE
滑动事件：
- 在每次滑动前调用前的dispatchNestedPreScroll
- 拖动状态下分发dispatchNestedScroll
```java
case MotionEvent.ACTION_MOVE:
//deltaY：垂直移动的距离，dy
	int deltaY = mLastMotionY - y;  
	deltaY -= releaseVerticalGlow(deltaY, ev.getX(activePointerIndex));  
//子View准备滑动，通知父控件
	if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {  
	    final ViewParent parent = getParent();  
	    if (parent != null) {  
	        parent.requestDisallowInterceptTouchEvent(true);  
	    }  
	    mIsBeingDragged = true;  
	    if (deltaY > 0) {  
	        deltaY -= mTouchSlop;  
	    } else {  
	        deltaY += mTouchSlop;  
	    }  
	}
//在拖动状态下
	if (mIsBeingDragged) {  
    // Start with nested pre scrolling  
//子View消费滑动后，将消费距离详情通知父控件
    if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset,  
            ViewCompat.TYPE_TOUCH)) {  
        deltaY -= mScrollConsumed[1];  
        mNestedYOffset += mScrollOffset[1];  
    }
```
## NestedScrollingChildHelper#dispatchNestedScrollInternal
```java
//根据触摸类型获取相应的控件
            final ViewParent parent = getNestedScrollingParentForType(type);
            if (parent == null) {
                return false;
            }
//如果存在滑动距离
            if (dxConsumed != 0 || dyConsumed != 0 || dxUnconsumed != 0 || dyUnconsumed != 0) {
                int startX = 0;
                int startY = 0;
                if (offsetInWindow != null) {
                    mView.getLocationInWindow(offsetInWindow);
                    startX = offsetInWindow[0];
                    startY = offsetInWindow[1];
                }

                if (consumed == null) {
                    consumed = getTempNestedScrollConsumed();
                    consumed[0] = 0;
                    consumed[1] = 0;
                }
//回调父控件方法：onNestedPreScroll
                ViewParentCompat.onNestedScroll(parent, mView,
                        dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed, type, consumed);

                if (offsetInWindow != null) {
                    mView.getLocationInWindow(offsetInWindow);
                    offsetInWindow[0] -= startX;
                    offsetInWindow[1] -= startY;
                }
                return true;
            } else if (offsetInWindow != null) {
                // No motion, no dispatch. Keep offsetInWindow up to date.
                offsetInWindow[0] = 0;
                offsetInWindow[1] = 0;
            }
```
# NestedScrollView#onTouch#ACTION_UP
```java
final VelocityTracker velocityTracker = mVelocityTracker;
                velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);
                if ((Math.abs(initialVelocity) >= mMinimumVelocity)) {
//还有剩余距离，执行fling动作
                    if (!edgeEffectFling(initialVelocity)
                            && !dispatchNestedPreFling(0, -initialVelocity)) {
                        dispatchNestedFling(0, -initialVelocity, true);
                        fling(-initialVelocity);
                    }
                } else if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0,
                        getScrollRange())) {
                    ViewCompat.postInvalidateOnAnimation(this);
                }
                mActivePointerId = INVALID_POINTER;
//回收工作


                endDrag();
```