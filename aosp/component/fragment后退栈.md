# Fragment返回栈

FragmentTransaction，它是一个事务模型，事务可以回滚到之前状态。

`FragmentTransaction`的实现类为`BackStackRecord`，所以fragment的返回栈其实存放的就是`BackStackRecord`作为返回栈元素

## popBackStack

```java
// BackStackRecord
public void popBackStack(..) {
    enqueueAction(new PopBackStackState(..), false);
}
```

```java
// FragmentManager
private class PopBackStackState implements OpGenerator {
    public boolean generateOps(..) {
        return popBackStackState(..);
    }
}

boolean popBackStackState(..) {
    // 将返回栈栈顶移除
    int last = mBackStack.size() - 1;
    records.add(mBackStack.remove(last));
    isRecordPop.add(true);

    // 如果传入的name或者id有值，且flag为0
    // 则找到返回栈（倒序遍历）中第一个符合 name 或 id 的位置
    if (name != null || id >= 0) {
        BackStackRecord bss = mBackStack.get(index);
        if (name != null && name.equals(bss.getName())) {
            break;
        }
        if (id >= 0 && id == bss.mIndex) {
            break;
        }
        index--;
    }

    if ((flag & POP_BACK_STACK_INCLUSIVE) != 0) {
        // 如果传入的 name 或 id 有值，且 flag 为 POP_BACK_STACK_INCLUSIVE，则在上一条获取位置的基础上继续遍历，直至栈底或者遇到不匹配的跳出循环，接着出栈所有 BackStackRecord
        index--;
        while(index >= 0) {
            BackStackRecord bss = mBackStack.get(index);
            if ((name != null && name.equals(bss.getName()))
                    || (id >= 0 && id == bss.mIndex)) {
                index--;
                continue;
            }
            break;
        }
    }

// 将该位置上方的所有 BackStackRecord 并添加到 record list 中，同时 isRecordPop list 全部传入 true
    for (int i = mBackstack.size() - 1; i > index; i--) {
        records.add(mBackStack.remove(i));
        isRecordPop.add(true);
    }
}
```


```java
//FragmentManager
private static void executeOps(..) {
    for (int i = startIndex; i < endIndex; i++) {
        final BackStackRecord record = records.get(i);
        final boolean isPop = isRecordPop.get(i);
        if (isPop) {
            boolean moveToState = i == (endIndex - 1);
            record.executePopOps(moveToState);
        }
    }
}

// BackStackRecord
// 根据不同mCmd执行相应的逆操作
void executePopOps(boolean moveToState) {

}
```

## fragment怎么拦截activity的返回逻辑

```java
// ComponentActivity
public void onBackPressed() {
    mOnBackPressedDispatcher.onBackPressed();
}

// FragmentManager
void handleOnBackPressed() {
    execPendingActions(true);
    // 如果开启事务调用了addToBackStack方法，则mBackPressedCallback的isEnabled属性会赋值为true
    if (mOnBackPressedCallback.isEnableed()) {
        popBackStackImmediate();
    } else {
        mOnBackPressedDispatcher.onBackPressed();
    }
}
```