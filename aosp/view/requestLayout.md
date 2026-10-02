# rquestLayout

1. 整个View树结构是一个双向指针结构。其实这个不难理解，因为View的三大流程的分发、事件分发机制都是基于责任链模式，而责任链模式分为先上传递和向下传递，所以拥有child和parent两个指针并不奇怪。
2. 我们知道获取一个View的Parent得到的是一个ViewParent对象，可能大家都有一个疑问为什么不是ViewGroup呢？因为View只会在一个ViewGroup啊。从这里，我们可以得到答案，DecorView的Parent并不是View，更不是ViewGroup，所以一个View的Parent不一定是ViewGroup。

## 执行流程

`PFLAG_FORCE_LAYOUT`：会导致重新测量和布局
`PFLAG_INVALIDATED`：会导致重新绘制

![大体执行流程](/img/request_invalidate.webp)

当View发生改变使得这个view的布局无效的时候，调用该方法，如果View正在请求布局的时候，View树正在进行布局，那么requestLayout会等到布局流程完成之后，或者绘制流程完成且下一次布局出现的时候执行
当我们动态移动一个View的位置或者View的大小、形状发生了变换的时候，我们可以在View中调用这个方法。

子View调用requestLayout方法，会标记当前View及父容器，同时逐层向上提交，直到ViewRootImpl处理该事件，ViewRootImpl会调用三大流程，从measure开始，对于每一个含有标记位的view及其子View都会进行测量、布局、绘制。

```java
public void requestLayout() {
  if (mMeasureCache != null) mMeasureCache.clear();
  if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
          ViewRootImpl viewRoot = getViewRootImpl();
        //   1. 如果此时，View结构还处于layout阶段之前，直接调用ViewRootImpl的requestLayoutDuringLayout
          if (viewRoot != null && viewRoot.isInLayout()) {
                //   判断View树是否正在布局流程，将当前view对象传递进行
                  if (!viewRoot.requestLayoutDuringLayout(this)) {
                          return;
                  }
          }
          mAttachInfo.mViewRequestingLayout = this;
  }
//   2. 为当前view设置标记为 PFLAG_FORCE_LAYOUT，该标记位的作用就是标记了当前的View是需要进行重新布局的，接着调用mParent.requestLayout方法，为父容器添加PFLAG_FORCE_LAYOUT标记位，而父容器又会调用它的父容器的requestLayout方法，即requestLayout事件层层向上传递，直到DecorView，即根View，而根View又会传递给ViewRootImpl
// 见view.assignParent(this);)
// 也即是说子View的requestLayout事件，最终会被ViewRootImpl接收并得到处理。
  mPrivateFlags |= PFLAG_FORCE_LAYOUT;
  mPrivateFlags |= PFLAG_INVALIDATED;

  if (mParent != null && !mParent.isLayoutRequested()) {
        //   3. 向父容器请求布局
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
        //检查线程
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
  }
}
```

scheduleTraversals最终会调用performTraversals方法，

首先是判断一下标记位，如果当前View的标记位为PFLAG_FORCE_LAYOUT，那么就会进行测量流程，调用onMeasure，对该View进行测量，接着最后为标记位设置为PFLAG_LAYOUT_REQUIRED,这个标记位的作用就是在View的layout流程中，如果当前View设置了该标记位，则会进行布局流程

### requestLayout的责任链

`requestLayout`的责任链就是从调用`Parent`的`requestLayout`方法开始的，一层一层递归上去，最终调用到`ViewRootImpl`的`requestLayout`方法里面

### 从requestLayout角度来看三大流程

`PFLAG_LAYOUT_REQUIRED`：layout方法就会执行onLayout进行布局流程。

```java
public class View {
    public void measure(..) {
        //判断当前是否是强制布局
        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

        if (forceLayout || needsLayout) {
                // 强制布局会设置一个PFLAG_LAYOUT_REQUIRED的标识
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }
    }

    public void layout(..) {
        //强制布局
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(..);
        }
    }
}
```

### 在ViewGroup的onLayout方法调用requestLayout方法不会导致死递归

```java
public class View {
    public void requestLayout() {
        if (viewRoot != null && viewRoot.isInLayout()) {
            //在layout中的话，不会立即执行三大流程，而是将这个任务放在ViewRootImpl的一个任务队列里面
            if (!ViewRoot.requestLayoutDuringLayout(this)) {
                return;
            }
        }
    }
}


public class ViewRootImpl {
    boolean requestLayoutDuringLayout() {
        if (!mLayoutRequesters.contains(view)) {
            mLayoutRequesters.add(view);
        }
    }

    private void performLayout(..) {
        try {
            host.layout(..);

            int numViewsRequestingLayout = mLayoutRequesters.size();
            if (numViewsRequestingLayout > 0) {
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, false);
                if (validLayoutRequesters != null) {
                    // Process fresh layout requests, then measure and layout
                    int numValidRequests = validLayoutRequesters.size();
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        view.requestLayout();
                    }
                }
            }
        } finally {
            //
        }
    }

    public void layout(..) {
        boolean changed = isLayoutModeOptical(mParent) ? setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
        }
    }
}
```

## 总结

详细执行时序图：

![requestLayout](/img/requestlayout.png)
