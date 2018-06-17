# Invalidate

```java
public void invalidate() {
    invalidate(true);
}

public void validateInternal(int l, int t, int r, int b, boolean invalidateCache, boolean fullInvalidate) {
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
            if ()


    }
}
```