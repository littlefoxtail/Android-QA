背后有一个树结构。
不允许 null 元素。尝试 null 元素添加到 TreeSet 中会导致`NullPointerException`。
底层使用的是红黑树。
使用红黑树作为底层数据结构的优势在于，它可以保证`TreeSet`中的元素是按照顺序排列的。
# TreeSet和TreeMap的关系

TreeMap是Map接口的常用实现类，而TreeSet是Set接口的常用实现类。TreeSet底层是通过TreeMap来实现，而TreeMap的实现就是红黑树算法。

相同点：

1. TreeMap和TreeSet都是有序的集合
2. TreeMap和TreeSet都是非同步集合
3. 运行速度都要比HashMap集合慢，他们内部对元素的操作实现复杂度为O(logN)。

不同点：

1. 最主要区别是TreeSet和TreeMap分别实现了不同接口，TreeSet只存储一个对象，而TreeMap存储两个对象Key和Value（仅仅key对象有序，TreeSet中不能有重复对象，而TreeMap中可以存在
# 适用场景
- 需要保存一组元素，并且希望这些元素按照顺序排列。
- 需要频繁地进行查询、插入和删除操作，并且希望这些操作的时间复杂度都是较低的。
