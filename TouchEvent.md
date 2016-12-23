# Managing Touch Events in a ViewGroup
> Managing Touch Events in a ViewGroup

```
├── View
│   ├── ViewGroup

├── View
│   ├── dispatchTouchEvent()
│   ├── onTouchEvent() 

├── ViewGroup
│   ├── this.dispatchTouchEvent()
│   ├── onInterceptTouchEvent()
│   ├── super.onTouchEvent()
```

关于`ViewGroup`的触摸事件，要能正确处理Touch事件。必须重写`onInterceptTouchEvent`方法。

## Intercept Touch Events in a ViewGroup
当`ViewGroup`检测到有事件发生时，`onInterceptToucheEvent()`将会被调用。包含`ViewGroup`和`View`。如果`onInterceptTouchEvent()`返回true，表示`MotionEvent`已经被拦截。他将不会下发到他的子视图。而是到当前`ViewGroup`的`onTouchEvent()`。

`onInterceptTouchEvent()`会拿到子视图的`MotionEvent`。如果`onInterceptTouchEvent()`返回true，发给