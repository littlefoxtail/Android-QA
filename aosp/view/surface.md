# surface的源码分析

每一个在C++层实现的应用程序窗口都需要有一个绘图表面，然后才可以将自己的UI表现出来。这个绘图表面是需要由应用程序进程请求SurfaceFlinger服务来创建的，在SurfaceFlinger服务内部使用一个实现了ISurface接口的Binder本地对象给应用程序进程，于是，应用程序进程就可以获得一个实现了ISurface接口的Binder代理对象。有了这个实现ISurface接口的Binder代理对象。有了这个实现ISurface接口的Binder代理对象之后，在C++层实现的应用程序窗口就可以请求SurfaceFlinger服务分配图形缓冲区以及渲染已经填充好UI数据的图形缓冲区

## Surface创建过程

在应用进程这一侧，每一个应用程序窗口，即每一个Activity组件，都有一个关联的Surface对象，这个Surface对象是保在一个关联的ViewRootImpl对象的成员变量mSurface中。

在WindowManagerService服务这一侧，每一个应用程序窗口对有一个WindowState对象，这个WindowState.winAnimator.mSurfaceControl成员变量中
> 通过`getSurface()`方法获取c++ surface中数据

虽然应用进程和WindowManagerService服务都是用来操作位于SurfaceFlinger服务中的同一个Layer对象

1. 位于应用程序的Surface对象负责绘制应用程序窗口的UI，即往应用程序窗口的图形缓冲区填充UI数据
2. WindowManagerService服务这一侧的Surface对象负责设置应用程序窗口的属性
这两种不同的操作方式分别是通过C++层的Surface对象和SurfaceControl对象来完成的。

### 创建步骤

![image](/img/surface_create.png)

## SurfaceView

SurfaceView 是 Android 中一种比较特殊的视图（View），它跟平时时候的 TextView、Button 最大的区别是它跟它的视图容器并不是在同一个视图层上，它的 UI 显示也可以不在一个独立的线程中完成，所以对 SurfaceView 的绘制并不会影响到主线程的运行。综合这些特点，SurfaceView 一般用来实现动态的或者比较复杂的图像还有动画的显示。

### SurfaceView的MVC框架

Model:Surface
Controller:SurfaceHolder
View:SurfaceView

### 应用场景

区别与普通的控件，Suerface View可以运行于主线程之外，不需要及时响应用户的输入，也不会造成ANR问题。SurfaceView一般用在游戏、视频、摄影等一些复杂UI及
高效的图像的显示，这类的图像处理都需要开单独的线程来处理。

#### Surface

```java
public class Surface implements Parcelable {

}
```

Surface实现了Parcelable接口进行序列化，出来处理屏幕显示缓冲区的数据，源码对它的注释为*Handle onto a raw buffer that is being managed by the screen compositor*。Surface是原始图像缓冲区的一个句柄，而原始图像缓冲区由屏幕图像合成器(screen compositor)管理的。

```java
private final Canvas mCanvas = new CompatibleCanvas();
```

Java中，绘图通常在一个Canval对象上进行的，Surface中也包含了一个Canvas对象，这里的CompatibleCanvas是Surface.java中的一个内部类
CompatibleCanvas的内部类，这个内部类的作用是为了能够兼容Android各个分辨率的屏幕，根据不同屏幕的分辨率处理不同的图像数据

```java
private final class CompatibleCanvas extends Canvas {
    private Martrix mOrigMatrix = null;

    @Override
    public void setMatrix(Matrix matrix) {
        if (mCompatibleMatrix == null || mOrigMatirx == null || mOrigMatrix.equals(matrix)) {
            super.setMatrix(matrix);
        } else {
            Matrix m = new Matrix(mCompatibleMatrix);
            m.preConcat(matrix);
            super.setMatrix(m);
        }
    }

    @Override
    public void getMatrix(Matrix m) {
        super.getMatrix(m);
        if (mOrigMatrix == null) {
            mOrigMatrix = new Matrix();
        }
        mOrigMatrix.set(m);
    }
}
```
