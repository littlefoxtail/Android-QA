# setMaxLifecycle

设置流程：

```java
// FragmentTransaction
public FragmentTransaction setMaxLifecycle(Fragment fragment, Lifecycle.State state) {
    addOp(new Op(OP_SET_MAX_LIFECYCLE, fragment, state));
    return this;
}

//BackStackRecord # executeOps
case OP_SET_MAX_LIFECYCLE:
    mManager.setMaxLifecycle(f, op.mCurrentMaxState);
    break;

// FragmentManager
void setMaxLifecycle(Fragment f, Lifecycle.State state) {
    f.MaxState = state;
}
```

正常执行流程：

```java
// FragmentManager
void moveToState(Fragment f, int newState) {
    newState = Math.min(newState, fragmentStateManager.computeExpectedState());
}
```

```java
// FragmentStateManager
public computeExpectedState() {
        // Don't allow the Fragment to go above its max lifecycle state
    switch (mFragment.mMaxState) {
        case RESUMED:
            // maxState can't go any higher than RESUMED, so there's nothing to do here
            break;
        case STARTED:
            maxState = Math.min(maxState, Fragment.STARTED);
            break;
        case CREATED:
            maxState = Math.min(maxState, Fragment.CREATED);
            break;
        default:
            maxState = Math.min(maxState, Fragment.INITIALIZING);
    }
    return maxState;
}
```


