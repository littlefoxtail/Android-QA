<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

* [Android显示框架：Android应用视图的载体View](#android显示框架android应用视图的载体view)
	* [View的生命周期](#view的生命周期)
	* [View的次梁流程](#view的次梁流程)

<!-- /code_chunk_output -->

![img](/img/view_measure_layout_draw.png)

# Android显示框架：Android应用视图的载体View




## View的生命周期

```java
public class View {
    /**
     *View在xml文件里加载完成时调用
     */
    protected void onFinishInflate() {

    }

    /**
     * 测量View及其子View大小调用
     */
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    }

    /**
     * 布局View及其子View大小时候调用
     */
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }

    /**
     * View大小发生改变时调用
     */
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
    }

    /**
     * 绘制View及其子View大小时调用
     */
    protected void onDraw(Canvas canvas) {

    }

    /**
     * 物理按键事件发生时候调用
     *
     */
    public boolean onKeyDown(int keyCode, KeyEvent event) {

    }

    /**
     * 触摸事件发生时候调用
     */
    public boolean onTouchEvent(MotionEvent event) {

    }

    /**
     * View获取焦点或者失去焦点时调用
     */
    protected void onFocusChanged(boolean gainFocus, int direction, Rect previouslyFocusedRect) {

    }

    /**
     * View所在窗口获取焦点或者失去焦点时调用
     */
    public void onWindowFocusChanged(boolean hasWindowFocus) {

    }

    /**
     * View被关联到窗口时调用
     */
    protected void onAttachedToWindow() {

    }

    /**
     * View从窗口分离时调用
     */
    protected void onDetachedFromWindow(){

    }

     /**
     * View的可见性发生变化时调用
     */
    protected void onVisibilityChanged(View changeView, int visibility){

    }

    /**
     * View所在窗口的可见性发生变化时调用
     */
    protected void onWindowVisibilityChanged(int visibility) {

    }
}
```

View的声明随着Activity生命周期变化的情况：

![view_lifecycle_complete](/img/view_lifecycle_complete.png)

## View的次梁流程

View的位置
> View的位置：在左上角(getLeft(),getTop())，该坐标是以它的父View的左上角为坐标原点，单位是pexels

View大小
> View大小：View的大小有两对值来表示。getMeasuredWidth()/getMeasuredHeight()这组值表示了该View在它的父View里期望的大小值，在measure()方法完成后可获得。 getWidth()/getHeight()这组值表示了该View在屏幕上的实际大小，在layout()方法完成后可获得。

View内边距
> View内边距：View的内边距用padding来表示，它表示View的内容距离View边缘的距离。通过getPaddingXXX()方法获取。需要注意的是我们在自定义View的时候需要单独处理 padding，否则它不会生效，这一块的内容我们会在View自定义实践系列的文章中展开

View外边距

> View内边距：View的外边距用margin来表示，它表示View的边缘离它相邻的View的距离。
Measure过程决定了View的宽高，该过程完成后，通常都可以通过getMeasuredWith()/getMeasuredHeight()获得宽高。


```sequence
View->View:1.measure
View->FrameLayout:2.onMeasure
FrameLayout->ViewGroup:3.measureChildWithMargins
ViewGroup->ViewGroup:4.getChildMeasureSpec
```
测量时候，measure()方法被父View调用，在measure()中做一些优化工作后，调用onMeasure()来进行实际的自我测量

- View：View在onMesaure()中会计算出自己的尺寸然后保存
- ViewGroup：ViewGroup在onMeasure()会调用所有子View的measure()让它们进行自我测量，并根据子View计算出的期望尺寸来计算出它们的实际尺寸和位置然后保存。同时，根据子View尺寸和位置来计算出自己的尺寸然后保存

日常开发中接触最多的不是MeasureSpec而是LayoutParams，在View测量的时候，LayoutParams会和父View的MeasureSpec相结合被换算成View的MeasureSpec，进而决定View的大小

```java
public abstrct class ViewGroup extends View implments ViewParent, ViewManager {
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);
        int size = Math.max(0,specSize - padding);

        int resultSize = 0;
        int resuleMode = 0;

        switch(specMode) {
            case MeasureSpec.EXACTLY:
                if (childDimension >= 0) {
                    resuleSize = childDimension;
                    resuleMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    resuleSize = size;
                    resuleMode = MeasureSpec.EXACTLY;
                } else if (childDimensin == LayoutParams.WRAP_CONTENT) {
                    resultSize = size;
                    resuleMode = MeasureSpec.AT_MOST;
                }
                break;

            case MeasureSpec.AT_MOST:
                if (childDimension >= 0) {
                    resuleSize = childDimension;
                    resuleMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
            }
        }
    }

}
```

- 对于顶级View(DecorView)，其MeasureSpec由窗口的尺寸和自身的LayoutParams共同确定的
- 对于普通View其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams共同确定
