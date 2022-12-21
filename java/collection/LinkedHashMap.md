Java中的一种数据结构，继承了HashMap的所有功能，并提供了一个链表来记录每个元素的插入顺序。

工作原理是散列表和链表。
是集成于HashMap实现的哈希链表。
1. 具备双向链表和散列表的特点。作为散列表，主要体现出O(1)时间复杂度的查询效率。当LinkedHashMap作为双向链表时，主要体现有序的特性。
2. LinkedHashMap支持FIFO和LRU两种排序模式，默认是FIFO排序模式，即按照插入顺序排序。Android中的LruCache内存缓存和DiskLruCache磁盘缓存也是直接复用LinkedHashMap提供的缓存管理能力。 

为了保持插入顺序，需要在每个节点上增加两个指针
```java
static class Entry<K,V> extends HashMap.Node<K,V> {
	Entry<K,V> before, after;
	Entry(int hash, K key, V value, Node<K,V> next) {
		super(hash, key, value, next);
	}
}
```
- before：指向此节点之前插入的节点。
- after：指向此节点之后插入的节点。
- next：指向数组表中同一桶中的下一个节点。
- hash：哈希码，用于计算该节点的索引，并检查是否相等。

# get
先在哈希表中查找，如果找到了该元素，它会现在哈希表中查找，如果找到了该元素，就会将该元素从链表中删除，然后将其插入到链表头部。这个过程称为访问顺序。

