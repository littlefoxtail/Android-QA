# Fragment事务管理

Fragment事务管理都是通过`FragmentTransaction`进行事务管理的。一个事务开始于第一次执行操作语句，结束于commit。就是把多个操作缓存起来，等调用commit的时候，统一批处理

## FragmentTransaction#add

```java
void doAddOp() {
    //添加操作
    addOp(new Op(opccmd, fragment));
}

void addOp(Op op){
    mOps.add(op);
    op.mEnterAnim = mEnterAnim;
    op.mExitAnim = mExitAnim;
    op.mPopEnterAnim = mPopEnterAnim;
    op.mPopExitAnim = mPopExitAnim;
}
```

## FragmentTransaction#commit

```java
// BackStackRecord
public int commit() {
    return commitInternal(false);
}
// BackStackRecord
int commitInternal(boolean allowStateLoss) {
    mCommitted = true;
    // 只有调用了addToBackStack方法之后，这个标记才会为true
    if (mAddToBackStack) {
        mIndex = mManager.allowBackStackIndex();
    } else {
        mIndex = -1;
    }
    // 插入事务队列
    mManager.enqueueAction(this, allowStateLoss);
    return mIndex;
}
```

```java
// FragmentManager
void enqueuAction(OpGenerator action, boolean allowStateLoss) {
    if (!allowStateLoss) {
        //状态丢失的异常检查
        checkStateLoss();
    }

    synchronized (mPendingActions) {
        if (mHost == null) {
            if (allowStateLoss) {
                return;
            }
        }
        // 加入待定任务队列中，mPendingActions是ArrayList
        mPendingActions.add(action);
        scheduleCommit();
    }
}

public boolean isStateSaved() {
    // mStateSaved：fragment保存、恢复过程--fragment保存标识
    // 由Activity.onSaveInstanceState方法最终会调到FragmentManagerImpl.saveAllState方法中设置mStateSaved为true
    // mStopped：fragment onStop周期的分发方法FragmentManagerImpl.dispatchStop()中设置为true，此时Activity的周期也是onStop
    return mStateSaved || mStopped;
}


// FragmentManager
void scheduleCommit() {
    synchronized (mPendingActions) {
        boolean postponeReady =
                    mPostponedTransactions != null && !mPostponedTransactions.isEmpty();
        boolean pendingReady = mPendingActions.size() == 1;
        if (postponeReady || pendingReady) {
            mHost.getHandler().removeCallbacks(mExecCommit);
            mHost.getHandler().post(mExecCommit);
            updateOnBackPressedCallbackEnabled();
        }
    }
}
```

## ExecCommit

```java
boolean execPendingActions(boolean allowStateLoss) {
    ensureExecReady(allowStateLoss);

    boolean didSomething = false;
    //mTmpRecords：临时存储所有待执行的动作（mPendingActions）生成的BackStackRecord
    // mTmpIsPop：存储BackStackRecord是否为出栈
    while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
        mExecutingActions = true;
        try {
            removeRedundantOperationsAndExe
        } finally {
            cleanupExec();
        }
        didSomething = true;
    }

}

/**
 * 遍历mPendingActions调用OpGenerator.generateOps()方法生成 BackStackRecord 添加到 mTmpRecords 并把是否为出栈添加到 mTmpIsPop 中
 */
private boolean generateOpsForPendingActions(ArrayList<BackStackRecord> records, ArrayList<Boolean> isPop) {
    boolean didSomething = false;
    synchronized (mPendingActions) {
        if (mPendingActions.isEmpty()) {
            return false;
        }
        final int numActions = mPendingActions.size();
        for (int i = 0; i < numActions; i++) {
            didSomething |= mPendingActions.get(i).generateOps(records, isPop);
        }
        mPendingActions.clear();
        mHost.getHandler().removeCallbacks(mExecCommit);
        return doSomething;
    }
}
```

generateOps()实现1：

```java
//BackStackRecord
public boolean generateOps(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop) {
    records.add(this);
    isRecord.add(false);
    if (mAddToBackStack) {
        //添加到“回退栈”中
        mManager.addBackStackState(this);
    }
    return true;
}
```

generateOps()实现2：

```java
// PopBackStackState
public boolean generateOps(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop) {
    return popBackStackState(records,  isRecordPop, mName, mId, mFlags);
}


// FragmentManager
boolean popBackStackState(ArrayList<BackStackRecord> records, ArrayList<Boolean > isRecordPop, String name, int id, int flags) {
    ..
    isRecordPop.add(true);
    ..
}
```

```java
//FragmentManager
void addBackStackState(BackStackRecord state) {
    if (mBackStack == null) {
        mBackStack = new ArrayList<>();
    }
    //"回退栈" 就是mBackStack
    mBackStack.add(state);
}

/**
 * 该方法实际上是对records集合中所有动作的
 * startIndex（起始动作位置）
 * recordNum（需要操作的动作个数）的设置
 * 都会去调用executeOpsTogether方法
 */
private void removeRedundantOperationsAndExecute(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop) {
    executePostponedTransaction(records, isRecordPop);
    for (int recordNum = 0; recordNum < numRecords; recordNum++) {
        final boolean canReorder = records.get(recordNum).mReoderingAllowed;
        if (!canReorder) {
            if (startIndex != recordNum) {
                executeOpsTogether(records, isRecordPop, startIndex, recordNum);
            }
            int reorderingEnd = recordNum + 1;
            // 只有“回退栈”执行出栈才会执行此处代码
            if (isRecordPop.get(recordNum)) {
                while(reorderingEnd < numRecords
                        && isRecordPop.get(reorderingEnd)
                        && !records.get(reorderingEnd).mReorderingAllowed) {
                    reorderingEnd++;
                }
            }
            executeOpsTogether(records, isRecordPop, recordNum, reorderingEnd);
            startIndex = reorderingEnd;
            recordNum = reorderingEnd - 1;
        }
    }
    if (startIndex != numRecords) {
        executeOpsTogether(records, isRecordPop, startIndex, numRecords);
    }
}

private void executeOpsTogether(..) {
    for (int recordNum = startIndex; recordNum < endIndex; recordNum++) {
        final BackStackRecord record = records.get(recordNum);
        final boolean isPop = isRecordPop.get(recordNum);
        if (!isPop) {
            // 28之前这里会执行 把replace替换成remove、add
            oldPrimaryNav = record.expandOps(mTmpAddedFragments, oldPrimaryNav);
        } else {
            oldPrimaryNav = record.trackAddedFragmentsInPop(mTmpAddedFragments, oldPrimaryNav);
        }
        addToBackStack = addToBackStack || record.mAddToBackStack;
    }
    mTmpAddedFragments.clear();

    executeOps(records, isRecordPop, startIndex, endIndex);
}
```

```java
// FragmentManager
private static void executeOps(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
    for (int i = startIndex; i < endIndex; i++) {
        final BackStackRecord record = records.get(i);
        final boolean isPop = isRecordPop.get(i);
        if (isPop) {
            // 若为回退栈出栈操作，则执行此方法
            // 此方法中根据op.cmd判断对fragment进行相应的处理
            record.bumpBackStackNesting(-1);
            boolean moveToState = i = (endIndex - 1);
            record.executePopOps(moveToState);
        } else {
            record.bumpBackStackNesting(1);
            record.executeOps();
        }
    }
}

// BackStackRecord
/**
 * 通过for循环对mOps进行了遍历，而此次遍历对本地commit提交的所有操作进行设置
 * mAdded：包含了所有已经added并且没有被从Activity中removed和detached的Fragments
 * mActive：mAdded的一个超集，是绑定到一个Activity上的所有Fragment。包含返回栈中所有的通过任何FragmentTransaction添加的Fragments
 *  *当一个Activity要保存它的State时，它必须保存它所有的Fragment状态
 */
void executePopOps(boolean moveToState) {
    // 遍历执行所有的mOps
    for (int opNum = mOps.size() - 1; opNum >= 0; opNum--) {
        switch (op.mCmd) {
            case OP_ADD:
                f.setNextAnim(op.mPopExitAnim);
                mManager.setExitAnimationOrder(f, true);
                mManager.removeFragment(f);
                break
        }

        if (!mRecorderingAllowed && op.mCmd != OP_REMOVE && f != null) {
            mManager.moveFragmentToExpectedState(f);
        }
    }

    // 只有没设置setReorderingAllowed(true)的才能继续，
    // 而设置的会在前面的某步逻辑当中走到moveToState方法内，上面有说明
    if (!mReorderingAllowed && moveToState) {
        mManager.moveToState(mManager.mCurState, true);
    }
}
```

### 提交add操作到当前fragment添加

```java
// FragmentManager
void addFragment(Fragment fragment) {
    FragmentStateManager fragmentStateManager = createOrGetFragmentStateManager(fragment);
    mFragmentStore.makeActive(fragmentStateManager);
    if (!fragment.mDetached) {
        mFragmentStore.addFragment(fragment);
        fragment.mRemoving = false;
        if (fragment.mView = null) {
            fragment.mHiddenChanged = false;
        }
    }
}

// FragmentStore
void addFragment(Fragment fragment) {
    synchronized (mAdded) {
        mAdded.add(fragment);
    }
    fragment.mAdded = true;
}
```

## 改变fragment manager的状态

```java
// FragmentManager
void moveToState(int newState, boolean alway) {
    // 遍历mAdded集合
    for (Fragment f : mFragmentStore.getFragments()) {
        // 将fragment移至于预期状态
        moveFragmentToExpectedState(f);
    }
    // 遍历mActive集合
    for (Fragment f : mFragmentStore.getActiveFragments()) {
        if (f != null && !f.mIsNewlyAdded) {
            moveFragmentToExpectedState(f);
        }
    }
}

// FragmentManager
void moveFragmentToExpectedState(Fragment f) {
    moveToState(f);
}

void moveToState(Fragment f, int newState) {
    //根据fragment的mState自身状态和newState传过来的状态值进行比较来区分
    
}
```
