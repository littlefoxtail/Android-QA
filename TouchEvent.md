# Managing Touch Events in a ViewGroup
> Managing Touch Events in a ViewGroup

```
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
* 获取其父视图并且在主线程上新建一个线程。  
这确保了在父视图调用`getHitRect()`之间已经绘制好了子视图，这个方法用于父视图定位子视图的可触摸区域。
* 根据`ImageButton`的`getHitRect`来获取可触摸的矩形区域。
* 扩展`ImageButton`所命中的可触摸的矩形区域
* 实例化`TouchDelegate`，需要一个矩形区域和视图作为参数。
* 在父视图上设置`TouchDelegate`，使用委托模式。原理：使用委托模式的情况下，
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

触摸事件有下面一系列动作：

|!-----!|!-----!|
| 动作 | 表示 |
|ACTION_DOWN|表示用户开始触摸|

|   **动作**   | **描述** |
| :------------: | :----: |
|     ACTION_DOWN   |  表示用户开始触摸  |
|     ACTION_MOVE   |  表示用户在移动  |
|     ACTION_UP   |  表示用户抬起了手指  |
|     ACTION_CANCEL   |  表示手势被取消了  |
|     ACTION_OUTSIDE   |  表示用户触碰超出了正常的UI边界  |
|     ACTION\_POINTER\_DOWN   |  有一个非主要的手指按下了  |
|     ACTION\_POINTER\_UP   |  有一个非主要的手指抬起来了  |

ACTION\_DOWN->ACTION\_MOVE->ACTION\_MOVE...->ACTION_UP.

事件的大概流程：事件接收层(底层：硬件和软件，一般不需要了解)--->窗口管理系统WindowManagerServicer-->因为
所有的窗口都是由他创建的，所以WMS知道当前活动的窗口是谁，WMS将事件交给当前活动窗口--->当前活动窗口拿到事件，调用
ViewRoot类的dispatchTouchEvent，给当前活动窗口的根view-->根view开始dispatchTouchEvent事件到具体view。

1. 如果是具体view，（非ViewGroup），如TextView等，这种情况下根本不存在，因为每个窗口，肯定有个
    rootview，及viewGroup

2. 如果是ViewGroup，比如LinearLayout等。
    * 如果是down事件，开始下面逻辑。
    * (-递归开始点-)首先调用viewGroup的dispatchTouchEvent方法，如果是down事件，则清空上次
    处理该事件的对象（为了处理MOVE之类的事件，做的缓存）mMotionTarget= null;
    * 调用onInterceptTouchEvent方法，这个方法只有ViewGroup类有，具体view没有，该方法的作用
    是判断是否需要拦截该消息，如果返回的是true，那么消息传递结束，调用该view对象的onTouchEvent方法。
    如果返回的是false，说明该view没有消费事件，继续往下走。
    * 因为触摸事件是窗口坐标值，所以需要将坐标值转换为view自己的坐标体系。
    * 转换结束后，使用for循环遍历，该view的所有子view，读取子view的坐标体系，即子view所占的大小，是个
    Rect对象，上下左右，拿到这个值后，根据上面转换好的坐标，判断点击的坐标是否包含在当前子view中，如果不包含
    ，直接开始下一个子view
    * 如果坐标包含在子view中，则调用子view的dispatchTouchEvent，如果子view还是ViewGroup类型的，
    那么开始从上面标有（递归开始点）处递归调用，如果是具体view，则调用具体view的dispatchTouchEvent，
    这个方法比较简单，首先判断是否通过setTouchEventListener设置值，如果设置了，那么调用onTouch方法，
    如果该方法返回的是true，则直接返回，不在调用该view的onToucheEvent方法，如果返回false，则调用该
    view的onTouchEvent方法。并把该方法当做dispatchTouchEvent的返回者返回。




