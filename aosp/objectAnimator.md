mChoreographer的postCallback方法来启动动画执行的，相当于起了一个定时器来不断更新属性值alpha来实现动画刷新

horeographer这个类是用来控制同步处理输入(Input)、动画(Animation)以及绘制(Draw)三个UI操作的，通过接收显示系统的时间脉冲(垂直同步信号-VSync信号)，在下一个Frame渲染时控制执行这些操作。

## 属性动画
![image](../img/objectAnimatorimg.jpg)