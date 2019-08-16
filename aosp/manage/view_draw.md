
# draw流程

所有View进行自上而下绘图的调用过程：
![draw_hierarchy](/img/draw_hierarchy.png)
![Draw](/img/draw.png)

绘制流程将决定View的样子，一个View该显示什么由绘制流程完成。

## ViewRootImpl@performDraw

里面又调用了ViewRootImpl#draw方法，并传递了fullRedrawNeeded参数。
该参数由mFullRedrawNeeded成员变量获取，它的作用是判断是否重新绘制全部视图，如果是第一次绘制视图，
那么显示应该绘绘制所有的视图，如果由于某些原因，导致了视图重绘，那么就没有必要绘制所有视图。

## ViewRootImpl@draw

```java
private void draw(boolean fullRedrawNeeded) {
    //surface用来操作应用窗口的绘图表面
    Surface surface = mSurface;
    if (surface == null || )
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            //如果采用硬件渲染绘制且ThreadedRenderer可用，进入该流程
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this)
        } else {
            //如果需要硬件渲染，但ThreadedRenderer不可用
        //     则进行ThreadRenderer初始化工作（以便下次用）
            if (mAttchInfo.mThreadedRenderer != null &&
                !mAttachInfo.mThreadedRenderer.isEnabled() &&
                mAttachInfo.mThreadedRenderer.isRequested()) {
                    try {

                    } catch (OutOfResourcesException e) {
                        handleOutOfResourcesException(e);
                        return;
                    }
                }
                mFullRedrawNeeded = true;
                scheduleTraversals();
                return;
        }
        // 不用硬件渲染，或者硬件渲染不可用，则靠软件绘制
        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
            return;
        }
    }
}
```

首先获取了mDirty值，该值保存了需要重绘区域的信息，关于视图绘制。根据fullRedrawNeeded来判断是否需要
重置dirty区域，最后调用了ViewRootImpl#drawSoftware方法，并把相关参数传递进去，包括dirty区域

ViewRootImpl@drawSoftware

```java
final Canvas canvas;
try {
//获取画布
   canvas = mSurface.lockCanvas(dirty);
   canvas.setDensity(mDensity);
} catch (Surface.OutOfResourcesException e) {

}

try {
    try {
        mView.draw(canvas);
    } finally {

    }

} finally {
    try {
        surface.unlockCanvasAndPost(canvas);
    } catch(IllegalArgumentException e) {

    }
}
```

首先实例化了Canvas对象，然后锁定该canvas的区域，由dirty区域决定，接着对canvas进行一系列的属性赋值，
最后调用了mView.draw方法，前面分析过mView就是DecorView，也就是说从DecorView开始绘制。

## View的绘制流程

由于ViewGroup没有重写draw方法，因此所有的View都是调用View#draw
绘制流程的六个步骤：

1. 对View的背景进行绘制
2. 保存当前的图层信息
3. 绘制View的内容
4. 对View的子View进行绘制
5. 绘制当前视图在滑动时的边框渐变效果
6. 绘制View的装饰

View@draw

```java
public void draw(Canvas canvas) {
    int saveCount;
    // 1.
    if (!dirtyOpaque) {
       drawBackground(canvas);
    }

    final int viewFlags = mViewFlags;
    //判断View是否具有Fading Edge, xml里需要主动配置，以支持边缘渐变效果
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
//一般情况下，不支持这种效果时
    if (!verticalEdges && !horizontalEdges) {
        //     3. 绘制自身内容
        if (!dirtyOpaque) onDraw(canvas);
        // 4. 绘制child view
        dispatchDraw(canvas);
        // 6.
        onDrawForeground(canvas);
        // 7.
        drawDefaultFocusHighlight(canvas);

    }
}
```

ViewGroup@dispatchDraw

```java
@Override
protected void dispatchDraw(Canvas canvas) {
    boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
    final int childredCount = mChildredCount;
    final View[] children = mChildren;
    int flags = mGroupFlags;

    if ((flags & FLAG_RUN_ANIMATION) != 0  && canAnimate()) {

    }
    int clipSaveCount = 0;
    //如果CLIP_TO_PADDING_MASK != 1, 则说明参数canvas描述的是画布的裁剪区域，该裁剪区域不包含当前视图组的内边距
    final boolean clipToPadding = (flags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK;
    if (clipToPadding) {
        //裁剪画布
        clipSaveCount = canvas.save(Canvas.CLIP_SAVE_FLAG);
        canvas.clipRect(mScrollX, + mPaddingLeft, mScrollY + mPaddingTop, 
                        mScrollX + mRight - mLeft - mPaddingRight, 
                        mScrollY + mBottom - mTop - mPaddingBottom);
    }

    mPrivateFlags &= ~PFLAG_DRAW_ANIMATION;
    mPrivateFlags &= ~FLAG_INVALIDATE-REQUIRED;

    boolean more = false;
    final long drawingTime = getDrawingTime();

    final ArrayList<View> preorderedList = usingRenderNodeProperties 
        ? null : buildOrderedChildList();
    final boolean customOrder = preorderedList == null && isChildredDrawingOrderEnabled();
// 默认先序遍历绘制
    for (int i = 0; i < childrenCount; i++) {
       final int childIndex = getAndVerifyPreorderedIndex(childredCount, i, customOrder);
       final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
       if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
        //        内部还是调用View的draw函数
            more |= drawChild(canvas, child, drawingTime);
       }
    }
}
```

dispatchDraw用来循环绘制子View视图，它主要做了以下事情：

1. 检查是否需要显示动画，即FLAG_RUN_ANIMATION == 1，则开始显示动画，并通知动画监听者动画已经开始。
2. 如果FLAG_USE_CHILD_DRAWING_ORDER == 0，则说明子视图按照它们在children数组里顺序进行绘制否则需要调用getChildDrawingOrder来判断绘制顺序，最终调用drawChild()来完成 子视图的绘制。
3. 判断是否需要进行重绘以及通知动画监听者动画已经结束。

ViewGroup@drawChild

```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
   return child.draw(canvas, this, drawingTime);
}
```

View@draw

```java
boolean draw(Canvas canvas, ViewGroup child, long drawingTime) {
    //表示子视图child是否还在显示动画
    boolean more = false;
    //绘制子视图的ui
    if (!drawingWithDrawingCache) {
        if (drawingWithRenderNode) {
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            ((DisplayListCanvas) canvas).drawRenderNode(renderNode);
        } else {
            // Fast path for layouts with no backgrounds
            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                //SKIP_DRAW， 说明需要跳过当前子视图去绘制自己的子视图
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                dispatchDraw(canvas);
            } else {
                // 绘制自身，再绘制它的子视图
                draw(canvas);
            }
        }
    } else if (cache != null) {
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        if (layerType == LAYER_TYPE_NONE || mLayerPaint == null) {
            // no layer paint, use temporary paint to draw bitmap
            //缓冲方式绘制，需要将上一次的Bitmap对象cache绘制到canvas上
            Paint cachePaint = parent.mCachePaint;
            if (cachePaint == null) {
                cachePaint = new Paint();
                cachePaint.setDither(false);
                parent.mCachePaint = cachePaint;
            }
            cachePaint.setAlpha((int) (alpha * 255));
            canvas.drawBitmap(cache, 0.0f, 0.0f, cachePaint);
        } else {
            // use layer paint to draw the bitmap, merging the two alphas, but also restore
            int layerPaintAlpha = mLayerPaint.getAlpha();
            if (alpha < 1) {
                mLayerPaint.setAlpha((int) (alpha * layerPaintAlpha));
            }
            canvas.drawBitmap(cache, 0.0f, 0.0f, mLayerPaint);
            if (alpha < 1) {
                mLayerPaint.setAlpha(layerPaintAlpha);
            }
        }
    }
    //恢复画布的堆栈状态
    if (restoreTo >= 0) {
        canvas.restoreToCount(restoreTo);
    }


}
```
