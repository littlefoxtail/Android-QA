# DiffUtil在RecyclerView中的使用详解
DiffUtil的作用是比较两个数据列表并能计算出一系列将旧数据表转换成新数据表的操作。
使用Adapter.notifyDataSetChanged()有两个缺点
1. 不会触发RecyclerView动画，删除、新增、位移、change动画
2. 性能较低，极端情况下数据集一模一样，效率最低

```java
DiffUtil.DiffResult diffResult = DiffUtil.calculateDiff(new DiffCallBack(mDatas, newDatas), true);
diffResult.dispatchUpdatesTo(mAdapter);
```

它会自动计算新老数据集的差异，并根据差异情况，自动调用以下四个方法
```
adapter.notifyItemRangeInserted(position, count);
adapter.nofityItemRangeRemoved(position, count);
adapter.nofityItemMoved(fromPosition, toPosition);
adapter.nofityItemRangeChanged(position, count, payload);
```