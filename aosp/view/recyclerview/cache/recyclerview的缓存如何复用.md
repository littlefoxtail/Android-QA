# 咋复用

Recycler负责管理(scrapped)报废或(detached)分离的项目视图以供重复使用。在填充itemView的时候，itemView是从它获取的；
滑出屏幕的itemView是由它回收的

scrapped视图仍然附加到其父级RecyclerView，但是已被标记为删除或重复使用

## 复用机制

```java
public class RecyclerView {
    class Recycler {
        View getViewForPosition(int position, boolean dryRun) {
            return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
        }
        //这个方法是复用机制的入口，外部调用这个方法就可以返回想要的View
        /*
         * 1.mChangedScrap
         * 2.mAttachedScrap
         * 3.mCachedView
         * 4.mViewCacheExtension
         * 5.mRecyclerPool
         * 6.mAdapter创建一个ViewHolder
         */
        ViewHolder tryGetViewHolderForPositionByDeadline(int position,
            boolean dryRun, long deadlineNs) {
                //1.
                if (mState.isPreLayout()) {
                    holder = getChangedScrapViewForPosition(position);
                    fromScrapOrHiddenOrCache = holder != null;
                }
                //2.
                if (holder == null) {
                    holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
                    if (holder != null) {

                    }
                }
                //3.
                if (holder == null) {
                    holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                        type, dryRun);
                    //4.
                    if (holder == null && mViewCacheExtension != null) {
                        final View view = mViewCacheExtension.
                            getViewForPositionAndType(this, position, type);
                    }
                    //5
                    if (holder == null) {
                        holder = getRecyclerViewPool().getRecyclerView(type);
                    }
                    //6.所有缓存都没有命中，只能创建ViewHolder
                    if (holder == null) {
                        holder = mAdapter.createViewHolder(RecyclerView.this, type);
                    }
                }
                if () {

                } else if (...) {
                    bound = tryBindViewHolderByDeadline(...);
                }
        }

        ViewHolder getScrapOrHiddenOrCacheHolderForPosition(int position, boolean dryRun) {
            //1.首先去mAttachedScrap中寻找满足position一致的viewHolder，需要匹配一些条件。
            for i in mAttachedScrap.size():
                 final ViewHolder holder = mAttachedScrap.get(i);
            //2.移除屏幕中的视图中搜索ViewHolder，找到之后将他存入scrap回收集合中
            //3.在缓存中搜索ViewHolder
            for i in mCachedViews.size():
                final ViewHolder holder = mCacheViews.get(i)
        }
    }

}
```

- 初步结论：RecyclerView回收机制中，回收复用的对象是ViewHolder，且以ArrayList为结构存储在RecyclerView对象中

```java
public class RecyclerViewPool {
    static class ScrapData {
        ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
        int mMaxScrap = DEFAULT_MAX_SCRAP;
    }
    //回收池中存放所有类型ViewHolder的容器
    SparseArray<ScrapData> mScrap = new SparseArray<>();

}
```

## 缓存优先级

- 为了获取`ViewHolder`做了5次尝试，共从5个地方获取，排除3中特殊情况，即从`mChangedScrap`获取、通过id获取、
从自定义缓存获取，正常流程中只剩下3种获取方式：
    1. 从mAttachdScrap获取
    2. 从mCachedViews获取
    3. 从mRecyclerPool获取
- 这样的缓存优先级，复用性能越好意味着所做的昂贵操作越少
    1. 最坏的情况：重新创建ViewHolder并重新绑定数据
    2. 次好的情况：复用ViewHolder但重新绑定数据
    3. 最好的情况：复用ViewHolder且不重新绑定数据

小总结：

- RecycleView中，并不是每次绘制表现，都会重新创建ViewHolder对象，也不是每次都会重新绑定ViewHolder数据
- RecyclerView通过Recycler获取下一个带绘制表项
- Recycler有四个层次用于缓存ViewHolder对象，优先级从高到低依次为`ArrayList<ViewHolder> mAttachedScrap`、`ArrayList<ViewHolder> mCachedViews`、ViewCacheExtension mViewCacheExtension、RecycledViewPool mRecyclerPool。如果四层缓存都未命中，则重新创建并绑定ViewHolder对象
- RecyclerViewPool对ViewHolder按viewType分类存储，同类ViewHolder存储在默认大小为5的ArrayList中
- mRecyclerPool中复用的ViewHolder需要重新绑定数据，从mAttachedScrap中复用的ViewHolder不需要重新创建也不需要重新绑定数据
- 从mRecyclerPool中复用的ViewHolder，只能复用于viewType相同的表项，从mCachedViews中复用的ViewHolder，只能复用于指定位置的表项

