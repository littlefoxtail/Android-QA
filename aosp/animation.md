动画资源分为两类
- 属性动画
使用`Animator`在设定的一段时间修改对象的属性值来创建动画
- View动画
    - Tween(补间)动画 
    通过使用`Animation`在单个图像上执行一系列转换来创建动画
    - Frame(帧)动画
    通过使用`AnimationDrawable`顺序显示一系列图像来创建动画


# 属性动画
在XML中定义的动画，可在一段时间内修改目标对象的属性值，如背景颜色或alpha值
![image](../img/objectAnimator.png)

属性动画功能非常强大，也是最常用的动画方法。可自定义如下属性：
- 动画时间(Duration)，指定动画总共完成所需要的时间，默认为300ms
- 时间插值器(Time interpolation)，是一个基于当前动画已消耗时间的函数，用来计算属性的值
- 重复次数(Repeat count)：指定动画是否重复执行，重复执行的次数，也可以指定动画向反方向地回退操作
- 动画集(Animator sets)，将一系列动画放进一个组，可以设置


# 插值器
插值器是一种的动画修改器，它影响动画中的变化率。 这可以让您现有的动画效果加速，减速，重复，反弹等。
插值器应用于具有android：interpolator属性的动画元素，其值是对插值器资源的引用。
Android中提供的所有插补器都是Interpolator类的子类。 对于每个插值器类，Android都包含一个公共资源，您可以使用android：interpolator属性将插值器应用于动画。 


mChoreographer的postCallback方法来启动动画执行的，相当于起了一个定时器来不断更新属性值alpha来实现动画刷新

choreographer这个类是用来控制同步处理输入(Input)、动画(Animation)以及绘制(Draw)三个UI操作的，通过接收显示系统的时间脉冲(垂直同步信号-VSync信号)，在下一个Frame渲染时控制执行这些操作。


