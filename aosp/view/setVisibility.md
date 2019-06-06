# setVisibility

View有一个方法setVisibility，作用是可以控制视图的显示和隐藏，int类型的参数可以传入三种值View.VISIBLE, View.GONE, View.INVISIBLE，其中VISIBILE表示设置视图显示，GONE和INVISIBLE设置视图隐藏，区别在于前者隐藏后不占用视图空间，而后者隐藏后依然占用视图空间

```java
public void setVisibility(int visibility) {
    setFlag(visibility, VISIBILITY_MASK);
}
```

```java
/**
 * flags要改变的值
 * mask要改变的标记位
 **/
void setFlags(int flags, int mask) {
    int old = mViewFlags;
    // &~ 清除某个值 更新视图状态为将要更改的属性
    mViewFlags = (mViewFlags & ~mask) | (flags & mask);
    // 获取其更改的属性位
    // 前后的bit位都会存储
    int change = mViewFlags ^ old;
    if(change == 0) {
        return;
    }
    // 记录当前视图的逻辑位
    int privateFlags = mPrivateFlags;

    final int newVisibility = flags & VISIBILITY_MASK;
    //如果设置标记的flags的
    if (newVisibility == VISIBLE) {
        if ((change & VISIBILITY_MASK) != 0) {
            /*
             *如果这个视图将要成为可视的，设置DRAWN的标记，以至于在下次重绘时将不跳过其绘制
             */

            // 标记mPrivateFlags添加DRAWN标记
            mPrivateFlags |= PFLAG_DRAWN;
            invalidate(true);

            needGlobalAttributesUpdate(true);
            // 通知父视图有了新的子视图。添加的新视图不管怎样都需要将其申请一次获取焦点，即使其没有获取焦点的能力
            if (mParent != null && mBottom > mTop && (mRight > mLeft)) {
                mParent.focusableViewAvaliable(this);
            }
        }
    }
    /* 检查如果是GONE属性位发生了变化*/
    if ((changed & GONE) != 0) {
        //需要全局属性更新，因为GONE属性设置其视图不见了，其他视图的位置也会受到影响。
        needGobalAttributesUpdate(false);
        // 申请重新布局
        requestLayout();

        //如果更改后的视图属性的VISIBILITY属性为GONE
        if(((mViewFlags & VISIBILITY_MASK) == GONE)) {
            //如果当前视图有焦点的话，就将其焦点清除
            if (hasFocus()) {
                clearFocus();
                if (mParent instanceOf ViewGroup) {
                    ((ViewGroup) mParent).clearFocusedInCluster();
                }
            }
            clearAccessibilityFocus();
            //释放视图所使用的缓存，当然不是所有视图都有缓存
            destoryDrawingCache();
            if (mParent instanceOf View) {
                ((View) mParent).invalidate(true);
            }
            mPrivateFlags |= PFLAG_DRAWN;
        }
        if (mAttachInfo != null) {
            mAttachInfo.mViewVisibilityChanged = true;
        }
    }

    if ((changed & INVISIBLE) != 0) {
        needGlobalAttributesUpdate(false);

        mPrivateFlags |= PFLAG_DRAWN;

        if (((mViewFlags & VISIBILITY_MASK) == INVISIBLE)) {
            // 当前视图不是跟视图
            if (getRootView() != this) {
                if (hasFocus()) {
                    //清除焦点
                    clearFocus();
                    if (mParent instanceOf ViewGroup) {
                        ((ViewGroup) mParent).clearFocusedInCluster();
                    }
                }
                clearAccessibilityFocus();

            }
        }
        if (mAttachInfo != null) {
            //更改视图的标记对象的为true
            mAttachInfo.mViewVisibilityChanged = true;
        }
    }

    if ((change & VISIBILITY_MASK) != 0) {
        // 如果视图不可见，请清理器显示列表以释放资源
        if (newVisibility != VISIBLE && mAttachInfo != null) {
            cleanupDraw();
        }
        if (mParent instanceof ViewGroup) {
            ((ViewGroup) mParent).onChildVisibilityChanged(this, (changed & VISIBILITY_MASK), newVisibility);
        } else if (mParent != null) {
            mParent.invalidateChild(this, null);
        }

        if (mAttachInfo != null) {
            dispatchVisibilityChanged(this, newVisilibity);


            if (mParent != null && getWindowVisibility() == VISIBLE
                && ((!(mParent instanceof ViewGroup)) || ((ViewGroup)mParent).isShown())) {
                dispatchVisibilityAggregated(newVisibility == VISIBLE);
            }
            notifySubtreeAccessibilityStateChangedIfNeeded();
        }
    }
    //对视图缓存以及绘制标记的处理
    if ((change & WILL_NOT_CACHE_DRAWING) != 0) {
        destroyDrawingCache();
    }

    if ((change & DRAWING_CACHE_ENABLED) != 0) {
        destroyDrawingCache();
        mPrivateFlags &= ~DRAWING_CACHE_VALID;
        invalidateParentCaches();
    }

    if ((change & DRAWING_CACHE_QUALITY_MASK) != 0) {
        destroyDrawingCache();
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
    }
}
```

