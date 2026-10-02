# layout流程

1. 如果`host`不为null，也就是`DecorVeiw`不为null，调用`DecorView`的`layout`方法，将布局操作分发下去。
2. 如果`mLayoutRequesters`不为空的话，进行第二次布局。至于`mLayoutREquesters`什么不为空，这就涉及到`requestLayout`方法了

```java
public class ViewRootImpl {
    private void performLayout(..) {
        mLayoutRequested = false;
        final View host = mView;
        if (host == null) {
            return;
        }

    }
}
```

所有View自上而下布局的调用过程：
![layout_hierarchy](/img/layout_hierarchy.png)

在进行布局的时候，layout()方法被父View调用，在layout()中它会保存父View传进来的自己的位置和尺寸，并且调用onLayout来进行实际的内部布局。对于onLayout()，View和ViewGroup有所区别

- View：由于没有子View，所以View的onLayout()什么也不做
- ViewGroup：ViewGroup在onLayout()中会调用自己的所有子View的layout()方法，

## ViewGroup的布局流程

performLayout方法进行layout流程
通过对mLeft、mTop、mRight、mBottom这四个值进行了初始化，对于每一个View，包括ViewGroup来说，以上四个值保存了View的位置信息，所以这四个值是最终宽高。也即是说，如果要得到View的位置信息，那么就应该在layout方法完成后调用getLeft(),getTop()等方法来取得最终宽高，如果是在此之前调用相应的方法，只能得到0的结果，所以一般我们是在onLayout方法中获取View的宽高信息。

首先先获取父容器的padding值，然后遍历其每一个子View，根据子View的layout_gravity属性、子View的测量宽高、父容器的padding值、来确定子View的布局参数，然后调用child.layout方法，把布局流程从父容器传递到子元素

### View#layout()

```java
public void layout(int l, int t, int r, int b) {
    //成员变量mPrivateFlags3中的一些比特位存储和laout相关的信息
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        //如果在mPrivateFlag3的低位字节的第4位（从最右4向左第4位）的值为1，
        //那么久表示在layout布局的需要对View进行测量
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        //量完了需要移除标签
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    //1.设置四个顶点
    //这里setOpticalFrame还是会调用setFrame
    //setFrame方法会将新的left,top,right,bottom存储到view的成员变量
    //返回true代表发生了view的位置和尺寸变化
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        // 2. 确定View子元素的位置
        //如果view的布局发生了变化，或者mPrivateFlags有需要LAYOUT的标签PFLAG_LAYOUT_REQUIRED,那么就会执行
        //首先触发onLayout方法执行，View中默认的onLayout是个空方法
        //不过继承自ViewGroup的类都要实现onLayout，从而在onLayout中依次循环子View
        //并调用view#layout
        onLayout(changed, l, t, r, b);

        if (shouldDrawRoundScrollbar()) {
            if(mRoundScrollbarRenderer == null) {
                mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
            }
        } else {
            mRoundScrollbarRenderer = null;
        }
        //移除flag
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
        //可以通过View的addOnLayoutChangeListener(View.onLayoutChangeListener)方法
        //这些事件都存储在ListenerInfo.mOnLayoutChangeListeners
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
                // 对mOnlayoutChangeListeners中的事件监听器进行拷贝
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                    // 遍历注册器，依次调用onLayoutChange方法，这样Layout事件监听器就得到了响应
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }
    // 移除强制layout标签
    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    // 加入layout完成标签
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

    if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
        mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
        notifyEnterOrExitForAutoFillIfNeeded(true);
    }
}

```

- 在layout()方法内部刚开始执行的时候，首先会根据mPrivateFlag3变量是否具有标志位`PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT`，是则执行
onMeasure()方法，从而对View进行量算，量算的结果会保存在View的成员变量中。量算完成后就会将mPrivateFlag3移除标签`PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT`
- 如果isLayoutModeOptical()返回true，那么就会执行setOpticalFrame()方法，否则会执行setFrame()方法，并且setOpticalFrame()内部会调用setFrame()，所以无论如何都会执行setFrame()方法。setFrame()方法会将View新的left、top、right、bottom存储到View的成员变量中，并且返回一个boolean值，如果返回true表示View的位置或尺寸发生了变化，否则未发生变化。
- 如果View的布局发生了变化，或者mPrivateFlags有需要LAYOUT的标签PFLAG\_LAYOUT\_REQUIRED，就会触发onLayout方法的执行，View中默认的onLayout方法是个空方法。不过继承自ViewGroup的类都需要实现onLayout方法，从而onLayout方法中依次循环子View，并调用子View的layout方法。在执行完onLayout方法之后，从mPrivateFlags中移除标签PFLAG_LAYOUT_REQUIRED。然后会遍历注册的Layout Change事件监听器，依次调用onLayoutChange方法，这样Layout事件监听器就会得到响应
- 从mPrivateFlags中移除强制layout的标签PFLAG\_FORCE_LAYOUT，向mPrivateFlags3中加入Layout完成的标签PFLAG3\_IS_LAID_OUT

### setFrame

```java
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;

    if (DBG) {
        Log.d("View", this + " View.setFrame(" + left + "," + top + ","
                + right + "," + bottom + ")");
    }

    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        //将新旧left、right、top、bottom进行对比，只要不完全相对就说明View的局部发生了变化，则将changed变量设置为true 
        changed = true;

        // Remember our drawn bit
        // 先保存一下mPrivateFlags中的PFLAG_DRAW标签信息
        int drawn = mPrivateFlags & PFLAG_DRAWN;
        // 计算View的新旧尺寸
        int oldWidth = mRight - mLeft;
        int oldHeight = mBottom - mTop;
        int newWidth = right - left;
        int newHeight = bottom - top;
        // 比较View的新旧尺寸是否相同，如果尺寸发生了变化，那么sizeChanged的值为true
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

        // Invalidate our old position
        invalidate(sizeChanged);
        //将新的left、top、right、bottom存储到View的成员变量中 
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        //mRenderNode.setLeftTopRightBottom()方法会调用RenderNode中原生方法的nSetLeftTopRightBottom()方法，
        //该方法会根据left、top、right、bottom更新用于渲染的显示列表
        mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
        //向mPrivateFlags中增加标签PFLAG_HAS_BOUNDS,表示当前View具有了明确的边界范围
        mPrivateFlags |= PFLAG_HAS_BOUNDS;


        if (sizeChanged) {
            //如果View的尺寸和之前发生了变化，那么就执行sizeChange()方法
            //该方法中又会调用onSizeChanged方法，并将View的新旧尺寸传递进去
            sizeChange(newWidth, newHeight, oldWidth, oldHeight);
        }

        if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {
            // If we are visible, force the DRAWN bit to on so that
            // this invalidate will go through (at least to our parent).
            // This is because someone may have invalidated this view
            // before this call to setFrame came in, thereby clearing
            // the DRAWN bit.
            // 有可能在调用setFrame方法之前，invalidate方法就被调用了
            // 这会导致mPrivateFlag移除了PFLAG_DRAWN标签
            // 如果当前View处于可见状态就将mPrivateFlags强制添加PFLAG_DRAW状态位，
            // 这样会确保下面的invalidate()方法会执行到其父控件级别
            mPrivateFlags |= PFLAG_DRAWN;
            invalidate(sizeChanged);
            // parent display list may need to be recreated based on a change in the bounds
            // of any child
            // 父控件会重建用渲染的显示列表
            invalidateParentCaches();
        }

        // Reset drawn bit to original value (invalidate turns it off)
        // 重新恢复mPrivateFlags中原有的PFLAG_DRAWN标签信息
        mPrivateFlags |= drawn;

        mBackgroundSizeChanged = true;
        mDefaultFocusHighlightSizeChanged = true;
        if (mForegroundInfo != null) {
            mForegroundInfo.mBoundsChanged = true;
        }

        notifySubtreeAccessibilityStateChangedIfNeeded();
    }
    return changed;
}
```

- 该方法中，会将旧left、right、top、bottom进行对比，只要不完全相同就说明View的布局发生了变化，则将changed变量设置为true。然后比较新旧尺寸是否相同，如果尺寸发生了变化，并将其保存到变量sizeChanged中，如果尺寸发生了变化，那么sizeChanged的值为true

- 将新的left、top、right、bottom存储到View的成员变量中保存下来。并执行mRenderNode.setLeftTopRightBottom()方法会，其会调用RenderNode中原生方法的nSetLeftTopRightBottom()方法，该方法会根据left、top、right、bottom更新用于渲染的显示列表

- 如果View的尺寸和之前相比发生了变化，那么就执行sizeChange()方法，该方法中又会调用onSizeChanged()方法，并将View的新旧尺寸传递进去

- 如果View处于可见状态，那么会调用invalidate和invalidateParentCaches方法。invalidateParentCaches()方法会移除其父控件的PFLAG_INVALIDATED标签，这样其父控件就会重建用于渲染的显示列表

### sizeChange

sizeChange方法会在View的尺寸发生变化时调用，在setFrame()方法中就可能会调用sizeChange()方法。在View的setLeft/setTop/setRight/setBottom等其他改变View尺寸的方法也会调用

```java
private void sizeChange(int newWidth, int newHeight, int oldWidth, int oldHeight) {
    //将View的新旧尺寸传递给onSizeChange()方法
    if (mOverlay != null) {
        mOverlay.getOverlayView().setRight(newWidth);
        mOverlay.getOverlayView().setBottom(newHeight);
    }
    rebuidOutline();
}

```

#### onSizeChanged

onSizeChange()方法是个空方法时，通过sizeChange()方法的执行而被调用。当View第一次加到View树中，该方法也会被调用。只不过传入的旧尺寸OldWidth和oldHeight都是

View的onLayou为一个空实现

### FrameLayout@onLayout

```java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    layoutChildren(left, top, right, bottom, false);
}
```

```java
void layoutChildren(int left, int top, int right, int bottom, booean forceLeftGravity) {
    final int count= getChildCount();
// parentLeft表示当前View为其子View显示区域指定的一个左边界
// 也就是子View显示区域的左边缘到父View的左边缘的距离
    final int parentLeft = getPaddingLeftWithForeground();
    final int parentRight = right - left - getPaddingRightWithForeground();
    final int parentTop = getPaddingTopWithForeground();
    final int parentBottom = bottom - top - getPaddingBottomWithForeground();

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final int width = child.getMeasuredWidth();
            final int height = child.getMeasureHeight();

            int childLeft;
            int childTop;

            int gravity = lp.gravity;
            if (gravity == -1) {
                gravity = DEFAULT_CHILD_GRAVITY;
            }
            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = GRAVITY.GetAbsoluteGravity(gravity, layoutDirection);
                ...
            child.layout(childLeft, childTop, childLeft + width, childTop + height);
        }
    }
}
```

## 子View的布局流程

子View的布局流程也很简单。如果子View是一个ViewGroup,那么会重复以上步骤，如果是一个View，那么会直接调用  
View#layout方法，根据以上分析，在该方法内部会设置view的四个布局参数，接着调用onLayout方法，
View#onLayout方法是一个空实现，主要作用是在我们自定义View中重写该方法，实现自定义的布局逻辑。

## 定位总结

View的布局流程就已经全部分析完了。可以看出，布局流程的逻辑相比测量流程来说，简单许多，  
获取一个View的测量宽高是比较复杂的，而布局流程则是根据已经获得的测量宽高进而确定一个View的四个位置参数
