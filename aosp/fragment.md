# Android fragment源码完全解析

1. Fragment是Android3.0引入的API，号称为了解决屏幕碎片化和帮助重用代码的构造

2. FragmentTransaction它封装了一系列对fragment的操作，并一次性执行这些操作
    add/remove/replace/show/hide等等操作

3. 什么是FragmentManager
    它是和某个act相关联的，并不是全局唯一的，而是每个act都有一个自己的FragmentManager，内部有自己状态的mCurState，对应外部的act的生命周期状态。它提供和act中fragment交互的API

[事务管理](Fragment事务管理.md)
[setMaxLifecycle](fragment设置最大生命周期.md)

## Activity中的FragmentManager

Activity中有个FragmentController实例，那么它是在何时初始化呢？

```java
final FragmentController mFragments = FragmentController.createController(new HostCallbcks)
```


