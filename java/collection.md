
# HashMap
它根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。 HashMap最多只允许一条记录的键为null，允许多条记录的值为null。HashMap非线程安全，即任一时刻可以有多个线程同时写HashMap，可能会导致数据的不一致。如果需要满足线程安全，可以用 Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。
1. 默认容量 1<<4 = 16
2. 最大容量 1<<30 = 2^32
3. 默认加载因子 0.75f，当容量大于(容量x加载因子)，会进行扩容操作。
> 备注：异或的操作如下：0 ^ 0=0，1 ^ 1 =0，0 ^ 1=1，1 ^ 0=1，也就是相同时返回0，不同时返回1。


# TreeSet和TreeMap的关系
TreeMap是Map接口的常用实现类，而TreeSet是Set接口的常用实现类。TreeSet底层是通过TreeMap来实现，而TreeMap的实现就是红黑树算法。

相同点：
1. TreeMap和TreeSet都是有序的集合
2. TreeMap和TreeSet都是非同步集合
3. 运行速度都要比HashMap集合慢，他们内部对元素的操作实现复杂度为O(logN)。

不同点：
1. 最主要区别是TreeSet和TreeMap分别实现了不同接口
TreeSet只存储一个对象，而TreeMap存储两个对象Key和Value（仅仅key对象有序）
TreeSet中不能有重复对象，而TreeMap中可以存在