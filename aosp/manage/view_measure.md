# measure测量流程

## 一些概念

View的位置
> View的位置：在左上角(getLeft(),getTop())，该坐标是以它的父View的左上角为坐标原点，单位是pexels

View大小
> View大小：View的大小有两对值来表示。getMeasuredWidth()/getMeasuredHeight()这组值表示了该View在它的父View里期望的大小值，在measure()方法完成后可获得。 getWidth()/getHeight()这组值表示了该View在屏幕上的实际大小，在layout()方法完成后可获得。

View内边距
> View内边距：View的内边距用padding来表示，它表示View的内容距离View边缘的距离。通过getPaddingXXX()方法获取。需要注意的是我们在自定义View的时候需要单独处理 padding，否则它不会生效，这一块的内容我们会在View自定义实践系列的文章中展开

View外边距

> View内边距：View的外边距用margin来表示，它表示View的边缘离它相邻的View的距离。

Measure过程决定了View的宽高，该过程完成后，通常都可以通过getMeasuredWith()/getMeasuredHeight()获得宽高。

[测量、布局、绘制流程](measure_layout_draw.md)

测量时候，measure()方法被父View调用，在measure()中做一些优化工作后，调用onMeasure()来进行实际的自我测量

- View：View在onMesaure()中会计算出自己的尺寸然后保存
- ViewGroup：ViewGroup在onMeasure()会调用所有子View的measure()让它们进行自我测量，并根据子View计算出的期望尺寸来计算出它们的实际尺寸和位置然后保存。同时，根据子View尺寸和位置来计算出自己的尺寸然后保存

日常开发中接触最多的不是MeasureSpec而是LayoutParams，在View测量的时候，LayoutParams会和父View的MeasureSpec相结合被换算成View的MeasureSpec，进而决定View的大小。

所有View自上而下量算的过程：
![measure_hierarchy.png](/img/measure_hierarchy.png)

如果要进行量算过程的View是ViewGroup类型，那么ViewGroup会在onMeasure方法内部遍历子View依次进行量算。

- 整个应用量算的起点是ViewRootImpl类，从它开始依次对子View进行量算，如果子View是一个ViewGroup，那么又会遍历该ViewGroup的子View依次进行量算，量算会从View树的根节点，纵向递归进行，从而实现自上而下对View进行量算，直至完成对叶子节点View的量算
- 如何对一个View进行量算呢？Android通过调用View的measure()方法对View进行量算，让该View的父控件知道该View想要多大的尺寸空间
- 具体来说，View的父控件ViewGroup会调用View的measure方法，ViewGroup会将一些宽度和高度的限制条件传递给View的measure方法
- 在View的measure方法首先从成员变量中读取以前的缓存过的量算结果，如果能找到该缓存值，那么就基本完事了，如果没有找到缓存值，那么measure方法会执行onMeasure回调方法，measure方法会将上述的宽度和高度的限制条件依次传递给onMeasure方法。onMeasure方法会完成具体的量算工作，并将量算的结果通过View的setMeasuredDimension方法保存到View的成员变量mMeasuredWidth和mMeasuredHeight中
- 量算完成之后，View的父控件就可以通过调用getMeasuredWidth、getMeausreHeight中

## MeausreSpec

在View测量的时候，LayoutParams会和父View的MeasureSpec相结合被换算成View的MeasureSpec，进而决定View的大小

- 对于顶级View（DecorView）其父MeasureSpec由窗口的尺寸和自身的LayoutParams共同确定
- 对于普通View和MeasureSpec由父MeasureSpec和自身的LayoutParams共同确定

MeasureSpec并不是指View的测量宽高，MeasureSpec作用在于：在Measure流程中，系统会将View的LayoutParams根据父容器所施加的规则转换成对应的MeasureSpec，然后在onMeasure方法中根据这个MeasureSpec来确定View的测量宽高。

- MeasureSpec数值（数值1080(二进制为: 1111011000)为例）

	| 模式名称    | 模式数值(高2位) | 实际数值(低30位)               |
	| ----------- | --------------- | ------------------------------ |
	| UPSPECIFIED | 00              | 000000000000000000001111011000 |
	| EXACTLY     | 01              | 000000000000000000001111011000 |
	| AT_MOST     | 10              | 000000000000000000001111011000 |
- View.MeasureSpec:
	
    |模式|二进制数值|描述|
    | ----------- | --------------- | ------------------------------ |
    |UPSPECIFIED|00|默认值，父控件没有给子view任何限制，子View可以设置为任意大小|
    |EXACTLY|01|表示父控件已经确切的指定了子View的大小|
    |AT_MOST|10|表示子View具体大小没有尺寸限制，但是存在上限，上限一般为父View大小|

- 上面的测量模式跟`wrap_content`、`match_parent`以及写成固定的尺寸关系

    |对应关系|描述|
    |---|---|
    |match\_parent:EXACTLY|怎么理解呢？match_parent就是要利用父View给我们提供的所有剩余空间，而父View剩余空间是确定的，也就是这个测量模式的整数里面存放的尺寸。  |
    |wrap\_content:AT_MOST|怎么理解：就是我们想要将大小设置为包裹我们的view内容，那么尺寸大小就是父View给我们作为参考的尺寸，只要不超过这个尺寸就可以啦，具体尺寸就根据我们的需求去设定| 
    |固定尺寸[如100dp]:EXACTLY|用户自己指定了尺寸大小，我们就不用再去干涉了，当然是以指定的大小为主啦|

```java
int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
 // Ask host how big it wants to be
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```

<!-- ## 关键点1：View.onMeausre()

直接继承View来自定义View时候，需要重写onMeausre()方法，并设置wrap_content时的大小。
因为View.getDefaultSize(int size, int measureSpec)如果不处理wrap_content就相当于match_parent。
如何处理？

```java
@Override
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
      super.onMeasure(widthMeasureSpec, heightMeasureSpec);
      Log.d(TAG, "widthMeasureSpec = " + widthMeasureSpec + " heightMeasureSpec = " + heightMeasureSpec);

      //指定一组默认宽高，至于具体的值是多少，这就要看你希望在wrap_cotent模式下
      //控件的大小应该设置多大了
      int mWidth = 200;
      int mHeight = 200;

      int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
      int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);

      int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
      int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);

      if (widthSpecMode == MeasureSpec.AT_MOST && heightMeasureSpec == MeasureSpec.AT_MOST) {
          setMeasuredDimension(mWidth, mHeight);
      } else if (widthSpecMode == MeasureSpec.AT_MOST) {
          setMeasuredDimension(mWidth, heightSpecSize);
      } else if (heightSpecMode == MeasureSpec.AT_MOST) {
          setMeasuredDimension(widthSpecSize, mHeight);
      }
  }
``` -->

## measure方法

由于DecodeView继承自FrameLayout，是PhoneWindow的一个内部类，而FrameLayout没有measure方法，因此调用的是父类View的measure方法

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        //首先判断当前view的layoutMode是不是特例LAYOUT_MODE_OPTICAL_BOUNDS
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
                Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }
        //根据widthMeasureSpec和heightMeasureSpec计算key值
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        //mMeasureCache是LongSparseLongArray类型的成员变量
        //其缓存着view在不同widthMeasureSpec、heightMeasureSpec下量算的结果
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
        //mOldWidthMeasureSpec和mOldHeightMeasureSpec分别表示上次对view测量的值
        //mPrivateFlags是一个Int类型的值，其记录了View的各种状态位
        //forceLayout需要强制执行layout，所以这种情况下要尝试进行量算
        //如果新传入的widthMeasureSpec/heightMeasureSpec与上次量算时的mOldWidthMeasureSpec/mOldHeightMeasureSpec不等，
        //那么也就是说该View的父ViewGroup对该View的尺寸的限制情况有变化，这种情况下要尝试进行量算

        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
         //判断当前的宽高是否老的相同，如果相同，没必要再次测量
    final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec || heightMeasureSpec != mOldHeightMeasureSpec;
    //当mode为EXACTLY时，表示当前View的宽高在第一次调用onMeasure方法已经定死了，没必要调用onMeasure方法进行测量
    final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY &&
            MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
        ...
        if ((forceLayout || needsLayout) {
                // 通过按位操作，重置View的状态mPrivateFlags，将其标记为未量算状态
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
            //在真正进行量算之前，View还想进一步确认能不能从已有缓存mMeasureCache中读取缓存过的量算结果
            //如果是强制layout导致的量算，那么将cacheIndex设置为-1，即不从缓存中读取量算结果
            //如果不是强制layout导致的量算，那么就用key作为缓存索引cacheIndex
            int cahceIndex = fourceLayout ? -1 : mMeasureCache.indexOfKey(key);
        //sIgnoreMeasureCache是一个boolean类型的成员变量，其值是在View的构造函数中计算的，而且只计算一次
        //一些老的App希望在一次layou过程中，onMeasure方法总是被调用，
        //具体来说其值是通过如下计算的: sIgnoreMeasureCache = targetSdkVersion < KITKAT;
        //也就是说如果targetSdkVersion的API版本低于KITKAT，即API level小于19，那么sIgnoreMeasureCache为true
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                // 如果运行到此处，表示没有从缓存中取到
                // onMeasure方法将会进行实际的量算工作，并把量算的结果保存到成员变量
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                //如果运行到此，表示当前的条件允许View从缓存成员变量mMeasureCache中读取量算过的结果
                long value = mMeasureCache.valueAt(cacheIndex);

            }
        ...
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;
        mMeasureCache.put(key, ((long)mMeasureWidth << 32 | (long)mMeasuredHeight & 0xffffffffL);
}
```

DecoreView是FragmeLayout子类，因此它实际上调用的是DecorView@onMeasure方法，最后会调用super.onMeasure，即FrameLayout@onMeasure方法。

onMeasure不同的ViewGroup有着不同的实现，但大体是对每个子View进行遍历，根据ViewGroup的MeasureSpec及子View的layoutParams来确定自身的测量宽高，然后根据所有子View的测量宽高信息再确定父容器的测量宽高

> 1. RelativeLayout会让子View调用2次onMeasure，LinearLayout在有weight时，也会调用子View2次onMe
> 2. ReletiveLayout的子View如果高度和RelativeLayout不同，则会引发效率问题，当子View很复杂时候，这个问题会更加严重。如果可以，尽量使用padding代替margin
> 3. 在不影响层级深度的情况下，使用LinearLayout和FrameLayout而不是RelativeLayout
> 4. 布局中不得不使用ViewGroup多重嵌套时，不要使用LinearLayout改用RelativeLayout，可以有效降低嵌套数

## DecorView@onMeasure

1. 根据mode，来计算widthMeasureSpec和heightMeasureSpec
2. 如果存在outset，并且mode不为UNSPECIFIED，那么就会考虑到outset，重新计算widthMeasureSpec和heightMeasureSpec
3. 调用super.onMeasure方法，进行真正的测量

```java
public class DecorView {
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //
    }
}
```

## FrameLayout@onMeasure

1. 调用每个child的measure方法，测量每个child的宽高；并且记录设置了match_parent属性的child
2. 调用setMeasuredDimension方法，对自身宽高进行设置
3. 对设置了match_parent属性的child进行测量

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();
    // 长或宽的SpecMode不为EXACTLY时，measureMatchParentChildren置为true
    final boolean measureMathchParentChildren = MeasureSpec.getMode(widhtMeasureSpec) != MeasureSpec.EXACTLY || 
    MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    mMatchParentChildren.clear();
    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;
    // 依次measure子View
    for (int i = 0; i < count; i++) {
        if (mMeasureAllChildren || child.getVisibility() != GONE){
        // 具体的测量函数
        // 1.调用measureChildWidthMargins测量每一个子View的大小，找到最大高度和宽度保存在maxWidth/maxHeight
            measureChildWidthMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            // 不断迭代出子View需要的最大宽度和最大高度
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            maxWidth = Math.max(maxWidth, child.getMeasureWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight, child.getMeausreHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT || lp.height == LayoutParams.MATCH_PARENT) {
                            mMatchParentChildren.add(child);
                    }
            }
        }
    }
    // 2.将上一步计算的maxWidth/maxHeight加上padding
    maxWidth += getPaddingLefWithForegound() + getPaddingRightWithForeground();
    maxHeight += getPaddingTopWithForegound() + getPaddingBottomWithForegound();

    // 3. 当前视图是否设置最小宽度和高度，如果设置有的话，并且比前面计算得到的宽度和高度还要打，就将它们当作为当前视图的宽度和高度值
    maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
    maxWidth = Math.max(maxWidth, getSuggestedMinmumWidth());

    // 4.前景图， 如果设置有的话，并且它们比前面计算得到的宽度maxWidth和高度maxHeight还要大，那么就将它们作为当前视图的宽度和高度值
    final Drawable drawable = getForeground();
    if (drawable != null) {
        maxHeight = Math.max(maxHeight, drawable.getMininumHeight());
        maxWidth = Math.max(maxWidth, drawable.getMininumWidth());
    }
    // 经过一些列步骤，得到了ViewGroup的maxHeight和maxWidth的最终值
    // 表示当容器用这个尺寸就能够正常显示器所有字View

    // 5. 此处resolveSizeAndState根据数值、MeasureSpec和childState计算出最终的数值
    // 然后用setMeasureDimension保存到mMeasureWidth与mMeasureHeight成员变量(定义于View)
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState), resolveSizeAndState(maxHeight, heightMeasureSpec, 
    childState << MEASURED_HEIGHT_STATE_SHIFT));
    count = mMatchParentChildren.size();
    if (count > 1) {
        for (int i = 0; i < count; i++) {
            // 根据父容器的参数生成新的约束条件
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```

ViewGroup#measureChildWithMargins

```java
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) {
    // 获取子View的LayoutParams
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    // 生成新的约束条件
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec, mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasure, mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin + heightUsed, lp.height);
    // 调用子View的measur
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

日常开发中接触最多的不是MeasureSpec而是LayoutParams，在View测量的时候，LayoutParams会和父View的MeasureSpec
相结合被换算成View的MeasureSpec，进而决定View的大小。

ViewGroup@getChildMeasureSpec

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);
        // 得到父view在相应方向上的可用大小
        int resultSize = 0;
        int resultMode = 0;
        int size = Math.max(0, specSize - padding);
        switch (specMode) {
            case MeasureSpec.EXACTLY:
                if (childDimension >= 0) {
                    resultSize =` childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayotParams.MATCH_PARENT) {
                    // 子View想要父view的大小
                    resultSize = size;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParms.WRAP_CONTENT) {
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;
                // 父 已经指定一个最大值
            case MeasureSpec.AT_MOST:
                if (childDimension >= 0) {
                    // 子view,希望一个固定的大小
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (child == LayoutParams.MATCH_PARENT){
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;
            case MeasureSpec.UNSPECIFIED:
            // 不关注
                break;
        }
}
```

## View的测量过程

ViewGroup提到measureChildWithMargin方法，它接收的主要参数是子View以及父容器的MeasureSpec，所以它的作用就是对子View进行测量，
ViewGroup#measureChildWithMargins:

|子View的LayoutParams\父容器SpecMode|EXACTLY|AT_MOST|UNSPECIFIED|
| ---- | ---- | ---- | ---- |
|精确值(dp)|EXACTLY childSize|EXACTLY childSize|EXACTLY childSize|
|match_parent|EXACTLY parentSize|AT\_MOST parentSize|UNSPECIFIED 0|
|wrap_content|AT_MOST parentSize|AT\_MOST parentSize|UNSPECIFIED 0|

通过获取子View的MeasureSpec获得后，执行child.measure，绘制流程已经从ViewGroup转移到子View中了，
在measure方法中会调用onMeasure方法，

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
        getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

## 测量总结

测量始于DecorView，通过不断的遍历子View的measure方法，根据ViewGroup的MeasureSpec及子View的LayoutParams来决定
子View的MeasureSpec，进一步获取子View的测量宽高，然后逐层返回，不断保存ViewGroup的测量高度。
