
# 测量、定位和绘制

任何一控件其实都是直接或间接继承自View实现的，所以说View应该具有相同的绘制流程与机制才能显示到屏幕上
每一个View的绘制过程都必须经历三个最主要的过程，也就是measure、layout和draw。

```text
├── View
│   ├── measure(int, int) 测量view多大，父提供参数，该方法会调用onMeasure(int,int)
|   ├── onMeasure(int, int) 此方法可以被覆盖，子类有责任测量宽高
|   ├── layout(int, int, int, int) 布局定位，除了ViewGroup实现了,其他并不需要
|   ├── onLayout(boolean, int, int, int, int) 包含子的派生类需要实现，并且每个子要调用layout方法
|   ├── onDraw(Canvas) 绘制需要重写的方法
|   ├── draw(Canvas) 绘制流程的主方法，一般不重写
|   ├── draw(Canvas, ViewGroup, long) 该方法是ViewGroup.drawChild调用的，用于每个子类进行绘制自身
|   ├── dispatchDraw(Canvas) 绘制流程用于绘制子views，会被派生类重载，在children绘制之前。

├── ViewGroup
│   ├── layout(int, int, int, int) //ViewGroup实现了
│   ├── onLayout(boolean, int, int, int, int) //viewGroup是一个抽象类 需要实现
```

## 初始化工作

Activity方法onCreate里执行了setContentView只有View如何显示到屏幕上的。

1. Activity.setContentView->PhoneWindow.setContentView最终会生成一个DecorView对象
2. DecoreView是PhoneWindow类的内部类，继承自FrameLayout，所以调用Activity方法

```text
ViewRootImpl->performTraversals->performMeasure->performLayout->performDraw

performMeasure->view.measure->view.onMeasure
performLayout->view.layout->view.onLayout
performDraw->view.draw->view.onDraw
```

ViewRootImpl，他是链接WindowManager和DecoreView的纽带。更广阔可以说是Window和View之间的纽带
完成View的绘制
向DecoreView分发收到的用户发起的event事件，如按键，触屏等事件

Window是一个抽象概念，每一个Window都对应一个View和一个ViewRootImple,Window又通过ViewRootImpl与View建立联系
![image](/img/hierarchy.png)

## 开始

整个View树的绘图流程是在ViewRootImpl类的performTraversals()方法，该函数做的执行过程主要是根据之前设置的状态，判断是否重新计算视图大小
(measure)、是否重新放置视图的位置(layout)、以及是否重绘(draw)，其核心也就是判断来选择顺序执行这三个方法中的哪个。

1. Android自上而下对所有View进行量算，这样Android就知道了每个View想要的大小，即宽高信息
2. 在完成了所有的View的量算工作后，Android会自上而下对所有View进行布局，Android就知道了每个View在其父控件中的位置，即View到其父控件四边
3. 在完成了对所有View的布局工作后，Android会自上而下对所有View进行绘图，这样Android就将所有的View渲染到屏幕上。

### MeasureSpec的含义

MeasureSpec并不是指View的测量宽高，MeasureSpec作用在于：在Measure流程中，系统会将
View的LayoutParams根据父容器所施加的规则转换成对应的MeasureSpec，
然后在onMeasure方法中根据这个MeasureSpec来确定View的测量宽高。

- MeasureSpec数值（数值1080(二进制为: 1111011000)为例）

    |模式名称|模式数值(高2位)|实际数值(低30位)|
    |---|---|----|
    |UPSPECIFIED|00|000000000000000000001111011000|
    |EXACTLY|01|000000000000000000001111011000|
    |AT_MOST|10|000000000000000000001111011000|
- View.MeasureSpec:

    |模式|二进制数值|描述|
    |---|---|----|
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

## 总结

### 详细流程图

![详细流程图](/img/view_measure_layout_draw.png)

### 总结信息

1. 渲染Android应用视图的渲染流程，测量流程用来确定视图的大小、布局流程用来确定视图的位置、绘图流程最终将视图绘制在应用窗口上
2. Android应用程序窗口UI首先是使用Skia图形库API来绘制在一块画布上，实际地是绘制在这块画布里面的一个图形缓冲区中，这个图形缓冲区最终会被交给SurfaceFlinger服 务，而SurfaceFlinger服务再使用OpenGL图形库API来将这个图形缓冲区渲染到硬件帧缓冲区中。

### 补充章节

#### 确定View大小（onSizeChanged

**Q: 在测量完View并使用setMeasuredDimension函数之后View的大小基本上已经确定了，那么为什么还要再次确定View的大小呢？**
**A: 这是因为View的大小不仅由View本身控制，而且受父控件的影响，所以我们在确定View大小的时候最好使用系统提供的onSizeChanged回调函数。**

### 重点知识梳理

### 自定义View分类

> PS ：实际上ViewGroup是View的一个子类。

| 类别        | 继承自                 | 特点      |
| --------- | ------------------- | ------- |
| View      | View SurfaceView 等  | 不含子View |
| ViewGroup | ViewGroup xxLayout等 | 包含子View |

### 自定义View流程

| 步骤   | 关键字           | 作用                           |
| ---- | ------------- | ---------------------------- |
| 1    | 构造函数          | View初始化                      |
| 2    | onMeasure     | 测量View大小                     |
| 3    | onSizeChanged | 确定View大小                     |
| 4    | onLayout      | 确定子View布局(自定义View包含子View时有用) |
| 5    | onDraw        | 实际绘制内容                       |
| 6    | 提供接口          | 控制View或监听View某些状态。           |
