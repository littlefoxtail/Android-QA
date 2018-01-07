`View`、`Window`以及`Activity`主要用于显示并与用户交互的

# View
这个类表示用户界面组件的基本构建块。一个View在屏幕上占据一个矩形区域，负责绘制和事件处理。
View是widgets的基类，它们是用于创交互式UI组件，
View的子类(ViewGroup)是layout的的基类，它是包含其他Views(或者其他ViewGroups)并定义layout properties

# Window
顶级窗口外观和行为策略的抽象基类。
an instance of this class should be userd as the top-level view added to window manager。
它提供了标准的UI策略，如背景，标题，区域，默认密钥处理。

这个抽象类的唯一实现是android.view.PhoneWindow。


> Window负责管理View， View是Window里面用于交互的Ui元素。Window只attach一个View Tree，当Window需要重新绘制(如，当View调用invalidate)
最终转为window的surface, surface被锁住并返回Canvas对象，此时View拿到Canvas对象来绘制自己。当所有View绘制完成后，Surface解锁，并且post到绘制缓存用于绘制，
通过Surface Flinger来组织各个Window，显示最终的整个屏幕。

# Activity 
an activity is single, focused thing that the user can do.
几乎全部activities与用户交互，所以activity类负责处理创建一个窗口。可以在其中放置UI。

# WindowManager
The interface that app use to talk to the window manager
每个window manager实例绑定到一个特定的(Display)