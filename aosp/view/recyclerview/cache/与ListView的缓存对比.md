# 与ListView对比

## 缓存机制对比

### 层级不同

RecyclerView比ListView多两级缓存，支持多个ItemView缓存，支持开发者自定义缓存处理逻辑，支持所有RecyclerView共用同一个RecyclerViewPool

具体来说：

ListView(两级缓存)

|   ListView   | 是否需要回调createView | 是否需要回调bindView |生命周期|备注|
| :----------: | :--------------: | :------------: | :--------------------------------------: | :---------------: |
| mActiveViews |        否         |       否        |              onLayout函数周期内               | 用于屏幕内ItemView快速重用 |
| mScrapViews  |        否         |       是        | 与mAdapter一致，当mAdapter被更换时，mScrapViews即被清空 |                   |


RecyclerView拥有三级缓存(算上mAdapter.createViewHolder的话其实就有四级了)

|    RecyclerView     | 是否需要回调createView | 是否需要回调bindView |                   生命周期|                  备注                   |作用|
| :-----------------: | :--------------: | :------------: | :--------------------------------------: | :-----------------------------------: |:--:|
|   mAttachedScrap    |        否         |       否        |              onLayout函数周期内               |           用于屏幕内ItemView快速重用           | 未与RecyclerView分离的ViewHolder列表（一级缓存）|
|   mChangeedScrap    |                 |               |                             |                      | RecyclerView中需要改变的ViewHolder（一级缓存）|
|     mCacheViews     |        否         |       是        | 与mAdapter一致，当mAdapter被更换时，mScrapViews即被清空 |        默认上限为2，即缓存屏幕外2个ItemView        |RecyclerView的ViewHolder缓存列表（一级缓存）|
| mViewCacheExtension |                  |                |                                          |          不直接使用，需要用户在定制，默认不实现          |用户设置的RecyclerView缓存列表扩展（二级缓存）|
|    mRecyclerPool    |        否         |       是        |           与自身生命周期一致，不再被引用时即被释放           | 默认上限为5，技术上可以实现所有RecyclerViewPool共用同一个 |RecyclerView的ViewHolder缓存池（三级缓存）|

ListView与RecyclerView缓存机制基本上一致

1. mActiveViews和mAttachedScrap功能类似，意义在于快速重用屏幕上可见的列表项目itemView，而不需要重新createView和bindView
2. mScrapView和mCachedViews + mRecyclerViewPool功能相似，意义在于缓存离开屏幕的ItemView,目的是让即将进入屏幕的ItemView重用.
3. RecyclerView的优势在于：
    - mCacheViews的使用，可以做到屏幕外的列表项ItemView进入屏幕内时也无须bindView快速重用；
    - mRecyclerPool可以供多个RecyclerView共同使用，在特定场景下，如viewpager+多个列表页下有优势。

### 缓存不同

1. RecyclerView缓存RecyclerView.ViewHolder，抽象可理解为：View + ViewHolder(避免每次createView时调用findViewById) + flag(标识状态)；
2. ListView缓存View。

