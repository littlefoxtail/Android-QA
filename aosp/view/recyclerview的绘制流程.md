
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