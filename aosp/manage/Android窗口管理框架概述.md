![概述](/img/android_ui_system.png)

- UI框架层：负责管理窗口中View组件的布局与绘制以及响应用户输入事件
- WindowManagerService层：负责管理窗口Surface的布局和文字
- SurfaceFinger层：将WindowManagerService管理的窗口按照一定的次序显示在屏幕上

- Activity：应用视图的容器。
- Window：应用窗口的抽象表示，它的实际表现是View。
- View：实际显示的应用视图。
- WindowManagerService：用来创建、管理和销毁Window。