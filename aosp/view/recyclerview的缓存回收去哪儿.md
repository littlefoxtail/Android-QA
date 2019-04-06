# 回收入口

![回收入口](/img/回收入口.png)

```java
public class LinearLayoutManager {
    private void recycleChildren(RecyclerView.Recycler recycler, int startIndex, int endIndex) {
        if (endIndex > startIndex) {
            for (int i = endIndex - 1; i >= startIndex; i--) {
                removeAndRecycleViewAt(i, recycler);
            }
        } else {
            for (int i = startIndex; i > endIndex; i--) {
                removeAndRecycleViewAt(i, recycler);
            }
        }
    }
}
```

```java
public class RecyclerView {
    class LayoutManager {
        public void removeAndRecycleViewAt(int index, Recycler recycler) {
            remoteViewAt(index);
            recycler.recycleView(view);
        }
    }

    class Recycler {
        public void recycleViewHolderInternal(ViewHolder holder) {
            if ((forceRecycle || holder.isRecycable())) {
                //先存在mCachedViews里面
                //这里的判断条件决定了复用mViewCacheMax中的ViewHolder时不需要重新绑定数据
                if (mViewCacheMax > 0
                        && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                        | ViewHolder.FLAG_REMOVED
                        | ViewHolder.FLAG_UPDATE
                        | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
                    // Retire oldest cached view
                    if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                        //mCachedViews大小有限制，默认只能存2个ViewHolder，当第三个ViewHolder存入会将第一个移除掉
                        recycleCachedViewAt(0);
                    }
                    mCachedViews.add(targetCacheIndex, holder)
                }
                if (!cached) {
                    addViewHolderToRecyclerdViewPool(holder, true);
                    recycled = true;
                }
            }
        }
    }
}
```

从mCachedViews移除掉的ViewHolder会加入回收池中。mCachedViews有点像“回收预备队列”，即总是先回收到mCachedViews，当它放不下的时候，按照先进先出原则将最先进入的ViewHolder存入回收池

```java
public class RecyclerView {
    class Recycler {
        void addViewHolderToRecyclerdViewPool(..) {
            getRecyclerViewPool().putRecycledView(holder);
        }
    }
}
```

## 总结

- 滑出屏幕表项对应的ViewHolder会被回收到mCachedViews+mRecyclerPool结构中，mCachedViews是ArrayList，默认存储最多2个ViewHolder，当它放不下的时候，按照先进先出原则将先进入的ViewHolder存入回收池的方式腾出空间。mRecyclerPool是SparseArray，它会按viewType分类存储ViewHolder，默认每种类型最多存5个
- 从mRecyclerPool中复用的ViewHolder需要重新绑定数据
- 从mCachedViews中复用的ViewHolder不需要重新绑定数据