# invalidate

该方法调用会引起View树的重绘，常用于内部调用或者需要刷新界面的时候，需要在主线程中调用该方法。当子View调用了invalidate方法后，会为该View添加一个标记位，同时不断向父容器请求刷新，父容器通过计算得出自身需要重绘的区域，直到传递到ViewRootImpl中，最终触发performTraversals方法，进行开始View树重绘流程。

时序图：
![invalidate的内部实现](/img/invalidate的内部实现.png)

```java
public void invalidate() {
        invalidate(true);
}
```

```java
public void invalidate(boolean invalidateCache) {
        invalidateInternal(0,  0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}
```

```java
public void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache, boolean fullInvalidate) {
        ...
        if (skipInvalidate()) {
                return;
        }
        ////根据View的标记位来判断该子View是否需要重绘，假如View没有任何变化，那么就不需要重绘
        if ((mPrivateFlags & (PFLAG_DRAW | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAW | PFLAG_HAS_BOUNDS) || ...) {
                // 设置PFLAG_DIRTY
                mPrivateFlags |= PFLAG_DIRTY;

                final AttachInfo ai = mAttachInfo;
                final ViewParent p = mParent;
                if(p != null && ai != null && l < r && t < b) {
                        final Rect damage = ai.mTmpInvalRect;
                        damage.set(l, t, r, b);
                        // 调用父容器的方法，向上传递事件
                        p.invalidateChild(this, damage);
                }
        }

}
```

ViewGroup@invalidateChild

该方法内部，先设置当前视图的标记位，接着一个do...while循环，该循环的作用主要是不断向上回溯父容器，求得父容器和子View需要重绘的区域的并集(dirty)。当父容器不是ViewRootImpl，调用的是ViewGroup的invalidateChildInParent

```java
public final void invalidateChild(View child, final Rect dirty) {
        ViewParent parent = this;
        if (attachInfo != null) {
                final boolean drawAnimation = (child.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0;
                Matrix childMatrix = child.getMatrix();
                final boolean isOpaque = child.isOpaque() && !drawAnimation && child.getAnimation() == null && childMatrix.isIdentity();
                int opaqueFlag = isOpaque ? PFLAG_DIRTY_OPAQUE : PFLAG_DIRTY;

                if (child.mLayerType != LAYER_TYPE_NONE) {
                    mPrivateFlags |= PFLAG_INVALIDATED;
                    mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
                }
                //存储子View的mLeft和mTop值
                final int [] location = attachInfo.mInvalidateChildLocation;
                location[CHILD_LEFT_INDEX] = child.mLeft;
                location[CHILD_TOP_INDEX] = child.mTop;
                ...
                do {
                    View view = null;
                    if (parent instanceof View) {
                        view = (View) parent;
                    }
    
                    if (drawAnimation) {
                        if (view != null) {
                            view.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
                        } else if (parent instanceof ViewRootImpl) {
                            ((ViewRootImpl) parent).mIsAnimation = true;
                        }
                    }

                    if (view != null) {
                        if ((view.mViewFlags & FADING_EDGE_MASK) != 0 && view.getSolidColor() == 0) {
                            opaqueFlag = PFLAG_DIRTY;
                        }
                        if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY)  {
                        //对当前View的标记位进行设置
                            view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
                        }
                    }

                    parent = parent.invalidateChildInParent(location, dirty);
                    if (view != null) {
                        Matrix m = view.getMatrix;
                        if (!m.isIndentity()) {
                            RectF boundingRect  = attachInfo.mTmpTransformRect;
                            boundingRect.set(dirty);
                            m.mapRect(boundingRect);
                            dirty.set((int) Math.floor(boundngRect.left), 
                            (int) Math.floor(boundingRect.top), 
                            (int) Math.floor(boundingRect.right),
                            (int) Math.floor(boundingRect.bottom));
                        }
                    }
                } while (parent != null);
        }
}
```

ViewGroup@invalidateChildInParent
主要工作：调用offset方法，把当前dirty区域的坐标转化为父容器中的坐标，接着调用union方法，把子dirty区域与父容器的区域求并集，换句话说，dirty区域变成父容器区域。最后返回当前视图的父容器，以便下一次循环。

```java
public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
    if ((mPrivateFlags & (PFLAG_DRAW | PFLAG_DRAWING_CACHE_VALID)) != 0) {
        if ((mGroupFlags & (Flag_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE)) != FLAG_OPTIMIZE_INVALIDATE) {
                // 将dirty中的坐标转化为父容器中的坐标，考虑mScrollX和mScrollY的影响
            dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                        location[CHILD_TOP_INDEX] - mScrollY);
            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == 0) {
                    //求并集，结果是把子视图的dirty区域转化为父容器的dirty区域
                dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
            }

            final int left = mLeft;
            final int top =  mTop;

            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                if (!dirty.intersect(0, 0, mRight - left, mBottom - top)) {
                    dirty.setEmpty();
                }
            }
            //记录当前视图的mLeft和mTop值，在下一次循环中会把当前值再向父容器的坐标转化
            location[CHILD_LEFT_INDEX] = left;
            location[CHILD_TOP_INDEX] = top;
        } else {
            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == PFLAG_CLIP_CHILDREN) {
                dirty.set(0， 0， mRight - mLeft, mBottom - mTop);
            } else {
                dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
            }
            location[CHILD_LEFT_INDEX] = mLeft;
            location[CHILD_TOP_INDEX] = mTop;
            mPrivateFlags &= ~PFLAG_DRAWN;
        }
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        if (mLayerType != LAYER_TYPE_NONE) {
            mPrivateFlags |= PFLAG_INVALIDATED;
        }
        return mParent;
    }
    return null;
}
```

ViewRootImpl@invalidateChildInParent

```java
public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
    checkThread();

    if (dirty == null) {
        invalidate();
        return null;
    } else if (dirty.isEmpty() && !mIsAnimation) {
        return null;
    }

    if (mCurScrollY != 0 || mTranslator != null) {
        mTempRect.set(dirty);
        dirty = mTempRect;
        if (mCurScrolly != 0) {
            dirty.offset(0, -mCurScrollY);
        }
        if (mTranslator != null) {
            mTranslator.translateRectInAppWindowToScreen(dirty);
        }
        if (mAttachInfo.mScalingRequired) {
            dirty.inset(-1, -1)
        }
    }
    invalidateRectOnScreen(dirty);
    return null;
}
```

都进行了offset和union对坐标的调整，然后把dirty区域的信息保存在mDirty中，
最后调用scheduleTraversal方法，触发View的工作流程，由于没有添加measure和layout的标记位，因此measure、layout流程不会执行，而是直接从draw流程开始。

```java
private void invalidateRectOnScreen(Rect dirty) {
   final Rect localDirty = mDirty;
   if (!localDirty.isEmpty() && !localDirty.contains(dir)) {
        mAttachInfo.mSetIgnoreDirtyState = true;
        mAttachInfo.mIgnoreDirtyState = true;
   }
//    求并集
   localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);
   final float appScale = mAttachInfo.mApplicationScale;
//    求交集
   final boolean intersected = localDirty.intersect(0, 0, (int)(mWidth * appScale + 0.5f), (int)(mHeight * appScale + 0.5f));

   if (!intersected) {
        localDirty.setEmpty();
   }
   if(!mWillDrawSoon && (intersected || mIsAnimating)) {
       scheduleTraversals();
   }
}
```
