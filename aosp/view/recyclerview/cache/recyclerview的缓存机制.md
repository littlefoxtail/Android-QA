# RecyclerView条目缓存机制

RecyclerView缓存基本上是通过三个内部类管理的，Recycler、RecycledViewPool和ViewCacheExtension

- Recycler:一个Recycler是负责管理称为碎片的视图或者已经detached的视图，从而实现View的复用。
- RecycledViewPool:可以让你在多个RecyclerView之间分享视图

|变量|作用|
|:---:|:---:|
|mChangedScrap|mRecyclerView分离的ViewHolder列表|
|mAttachedScrap|未与RecyclerView分离的ViewHolder列表|
|mCachedViews|ViewHolder缓存列表|
|mViewCacheExtension|开发者可以控制的ViewHolder缓存的帮助类|
|mRecyclerPool|ViewHolder缓存池|

RecycledViewPool:
> RecycledViewPool类是用来缓存Item用的，是一个ViewHolder的缓存池，如果多个RecyclerView之间用
> `setRecycledViewPool(RecycleViewPool)`设置同一个RecycledViewPool,他们就可以共享Item。其实
> RecyclerViewPool的内部维护了一个Map，里面以不同的viewType为key存储了各自对应的ViewHolder集合，可以通过
> 提供的方法来修改内部缓存的viewHolder。

ViewCacheExtension:
> 这是一个需要开发者重写的类。在`Recycler.getScrapOrHiddenOrCachedHolderForPosition`方法获取View
> Recycler先检查自己内部的`attached scrap`和一级缓存，再检查
> `ViewCacheExtensien`最后检查`RecyclerViewPool`，从上面三个任何一个只要拿到View就不会调用下一个方法。

[recyclerview的缓存如何复用](recyclerview的缓存如何复用.md)
[recyclerview的缓存回收了啥](recyclerview的缓存回收些了啥.md)
[recyclerview的缓存回收去哪儿](recyclerview的缓存回收去哪儿.md)
[recyclerview的缓存scrapview](recyclerview的缓存scrapview.md)
