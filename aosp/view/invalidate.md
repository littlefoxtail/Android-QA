# Invalidate

时序图：
![invalidate的内部实现](/img/invalidate的内部实现.png)

```java
public void invalidate() {
    invalidate(true);
}

public void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache, boolean fullInvalidate) {
    // 如果View重绘，则它也将重绘
    if (mGhostView != null) {
        mGhostView.invalidate(true);
        return;
    }
    // View是否可见，是否在动画运行中
    if (skipInvalidate()) {
        return;
    }
    // 根据View的标记来判断View是否需要进行重绘

    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
            if (fullInvalidate) {
                mLastIsOpaque = isOpaque();
                mPrivateFlags &= ~PFLAG_DRAWN;
            }
            // 设置标志，表明View正在被重绘
            mPrivateFlags |= PFLAG_DIRTY;
            // 清除缓存，设置标志
            if (invalidateCache) {
                mPrivateFlags |= PFLAG_INVALIDATE;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }
            // 把需要重绘的View区域传递给父View
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);
                // 将要刷新区域直接传递给了父ViewGroup的invalidateChild方法
                p.invalidateChild(this, damage);
            }
    }
}
```

```java
class ViewGroup {
    public final void invalidateChild {
        ViewParent parent = this;
        do {
            // 第一次循环的时候parent是ViewGroup本身，循环中止条件是parent == null
            parent = parent.invalidateChildInParent(location, dirty);

        } while (parent != null);
    }

    public ViewParent invalidateChildInParent() {
        
        if (mPrivateFlags & (PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID) != 0) {
            return mParent;
        }
    }
}
```

```java
public class ViewRootImpl {
    @Override
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
        invalidateRectOnScreen(dirty)
    }

    private void invalidateRectOnScreen(Rect dirty) {
        if (!mWillDrawSoon && (intersected || mIsAnimation)) {
            scheduleTraversals();
        }
    }

    private void draw(boolean fullRedrawNeeded) {
        // 获取mDirty，该值表示需要重绘的区域
        final Rect dirty = mDirty;

        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
            return;
        }
    }

    private boolean drawSoftware() {
        final Canvas canvas;
        try {
            final int left = dirty.left;
            final int top = dirty.top;
            final int right = dirty.right;
            final int bottom = dirty.bottom;

            // 锁定canvas区域，由dirty区域决定
            canvas = mSurface.lockCanvas(dirty);

            try {
                dirty.setEmpty();
                mIsAnimation = false;
                mView.mPrivateFlags |= View.PFLAG_DRAWN;

                try {
                    canvas.translate(-xoff, -yoff);
                    if (mTranslator != null) {
                        mTranslator.translateCanvas(canvas);
                    }
                    canvas.setScreenDensity(scalingRequired ? mNoncompatDensity : 0);
                // 正式开始绘制
                    mView.draw(canvas);

                }
            }
        }
    }
}
```
