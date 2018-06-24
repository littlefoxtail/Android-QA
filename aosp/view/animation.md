
# Android中的动画

动画资源分为两类

- 属性动画

使用`Animator`在设定的一段时间修改对象的属性值来创建动画

- View动画
  - Tween(补间)动画
    通过使用`Animation`在单个图像上执行一系列转换来创建动画
  - Frame(帧)动画
    通过使用`AnimationDrawable`顺序显示一系列图像来创建动画

## 属性动画

在XML中定义的动画，可在一段时间内修改目标对象的属性值，如背景颜色或alpha值
![image](../../img/objectAnimator.png)


属性动画功能非常强大，也是最常用的动画方法。可自定义如下属性：

- 动画时间(Duration)，指定动画总共完成所需要的时间，默认为300ms
- 时间插值器(Time interpolation)，是一个基于当前动画已消耗时间的函数，用来计算属性的值
- 重复次数(Repeat count)：指定动画是否重复执行，重复执行的次数，也可以指定动画向反方向地回退操作
- 动画集(Animator sets)，将一系列动画放进一个组，可以设置

### ObjectAnimator.ofFloat，是一个静态方法：

- 首先创建ObjectAnimator对象，并指定target对象和属性名
- 然后`setFloatValues(values)`方法，经几次调用，最后调用
    `KeyframeSet.ofFloat(values)`，创建了一个(含多个keyframe)的KeyframeSet对象

```java
public class ObjectAnimator {
    public static ObjectAnimator ofFloat(Object target, String propertyName, float... value) {
        ObjectAnimator anim = new ObjectAnimator(target, propertyName);
        anim.setFloatValues(values);
        return anim;
    }
}
```

### ObjectAnimator.setDuration：

- 该方法用于设置动画的执行总时间，调用父类的ValueAnimator的方法：

### ObjectAnimator.start：

.
.
.

### Choreographer类

这是动画最为核心的一个类，动画最后都会走到这个类里面。

见[Choreographer](../choreographer.md)

#### schduleFrameLocked

```java
public class Choreographer {
    public void schduleFrameLocked(long now) {
        if (USE_VSYNC) {
            if (isRunningOnLooperThreadLocked()) {
                schduleVsyncLocked();
            } else {
                Messaage msg = mHandler.obtainMessage(MS_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        }
    }
}
```

其中USE_VSYNC = SystemProperties.getBoolean("debug.choreographer.vsync", true),该属性值一般都是缺省的，
则USE_VSYNC = true，启动VSYNC垂直同步信号方式来触发动画，当fps=60时候，则1/60=16.7，故VSYNC信号上报的周期为16.7ms；

#### scheduleVsyncLocked

### AnimationHandler

通过Choreographer AnimationHandler获得回调

```java
public class AnimationHandler {
    private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
        @Override
        public void doFrame(long frameTimeNanos) {
            doAnimationFrame(getProvider().getFrameTime());
            if (mAnimationCallbacks.size() > 0) {
                getProvider().postFrameCallback(this);
            }
        }
    }

    private void doAnimationFrame(long frameTime) {
        long currentTime = SystemClock.uptimeMillis();
        final int size = mAnimationCallbacks.size();
        for (int i = 0; i < size; i++) {
            final AnimationFrameCallback callback = mAnimationCallbacks.get(i);
            if (callback == null) {
                continue;
            }
            if (isCallbackDue(callback, currentTime)) {
                callback.doAnimationFrame(frameTime);
                if (mCommitCallbacks.contains(callback)) {
                    getProvider().postCommitCallback(new Runnable() {
                        @Override
                        public void run() {
                            commitAnimationFrame(callback, getProvider().getFrameTime());
                        }
                    });
                }
            }
        }
    }
}
```

```java
public class ValueAnimator {
    public final boolean doAnimationFrame(long frameTime) {
        boolean finished = animateBasedOnTime(currentTime);
    }


    boolean animateBaseOnTime(long currentTime) {
        if (mRunning) {
            animateValue(currentIterationFraction);
        }
    }

    void animateValue(float fraction) {
        for (int i = 0; i < numValues; ++i) {
            mValues[i].calculateValue(fraction);
        }
    }
}
```

```java
public class ObjectAnimator {
    void animateValue(float fraction) {
        final Object target = getTarget();
        if (mTarget != null && target == null) {
            cancel();
            return;
        }
        super.animateValue(fraction);
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].setAnimatedValue(target);
        }
    }
}
```

```java
public class PropertyValuesHolder {
    void calculateValue(float fraction) {
        Object value = mKeyframes.getValue(fraction);
        mAnimatedValue = mConverter == null ? value : mConverter.convert(value);
    }

    void setAnimatedValue(Object target) {
        if (mProperty != null) {
            mProperty.set(target, getAnimatedValue());
        }
        if (mSetter != null) {
            try {
                mTemValueArray[0] = getAnimatedValue();
                mSetter.invoke(target, mTmpValueArray);
            }
        }
    }
}
```

## 插值器

插值器是一种的动画修改器，它影响动画中的变化率。 这可以让您现有的动画效果加速，减速，重复，反弹等。
插值器应用于具有android：interpolator属性的动画元素，其值是对插值器资源的引用。
Android中提供的所有插补器都是Interpolator类的子类。 对于每个插值器类，Android都包含一个公共资源，您可以使用android：interpolator属性将插值器应用于动画。

mChoreographer的postCallback方法来启动动画执行的，相当于起了一个定时器来不断更新属性值alpha来实现动画刷新

choreographer这个类是用来控制同步处理输入(Input)、动画(Animation)以及绘制(Draw)三个UI操作的，通过接收显示系统的时间脉冲(垂直同步信号-VSync信号)，在下一个Frame渲染时控制执行这些操作。
