# Managing Touch Events in a ViewGroup


![图裂](/img/function_touch.png)

![触摸事件时序图](/img/触摸事件时序图.png)

```text
├── View
│   ├── ViewGroup

├── View
│   ├── dispatchTouchEvent()
│   ├── onTouchEvent()

├── ViewGroup
│   ├── this.dispatchTouchEvent() //分派事件
│   ├── onInterceptTouchEvent() //拦截事件
│   ├── super.onTouchEvent() //处理事件
```

下表省略了 PhoneWidow 和 DecorView。

> `√` 表示有该方法。
> `X` 表示没有该方法。

| 类型   | 相关方法                  | Activity | ViewGroup | View |作用|调用时刻|
| ---- | --------------------- | -------- | --------- | ---- |:--:|:--:|
| 事件分发 | dispatchTouchEvent    | √        | √         | √    |分发点击事件|当点击事件能够传递给当前View时，该方法就会被调用|
| 事件拦截 | onInterceptTouchEvent | X        | √         | X    |判断是否拦截了某个事件|在dispatchTouchEvent()内部调用|
| 事件消费 | onTouchEvent          | √        | √         | √    |处理点击事件|在dispatchTouchEvent()内部调用|

这个三个方法均有一个 boolean(布尔) 类型的返回值，通过返回 true 和 false 来控制事件传递的流程。

PS: 从上表可以看到 Activity 和 View 都是没有事件拦截的，这是因为：

> Activity 作为原始的事件分发者，如果 Activity 拦截了事件会导致整个屏幕都无法响应事件，这肯定不是我们想要的效果。
>
> View最为事件传递的最末端，要么消费掉事件，要么不处理进行回传，根本没必要进行事件拦截。
1. Activity
	- ViewRootImpl中DecorView分发touch事件
	- Activity的dispatchTouchEvent
	- DecorView的superDispatchKeyEvent
	- ViewGroup的dispatchTouchEvent
2. ViewGroup
	- dispatchTouchEvent
		- onInterceptTouchEvent
		- onTouchEvent
3. 分发和消费两个方法，如果不处理这个事件，会被传递给父视图

### 在Native层android系统的事件流程：

- Android系统是从底层驱动中获取各种原始的用户消息，包括按键、触摸屏、鼠标
- 在获取用户消息之后，android系统会对最原始的消息进行预处理，包括两个方面：一个方面讲消息转化成系统可以处理的消息事件；另一个方面，处理一些特殊的事件，比如HOME/MENU/POWER键等处理
- Android系统使用InputManager类来管理消息，而具体功能则是通过InputReaderThread和InputDispatcherThread线程来实现。其中InputReaderThread线程负责消息的读取，而InputDispatcherThread则负责消息的预处理和分发到各个应用进程中。
- Activity系统在SystemServer进程中启动WindowManagerService服务，然后在WindowManagerService服务中启动InputManagerService

### ViewRootImpl

在Native层的事件分发线程中，经过事件的分发流程，最终会调用InputEventSender的dispatchInputEventFinished

```java
private void dispatchInputEventFinished(int seq, boolean handled) {
    onInputEventFinished(seq, handled);
}
```

Native层最终调用的是ImeInputEventSender

```java
private final class ImeInputEventSender extends InputEventSender {
    public ImeInputEventSender(InputChannel inputChannel, Looper looper) {
        super(inputChannel, looper);
    }

    @Override
    public void onInputEventFinished(int seq, boolean handled) {
        finishedInputEvent(seq, handled, false);
    }
}
```

```java
void finishedInputEvent(int seq, boolean handled, boolean timeout) {
    final PendingEvent p;
    synchronized(mH) {
        int index = mPendingEvents.indexOfKey(seq);
        if (index < 0) {
            return;
        }
        p = mPendingEvents.valueAt(index);
        mPendingEvents.removeAt(index);

        invokeFinishedInputEventCallback(p, handled);
    }
}
```

```java
void invokeFinishedInputEventCallback(PendingEvent p, boolean handled) {
    p.mHandled = handled;
    if (p.mHandler.getLooper().isCurrentThread()) {
        p.run();
    } else {
        Message msg = Message.obtain(p.mHandler, p);
        msg.setAsynchronous(true);
        msg.sendToTarget();
    }
}
```

InputMethodManager

```java
public void run() {
    mCallback.onFinishedInputEvent(mToken, mHandled);
    synchronized(mH) {
        recyclePendingEventLocked(this);
    }
}
```

可以发现在run方法中我们调用了mCallback的onFinishedInputEvent方法，需要说明的是这里的mCallback就是我们ViewRootImpl中的ImeInputStage类对象

ViewRootImpl@ImeInputStage

```java
final class ImeInputStage extends AsyncInputStage 
        implements InputMethodManager.FinishedInputEventCallback {
    public void onFinshedInputEvent(Object token, boolean handled) {
        QueuedInputEvent q = (QueuedInputEvent)token;
        if (handled) {
            finish(q, true);
            return;
        }
        forward(q);
    }
}
```

经过一系列责任链：

```text
android.view.ViewRootImpl$ViewPostImeInputStage.processKeyEvent(ViewRootImpl.java:4152)
at android.view.ViewRootImpl$ViewPostImeInputStage.onProcess(ViewRootImpl.java:4114)
at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:3662)
at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:3715)
at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:3681)
at android.view.ViewRootImpl$AsyncInputStage.forward(ViewRootImpl.java:3807)
at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:3689)
at android.view.ViewRootImpl$AsyncInputStage.apply(ViewRootImpl.java:3864)
at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:3662)
at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:3715)
at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:3681)
at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:3689)
at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:3662)
at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:3715)
at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:3681)
at android.view.ViewRootImpl$AsyncInputStage.forward(ViewRootImpl.java:3840)
at android.view.ViewRootImpl$ImeInputStage.onFinishedInputEvent(ViewRootImpl.java:4006)
at android.view.inputmethod.InputMethodManager$PendingEvent.run(InputMethodManager.java:2272)
at android.view.inputmethod.InputMethodManager.invokeFinishedInputEventCallback(InputMethodManager.java:1893)
at android.view.inputmethod.InputMethodManager.finishedInputEvent(InputMethodManager.java:1884)
at android.view.inputmethod.InputMethodManager$ImeInputEventSender.onInputEventFinished(InputMethodManager.java:2249)
at android.view.InputEventSender.dispatchInputEventFinished(InputEventSender.java:141)
```

如果没有任何View消费掉事件，那么这个事件会按照反方向回传，最终传回给Activity，如果最后 Activity 也没有处理，本次事件才会被抛弃:

```text
Activity <－ PhoneWindow <－ DecorView <－ ViewGroup <－ ... <－ View
```

```java
if (mFirstTouchTarget == null) {
    handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
}
```

### 事件分发的对象？

- 当用户触摸屏幕时候(View或者ViewGroup派生的控件)，将产生点击事件(Touch事件)
    > Touch事件相关细节被封装成MotionEvent对象

- 主要发生的Touch事件有如下四种：
  - MotionEvent.ACTION_DOWN：按下View（所有事件的开始）
  - MotionEvent.ACTION_MOVE：滑动View
  - MotionEvent.ACTION_CANCEL：非人为原因结束本次事件
  - MotionEvent.ACTION_UP：抬起View（与DOWN对应）

### 事件分发的本质

将点击事件(MotionEvent)向某个View进行传递并最终得到处理

关于`ViewGroup`的触摸事件，要能正确处理Touch事件。必须重写`onInterceptTouchEvent`方法。

## Intercept Touch Events in a ViewGroup

************

当`ViewGroup`检测到有事件发生时，`onInterceptToucheEvent()`将会被调用。包含`ViewGroup`和`View`。  
如果`onInterceptTouchEvent()`返回true，表示`MotionEvent`已经被拦截。他将不会下发到他的子视图。  
而是到当前`ViewGroup`的`onTouchEvent()`。

`onInterceptTouchEvent()`会拿到子视图的`MotionEvent`。  
如果`onInterceptTouchEvent()`返回true，发给子视图的事件将是ACTION_CANCEL.并且子视图的事件将会交给`ViewGroup`处理。

> 注意，ViewGroup提供了一个requestDisallowInterceptTouchEvent()方法，调用此方法后，ViewGroup将关闭拦截效果

## Use ViewConfiguration Constants

************

上面的代码片段使用了`ViewConfigutation`来获取一个变量`mTouchSlop`，你可以通过`ViewConfiguration`来获取
一些常用的距离，速度，次数。
`Touch slop`用来获取用户手指可以滑动的最大距离。用来避免一些偶然情况的发生。
另外两个`ViewConfiguration`的方法是`getScaledMinimumFlingVelcity()`和`getScaledMaximumFlingVelocity()`。这两个方法返回滑动的最小和最大速度值。

## Extend a Child View's Touchable Area（扩展子视图的可触摸面积）

android提供了一个`TouchDelegate`类，这让view的可触摸区域比本身区域更大变成可能。当子视图不得不很小时，这个类是非常有用的。如果想这么做的话，你也可以使用
此类来缩小视图的可触摸面积。

下面的例子中`ImageBotton`就是这样。

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
     android:id="@+id/parent_layout"
     android:layout_width="match_parent"
     android:layout_height="match_parent"
     tools:context=".MainActivity" >

     <ImageButton android:id="@+id/button"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:background="@null"
          android:src="@drawable/icon" />
</RelativeLayout>
```

下面这些代码片段做了这些事情：

- 获取其父视图并且在主线程上新建一个线程。  
    这确保了在父视图调用`getHitRect()`之间已经绘制好了子视图，这个方法用于父视图定位子视图的可触摸区域。
- 根据`ImageButton`的`getHitRect`来获取可触摸的矩形区域。
- 扩展`ImageButton`所命中的可触摸的矩形区域
- 实例化`TouchDelegate`，需要一个矩形区域和视图作为参数。
- 在父视图上设置`TouchDelegate`，使用委托模式。原理：使用委托模式的情况下，
    父视图将接受子视图的所有事件，当事件发生的区域在子视图区域，才会将这些事件下发给子视图。

```java
public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // Get the parent view
        View parentView = findViewById(R.id.parent_layout);

        parentView.post(new Runnable() {
            // Post in the parent's message queue to make sure the parent
            // lays out its children before you call getHitRect()
            @Override
            public void run() {
                // The bounds for the delegate view (an ImageButton
                // in this example)
                Rect delegateArea = new Rect();
                ImageButton myButton = (ImageButton) findViewById(R.id.button);
                myButton.setEnabled(true);
                myButton.setOnClickListener(new View.OnClickListener() {
                    @Override
                    public void onClick(View view) {
                        Toast.makeText(MainActivity.this,"Touch occurred within ImageButton touch region.",
                                Toast.LENGTH_SHORT).show();
                    }
                });

                // The hit rectangle for the ImageButton
                myButton.getHitRect(delegateArea);

                // Extend the touch area of the ImageButton beyond its bounds
                // on the right and bottom.
                delegateArea.right += 100;
                delegateArea.bottom += 100;

                // Instantiate a TouchDelegate.
                // "delegateArea" is the bounds in local coordinates of
                // the containing view to be mapped to the delegate view.
                // "myButton" is the child view that should receive motion
                // events.
                TouchDelegate touchDelegate = new TouchDelegate(delegateArea,
                        myButton);

                // Sets the TouchDelegate on the parent view, such that touches
                // within the touch delegate bounds are routed to the child.
                if (View.class.isInstance(myButton.getParent())) {
                    ((View) myButton.getParent()).setTouchDelegate(touchDelegate);
                }
            }
        });
    }
}
```

## 一、概述

Android的事件分发是遵循类似责任链模式的，就是从根节点开始逐层往里分发事件，直到找到责任人（即响应事件的View）或找不到责任人事件“丢弃”为止。

## 二、触摸事件有下面一系列动作：

|   **动作**   | **描述** |
| :------------: | :----: |
|     ACTION_DOWN   |  表示用户开始触摸  |
|     ACTION_MOVE   |  表示用户在移动  |
|     ACTION_UP   |  表示用户抬起了手指  |
|     ACTION_CANCEL   |  表示手势被取消了（比如当你按下了按钮，然后移动到别处（按钮区域外）抬起，就会识别为事件取消）  |
|     ACTION_OUTSIDE   |  表示用户触碰超出了正常的UI边界  |
|     ACTION\_POINTER\_DOWN   |  有一个非主要的手指按下了  |
|     ACTION\_POINTER\_UP   |  有一个非主要的手指抬起来了  |

ACTION\_DOWN->ACTION\_MOVE->ACTION\_MOVE...->ACTION_UP.

## 三、对Touch事件的基本认知

* 一个事件只能被消费(consume)一次，事件消费了就不再传递。
* 一个事件只能有一个责任人
* 事件以按下为起点，谁消费了按下事件，后续的事件就交给谁处理。
* View可以自己处理事件，也可以分发给子View处理。

## 四、谁来处理Touch事件

![View](/img/View.jpg)

## 五、怎么处理Touch事件

事件的大概流程：事件接收层(底层：硬件和软件，一般不需要了解)--->窗口管理系统WindowManagerServicer-->因为
所有的窗口都是由他创建的，所以WMS知道当前活动的窗口是谁，WMS将事件交给当前活动窗口--->当前活动窗口拿到事件，调用
ViewRoot类的dispatchTouchEvent，给当前活动窗口的根view-->根view开始dispatchTouchEvent事件到具体view。

从根布局到子布局，即事件先传递给根布局，然后在传递给子布局

## 六、View中对事件的处理

在View中定义了跟事件处理相关的两个重要函数
![dispatchTouchEvent](/img/view_dispatchTouchEvent.jpg)

![onTouchEvent](/img/view_onTouchEvent.jpg)

## 七、ViewGroup中对事件的处理

ViewGroup是View的子类，所以自然继承了View的上述两个方法。ViewGroup还重写了`dispatchTouchEvent`方法。
ViewGroup包含了多个View，事件分发时总要先判断事件落在哪个View中，不像非ViewGroup那样简单。

ViewGroup@dispatchTouchEvent

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // 把ACTION_DOWN作为一个Touch手势的起始点，清除之前的手势状态
            // 把mFirstTouchTarget重置为null(mFirstTouchTarget为最近一次对处理Touch事件的View)
            cancelAndClearTouchTargets(ev);
            // 重置FLAG_DISALLOW_INTERCEPT该标记为true则表示禁止拦截本次事件
            resetTouchState();
        }
        // 判断是否拦截
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
                // 本次事件是ACTION_DOWN，或者mFirstTouchTarget不为null
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                // 判断禁止拦截的FLAG，因为子View可以通过requestDisallowInterceptTouchEvent禁止父View执行是否需要拦截的判断
                if (!disallowIntercept) {
                    // 禁止拦截的FLAG为false，说明可以执行拦截判断
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action);
                } else {
                    intercepted = false;
                }
        } else {
            intercepted = true;
        }

        if (!canceled && !intercepted) {
            // 非cancel且不拦截
            if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN) 
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex();
                    final int childredCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
// 寻找可以接收事件的child，扫描children从前到后
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();// buildTouchDispatchChildList内部会调用buildOrderedChildList方法，该方法会将子元素根据Z轴排序，同一个Z平面上的子元素则会根据绘制的先后顺序排序
                        final boolean customOrder = preorderedList == null && isChildrenDrawingOrderEnabled();
                        final View[] childrend = mChildren;
                        // 逆序遍历所有child
                        for (int i = childrenCount - 1; i >=0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(preorderedList,children, childIndex);

                            newTouchTarget = getTouchTarget(child);

                        // 把ACTION_DOWN事件传递给子View进行处理
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // 如果此子View消费了这个touch事件
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }


                        }
                    }
                }
        }
        if (mFirstTouchTarget == null) {
// 第三个参数为null， child = null
            handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
        }
    }
}
```
### DOWN事件时进行初始化
```java
//1. DOWN事件进行初始化，清空 TouchTargets 和 TouchState
//dispatchTouchEvent
if (actionMasked == MotionEvent.ACTION_DOWN) {  

	cancelAndClearTouchTargets(ev);  
    resetTouchState();  
}

private void cancelAndClearTouchTargets(MotionEvent event) {  
    if (mFirstTouchTarget != null) {  
        boolean syntheticEvent = false;  
        if (event == null) {  
            final long now = SystemClock.uptimeMillis();  
            event = MotionEvent.obtain(now, now,  
                    MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);  
            event.setSource(InputDevice.SOURCE_TOUCHSCREEN);  
            syntheticEvent = true;  
        }  
  
        for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {  
            resetCancelNextUpFlag(target.child);  
            dispatchTransformedTouchEvent(event, true, target.child, target.pointerIdBits);  
        }  
        clearTouchTargets();  
  
        if (syntheticEvent) {  
            event.recycle();  
        }  
    }}
```

```java
private void clearTouchTargets() {  
    TouchTarget target = mFirstTouchTarget;  
    if (target != null) {  
        do {  
            TouchTarget next = target.next;  
            target.recycle();  
            target = next;  
        } while (target != null);  
        mFirstTouchTarget = null;  
    }  
}
```
一个事件列表是由一个ACTION_DOWN，零个或多个ACTION_MOVE和一个ACTION_UP组成的。
用循环的方式把单链表*mFirstTouchTarget*给清空了。
### 检查是否拦截
```java
final boolean intercepted;  
if (actionMasked == MotionEvent.ACTION_DOWN  
        || mFirstTouchTarget != null) { 
		//这个标记是子View设置ViewGroup是否允许拦截的情况
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;  
    if (!disallowIntercept) {  
        intercepted = onInterceptTouchEvent(ev);  
        ev.setAction(action); 
    } else {  
        intercepted = false;  
    }  
} else {  
    intercepted = true;  
}
```
mFirstTouchTarget：
-  如果intercepted为false，后面要又一个子View的dispatchTouchEvent方法在ACTION_DOWN时返回了true，那么会对mFirstTouchTarget进行赋值
- 如果intercepted为true，那么后面就会对mFirstTouchTarget置为null。

**总结，onInterceptTouchEvent一旦某一次返回了ture，那么后面的事件都不会再调用onInterceptTouchEvent进行是否拦截的判断，intercepted的值一直为true**
### 处理DOWN事件
```java
//如果没有被拦截，先处理DOWN事件，主要赋值TouchTarget
if (actionMasked == MotionEvent.ACTION_DOWN  
        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)  
        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {

	for (int i = childrenCount - 1; i >= 0; i--) {  
    final int childIndex = getAndVerifyPreorderedIndex(  
            childrenCount, i, customOrder);  
    final View child = getAndVerifyPreorderedView(  
            preorderedList, children, childIndex);
//找到Visible并且处于点击范围的子View
	if (!child.canReceivePointerEvents()  
        || !isTransformedTouchPointInView(x, y, child, null)) {  
    ev.setTargetAccessibilityFocus(false);  
    continue;  
//相当于调用子View的dispatchTouchEvent
	if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {  
    // Child wants to receive touch within its bounds.  
    mLastTouchDownTime = ev.getDownTime();  
    if (preorderedList != null) {  
        // childIndex points into presorted list, find original index  
        for (int j = 0; j < childrenCount; j++) {  
            if (children[childIndex] == mChildren[j]) {  
                mLastTouchDownIndex = j;  
                break;  
            }  
        }    } else {  
        mLastTouchDownIndex = childIndex;  
    }  
    mLastTouchDownX = ev.getX();  
    mLastTouchDownY = ev.getY();  
    newTouchTarget = addTouchTarget(child, idBitsToAssign);  
    alreadyDispatchedToNewTouchTarget = true;  
    break;  
}
}
}
```
遍历每一个满足条件（处于Visible状态并且处于点击范围内）的子View，调用了dispatchTransformedTouchEvent，返回值为true，则调用addTouchTarget方法对mFirstTouchTarget进行赋值
```java
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {  
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);  
    target.next = mFirstTouchTarget;  
    mFirstTouchTarget = target;  
    return target;  
}
```
### ViewGroup@dispatchTransformedTouchEvent()
```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
                View child, int desiredPointerIdBits) {
    final boolean handled;

    final int oldAction = event.getAction()    ;
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
//如果传入的child不为空，则调用child的dispatchTouchEvent方法，否则调用自身的dispatchTouchEvent方法
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
}
```
dispatchTransformedTouchEvent主要做了两件事：
- 如果传入的事件是*ACTION_CANCEL*，或者cancel参数为true，则直接分发*ACTION_CANCEL*事件
- 分发过程中，如果child为空，则调用当前View的super.dispatchTouchEvent方法

## 八、Activity对Touch事件的处理

Activity持有一个Window，而Window持有一个DecorView。而事件是至上而下分发的，所以Activity对事件拥有最高的
优先处理权，它可以决定是否要将事件分发给Window。

默认情况下`dispatchTouchEvent`返回值是true，分发的；当没有任何的View来处理触摸事件时，会系统调用`onTouchEvent`方法。

## 九、总结

1. 事件的处理是至上而下的，从Activity到ViewGroup再到ViewGroup。
2. 理清事件处理关键在onDispatchTouchEvent方法。在这个方法中会做一个抉择：是要直接分发给子View处理，或是先交给自己onTouchEvent方法处理后再抉择。
3. ViewGroup作为容器类View，对事件的处理多了onInterceptTouchEvent这个阻断方法，其实我们只要看onDispatchTouchEvent就行了，因为它会在这个方法中调用onInterceptTouchEvent做是否阻断的判定。
4. 返回true，通常表示处理或消费了事件，不再传递。

通过Thread.dumpStack()来打印出当前线程的调用栈信息

> 1. 不管是ViewRootImpl的dispatchInputEvent方法，还是WindowInputEventReceiver的dispatchInputEvent方法，它们本质上都是调用deliverInputEvent方法来处理点击事件的消息。
