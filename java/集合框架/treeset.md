# TreeSet和TreeMap的关系

TreeMap是Map接口的常用实现类，而TreeSet是Set接口的常用实现类。TreeSet底层是通过TreeMap来实现，而TreeMap的实现就是红黑树算法。

相同点：

1. TreeMap和TreeSet都是有序的集合
2. TreeMap和TreeSet都是非同步集合
3. 运行速度都要比HashMap集合慢，他们内部对元素的操作实现复杂度为O(logN)。

不同点：

1. 最主要区别是TreeSet和TreeMap分别实现了不同接口，TreeSet只存储一个对象，而TreeMap存储两个对象Key和Value（仅仅key对象有序，TreeSet中不能有重复对象，而TreeMap中可以存在
