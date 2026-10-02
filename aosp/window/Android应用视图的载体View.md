# Android显示框架：Android应用视图的载体View

1. 整个View树结构式一个双向指针结构。View的三大流程的分发、事件分发机制都是基于责任链模式，而责任链模式分为先向上传递和向下传递，所以拥有`child`和`parent`两个指针
2. DecorView的Parent是`ViewRootImpl`，

## View的生命周期

```java
public class View {
    /**
     * View在xml文件里加载完成时调用
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

## View的测量、布局、绘制流程

[View的测量、布局、绘制前情概要](measure_layout_draw.md)
[测量流程](view_measure.md)
[定位流程](view_layout.md)
[绘制流程](view_draw.md)

## invalidate/postInvalidate/requestLayout

invalidate只会调onDraw方法且必须在UI线程中调用
postInvalidate只会调onDraw方法，可以再UI线程中回调
requestLayout会调onMeasure、onLayout和onDraw(特定条件下)方法

[requestLayout](requestLayout.md)
[invalidate](invalidate.md)
[postInvalidate](postInvalidate.md)
