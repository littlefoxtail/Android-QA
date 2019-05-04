# LruCache

LruCache算法就是Least Recently Used，最近最少使用算法
他的算法就是当缓存空间满了的时候，将最近最少使用的数据从缓存空间中删除以增加可用的缓存空间来缓存新内容。
LruCache内部有一个缓存列表。每当一个缓存数据被访问的时候，这个数据就会被提到列表头部，每次都这样的话，列表的尾部数据就是最近最不常使用的了，当缓存空间不足时，就会删除列表尾部的缓存数据。

1. 必填，需要提供一个缓存容量的作为构造参数。
2. 必填，覆写`sizeOf`方法，自定义设计一条数据放进来的容量计算，如果不覆写就无法预知数据的容量，不能保证缓存容量限定在最大容量以内
3. 选填，覆写 entryRemoved 方法 ，你可以知道最少使用的缓存被清除时的数据（ evicted, key, oldValue, newVaule ）。
4. LruCache是线程安全的，在内部的 get、put、remove 包括 trimToSize 都是安全的（因为都上锁了）
5. 选填，还有就是覆写 create 方法 

## LinkedHashMap

这个集合本来就有个排序功能，当第三个参数是true的时候，数据被访问的时候就会排序，这个排序的结果就是把最近访问的数据放到集合的最后面

1. LruCache的put方法调用了LinkedHashMap的put来存储数据，自己进行了对缓存空间的计算。LinkedHashMap的put方法也会进行排序
2. LruCache的get方法调用了LinkedHashMap的get来获取数据，也会触发排序

```java
public class LinkedHashMap {
    public V get(Object key) {
        Node<K, V> e;
        if ((e == getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }

    //判断目标节后的前后是否有对象，摘除出目标节点，将其放在last
    void afterNodeAccess(Node<K,V> e) {

    }
}
```

## 回到LruCache

```java
public class LruCache {
    public final V get(K key) {
        V mapValue;
        synchronized (this) {
            // LinkedHashMap 的get()方法会重新链接排序
            mapValue = map.get(key);
            if (mapValue != null) {
                //命中次数
                hitCount++;
                return mapValue;
            }
            //非命中次数
            missCount++;
        }
        /*
         *尝试创建一个值，这可能需要很长时间，并且Map可能在create()返回的值时有所不
         *同。如果在create()执行的时
         *候，一个冲突的值被添加到Map，我们在Map中删除这个值，释放被创造的值。
         */
        V createValue = create(key);
        if (createValue == null) {
            return null;
        }
        /***************************
        * 不覆写create方法走不到下面 *
        ***************************/

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);
            if (mapValue != null) {
                //有冲突的话, mapValue取得是上一次的值
                map.put(key, mapValue);
            } else {
                //没有冲突的话 size++
                size += safeSizeOf(key, createValue);
            }
        }
        if (mapValue ! = null) {
            entryRemoved(false, key createValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createValue;
        }
    }

}

public final V put(K key, V value) {
    V previous;
    synchronized (this) {
        //缓存次数添加
        putCount++;
        size +=safeSizeOf(key, value);
        previous = map.put(key, value);
        if (previous != null) {
            size -= safeSizeOf(key, previous);
        }
    }

    if (previous != null) {
        //缓存被替换，调用到的方法
        entryRemoved(false, key, previous, value);
    }

    trimToSize(maxSize);
    return previous;
}

public void trimToSize(int maxSize) {
    while (true) {
        K key;
        V value;
        synchronized(this) {
            if (size <= maxSize) {
                break;
            }
            Map.Entry<K, V> toEvict = map.eldest();
            if (toEvict == null) {
                break;
            }
            key = toEvict.getKey();
            value = toEvict.getValue();
            // LinkedHashMap构造函数中的accessOrder字段为true，表示读取排序
            // 从最少使用顺序排序到最多排序
            // 所以移除第一个value，等于移除最少使用的缓存
            // map存储结构，直接移除第一个
            map.remote(key);
            size -= safeSizeOf(key, value);
            evictionCount++;
        }
        entryRemoved(true, key, value, null);
    }
}
```

