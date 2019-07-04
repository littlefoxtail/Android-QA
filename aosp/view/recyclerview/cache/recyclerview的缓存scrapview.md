# scrap view

## 填充内容

![添加到scrap集合](/img/添加到scrap集合.png)

- LayoutManager的孩子，都是屏幕上可见的
- 填充到列表之前会将其先回收到`mAttachedScrap`中
- mAttachedScrap用于屏幕中可见表项的回收和复用
  
## 清空内容

![scrap清空内容](/img/scrap清空内容.png)

## 总结

1. Recycler有4个层次用于缓存ViewHolder对象，优先级从高到底依次为ArrayList<ViewHolder> mAttachedScrap、ArrayList<ViewHolder> mCachedViews、ViewCacheExtension mViewCacheExtension、RecycledViewPool mRecyclerPool。如果四层缓存都未命中，则重新创建并绑定ViewHolder对象

2. 缓存性能：
    |缓存|重新创建ViewHolder|重新绑定数据|
    |:--:|:--:|:--:|
    |mAttchedScrap|false|false|
    |mCachedView|false|false|
    |mRecyclerPool|false|true|

3. 缓存容量
    - mAttachedScrap：没有大小限制，但最多包含屏幕可见表项
    - mCachedViews：默认大小限制为2，放不下时，按照先进先出的原则将最先进入的ViewHolder存入回收池以腾出空间
    - mRecyclerPool：对ViewHolder按viewType分类存储（通过SparseArray），同类ViewHolder存储在默认大小为5的ArrayList中
4. 缓存用途
    - mAttachedScrap：用于布局过程中屏幕可见表项的回收和复用。
    - mCachedViews：用于移出屏幕表项的回收和复用，且只能用于指定位置的表项，有点像“回收池预备队列”，即总是先回收到mCachedViews，当它放不下的时候，按照先进先出原则将最先进入的ViewHolder存入回收池。
    - mRecyclerPool：用于移出屏幕表项的回收和复用，且只能用于指定viewType的表项
5. 缓存结构
    - mAttachedScrap：`ArrayList<ViewHolder>`
    - mCachedViews：`ArrayList<ViewHolder>`
    - mRecyclerPool：对ViewHolder按viewType分类存储在`SparseArray<ScrapData>`中，同类ViewHolder存储在ScrapData中的ArrayList中
