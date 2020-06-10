# 按键事件

## 时序图

![按键事件的时序图](/img/按键事件的时序图.png)

## processKeyEvent方法的具体实现

```java
// ViewPostImeInputStage
private int processKeyEvent() {
    // 这个mView是Activity的顶层容器
    // 所以这个是ViewGroup的dispatchKeyEvent
    if (mView.dispatchKeyEvent(event)) {
        return FINISH_HANDLED;
    }

    //是否终止事件
    // 属于保护措施
    if (shouldDropInputEvent(q)) {
        return FINISH_NOT_HANDLED;
    }

    //应用fallback策略
    // 具体实现见PhoneFallbackEventHandler中dispatchKeyEvent()方法
    // 主要是对媒体键，音量键，通话键做处理
    if (mFallbackEventHandler.dispatchKeyEvent(event)) {
        return FINISH_HANDLED;
    }

    // 自动追踪焦点
    // 该部分是重点
    if (event.getAction() == KeyEvent.ACTION_DOWN) {
        if (groupNavigationDirection != 0) {
            //如果是TAB键则groupNavigationDirection不为0，进行如下操作（这里不做重点解析）
            if (performKeyboardGroupNavigation(groupNavigationDirection)) {
                return FINISH_HANDLED;
            }
        } else {
            //对按键处理的焦点
            if (preformFocusNavigation(event)) {
                return FINISH_HANDLED;
            }
        }
    }
}

```

## performFocusNavigation

```java
//ViewPostImeInputStage
private boolean performFocusNavigation(KeyEvent event) {
    if (direction != 0) {
        //找到当前聚焦的View
        View focused = mView.findFocus();
        if (focused != null) {
            //如果focused不为空，说明找到了焦点，接着focusSearch会把
            // direction（遥控器按键的方向）作为参照，找到特定方向下一个将要获取焦点的View
            View v = focused.focusSearch(direction);
            if (v != null && v != focused) {
                focused.getFocuseRect(mTempRect);
                ...
                if ()
            }


        }
    }
}
```

## findFocus的具体

```java
//ViewGroup焦点判断
public View findFocus() {
    if (isFocused()) {
        return this;
    }
    if (mFocused != null) {
        // 否则通过mFocused 其实就是ViewGroup中获取焦点的子View
        return mFocused.findFocus();
    }
    return null;
}

// View焦点判断
public View findFocus() {
    return (mPrivateFlags & PFLAG_FOCUSED) != 0 ? this : null;
}

```

## focusSearch方法的具体实现

```java
// View
public View focusSearch(int direction) {
    if (mParent != null) {
        return mParent.focusSearch(this, direction);
    } else {
        return null;
    }
}

// ViewGroup
public View focusSearch(View focused, int direction) {
    if (isRootNamespace()) {
        //判断是否是顶层view
        return FocusFinder.getInstance().findNextFocus(this, focused, direction);
    } else if (mParent != null) {
        return mParent.focusSearch(focused, direction);
    }
    return null;
}
```

## findNextFocus

```java
// FocusFinder
public final View findNextFocus(..) {
    return findNextFocus(..);
}

//focused是当前焦点视图
private View findNextFocus(ViewGroup root, View focused, Rect focusedRect, int direction) {
    View next = null;
    ViewGroup effectiveRoot = getEffectiveRoot(root, focused);
    if (focused != null) {
        // 优先从xml或者代码中指定focusid的View中找
        next = findNextUserSpecifiedFocus(effectiveRoot, focused, direcation);
    }
    if (next != null) {
        return next;
    }
    ArrayList<Veiw> focusables = mTempList;
    try {
        focusables.clear();
        effectiveRoot.addFocusables(focusables, direction);
        if (!focusables.isEmpty()) {
            // 其次 ，根据算法去找，原理就是找到方向上最近的View
            next = findNextFocus(effectiveRoot, focused, focusedRect, direction, focusables);
        }
    }

}
```

### findNextUserSpecifiedFocus从指定focusid的View中找

```java
// FocusFinder
private View findNextUserSpecifiedFocus(..) {
    View userSetNextFocus = focused.findUserSetNextFocus(root, direction);
    View cycleCheck = userSetNextFocus;
}

```

`findNextUserSpecifiedFocus()`方法会执行focused（即当前获取焦点的View）的findUserSetNextFocus方法

### findNextFocus根据算法去找

1. 遍历找出所有isFocusable的视图
2. 将focused视图的坐标系，转换到root的坐标系中，统一坐标，以便进行下一步的计算
3. 进行一次遍历比较，得到最“近”的视图作为下一个焦点视图