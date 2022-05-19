# FlexboxLayout的实现过程

本质上是一个ViewGroup

## onMeasure

```java
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //保存每个子元素的排列顺序
        if (mOrderCache == null) {
            mOrderCache = new SparseIntArray(getChildCount());
        }
        // 如果测量子元素的顺序order发生了改变，那么就重新赋值
        if (mFlexboxHelper.isOrderChangedFromLastMeasurement(mOrderCache)) {
            mReorderedIndices = mFlexboxHelper.createReorderedIndices(mOrderCache);
        }

        // TODO: Only calculate the children views which are affected from the last measure.

        switch (mFlexDirection) {
            case FlexDirection.ROW: // Intentional fall through
            case FlexDirection.ROW_REVERSE:
                measureHorizontal(widthMeasureSpec, heightMeasureSpec);
                break;
            case FlexDirection.COLUMN: // Intentional fall through
            case FlexDirection.COLUMN_REVERSE:
                measureVertical(widthMeasureSpec, heightMeasureSpec);
                break;
            default:
                throw new IllegalStateException(
                        "Invalid value for the flex direction is set: " + mFlexDirection);
        }
    }
```

每个子元素都有自己的排列顺序值，所以需要保存好子元素的顺序值，然后根据顺序进行逐一测量。然后，通过FlexDirection来分别进行横向或者纵向的测量。

只从横向分析：

```java
/**
     * Sub method for {@link #onMeasure(int, int)}, when the main axis direction is horizontal
     * (either left to right or right to left).
     *
     * @param widthMeasureSpec  horizontal space requirements as imposed by the parent
     * @param heightMeasureSpec vertical space requirements as imposed by the parent
     * @see #onMeasure(int, int)
     * @see #setFlexDirection(int)
     * @see #setFlexWrap(int)
     * @see #setAlignItems(int)
     * @see #setAlignContent(int)
     */
    private void measureHorizontal(int widthMeasureSpec, int heightMeasureSpec) {
        mFlexLines.clear();

        mFlexLinesResult.reset();
        // 计算整个容器中需要有多少根轴线
    // 策略： 1. 如果容器的FlexWrap为NOWRAP时，返回1
    //       2. 如果FlexWrap为WRAP/WRAP_REVERSE时，遍历计算每个子元素的宽度，求和，当容器剩余的宽度不足以放置下一个
    //          子元素时（或者子元素wrapBefore为true）,换行，行数+1。
        mFlexboxHelper
                .calculateHorizontalFlexLines(mFlexLinesResult, widthMeasureSpec,
                        heightMeasureSpec);
        mFlexLines = mFlexLinesResult.mFlexLines;

        mFlexboxHelper.determineMainSize(widthMeasureSpec, heightMeasureSpec);

        // TODO: Consider the case any individual child's mAlignSelf is set to ALIGN_SELF_BASELINE
        if (mAlignItems == AlignItems.BASELINE) {
            for (FlexLine flexLine : mFlexLines) {
                // The largest height value that also take the baseline shift into account
                int largestHeightInLine = Integer.MIN_VALUE;
                for (int i = 0; i < flexLine.mItemCount; i++) {
                    int viewIndex = flexLine.mFirstIndex + i;
                    View child = getReorderedChildAt(viewIndex);
                    if (child == null || child.getVisibility() == View.GONE) {
                        continue;
                    }
                    LayoutParams lp = (LayoutParams) child.getLayoutParams();
                    if (mFlexWrap != FlexWrap.WRAP_REVERSE) {
                        int marginTop = flexLine.mMaxBaseline - child.getBaseline();
                        marginTop = Math.max(marginTop, lp.topMargin);
                        largestHeightInLine = Math.max(largestHeightInLine,
                                child.getMeasuredHeight() + marginTop + lp.bottomMargin);
                    } else {
                        int marginBottom = flexLine.mMaxBaseline - child.getMeasuredHeight() +
                                child.getBaseline();
                        marginBottom = Math.max(marginBottom, lp.bottomMargin);
                        largestHeightInLine = Math.max(largestHeightInLine,
                                child.getMeasuredHeight() + lp.topMargin + marginBottom);
                    }
                }
                flexLine.mCrossSize = largestHeightInLine;
            }
        }

        mFlexboxHelper.determineCrossSize(widthMeasureSpec, heightMeasureSpec,
                getPaddingTop() + getPaddingBottom());
        // Now cross size for each flex line is determined.
        // Expand the views if alignItems (or mAlignSelf in each child view) is set to stretch
        mFlexboxHelper.stretchViews();
        setMeasuredDimensionForFlex(mFlexDirection, widthMeasureSpec, heightMeasureSpec,
                mFlexLinesResult.mChildState);
    }
```
