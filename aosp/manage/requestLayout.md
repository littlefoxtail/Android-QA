# rquestLayout

## 执行流程

![大体执行流程](/img/request_invalidate.webp)

当View发生改变使得这个view的布局无效的时候，调用该方法，如果View正在请求布局的时候，View树正在进行布局，那么requestLayout会等到布局流程完成之后，或者绘制流程完成且下一次布局出现的时候执行
当我们动态移动一个View的位置或者View的大小、形状发生了变换的时候，我们可以在View中调用这个方法。

子View调用requestLayout方法，会标记当前View及父容器，同时逐层向上提交，直到ViewRootImpl处理该事件，ViewRootImpl会调用三大流程，从measure开始，对于每一个含有标记位的view及其子View都会进行测量、布局、绘制。

```java
public void requestLayout() {
  if (mMeasureCache != null) mMeasureCache.clear();
  if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
          ViewRootImpl viewRoot = getViewRootImpl();
          if (viewRoot != null && viewRoot.isInLayout()) {
                //   判断View树是否正在布局流程
                  if (!viewRoot.requestLayoutDuringLayout(this)) {
                          return;
                  }
          }
          mAttachInfo.mViewRequestingLayout = this;
  }
//   为当前view设置标记为 PFLAG_FORCE_LAYOUT，该标记位的作用就是标记了当前的View是需要进行重新布局的，接着调用mParent.requestLayout方法，为父容器添加PFLAG_FORCE_LAYOUT标记位，而父容器又会调用它的父容器的requestLayout方法，即requestLayout事件层层向上传递，直到DecorView，即根View，而根View又会传递给ViewRootImpl
// 见view.assignParent(this);)
// 也即是说子View的requestLayout事件，最终会被ViewRootImpl接收并得到处理。
  mPrivateFlags |= PFLAG_FORCE_LAYOUT;
  mPrivateFlags |= PFLAG_INVALIDATED;

  if (mParent != null && !mParent.isLayoutRequested()) {
        //   向父容器请求布局
          mParent.requestLayout();
  }
  if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
          mAttachInfo.mViewRequestingLayout = null
  }
}
```

ViewRootImpl@requestLayout

```java
public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
                checkThread();
                mLayoutRequested = true;
                scheduleTraversals();
        }
}
```

scheduleTraversals最终会调用performTraversals方法，

首先是判断一下标记位，如果当前View的标记位为PFLAG_FORCE_LAYOUT，那么就会进行测量流程，调用onMeasure，对该View进行测量，接着最后为标记位设置为PFLAG_LAYOUT_REQUIRED,这个标记位的作用就是在View的layout流程中，如果当前View设置了该标记位，则会进行布局流程

## 总结

详细执行时序图：

![requestLayout](/img/requestlayout.png)
