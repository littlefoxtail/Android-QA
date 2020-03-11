# HashMap

## 概要

HashMap是一个关联数组、哈希表，它是线程不安全的，允许key为null，value为null。遍历时无序。其底层结构式数组称之为哈希桶，每个桶里放的是链表，链表中的每个节点，就是哈希表中的每个元素。
在JDK8中，当链表长度达到8，会转化红黑树，以提升它的查询、插入效率，它实现了`Map<K,V>`，`Cloneable`，`Serializable`接口

它根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。 HashMap最多只允许一条记录的键为null，允许多条记录的值为null。HashMap非线程安全，即任一时刻可以有多个线程同时写HashMap，可能会导致数据的不一致。如果需要满足线程安全，可以用 Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。

1. 默认容量 1<<4 = 16
2. 最大容量 1<<30 = 2^32
3. 默认加载因子 0.75f，当容量大于(容量x加载因子)，会进行扩容操作。

> 备注：异或的操作如下：0 ^ 0=0，1 ^ 1 =0，0 ^ 1=1，1 ^ 0=1，也就是相同时返回0，不同时返回1。

## HashMap的工作原理

HashMap底层是`hash数组`和`单向链表`实现，数组中的每个元素都是链表，由`Node内部类`（实现Map.Entry<K, V>接口）实现，HashMap通过put&get方法存储和获取。
存储对象时，将K/V键值传给put()方法：

1. 调用hash(K)方法计算K的hash值，然后结合数组长度，计算得数据下标。
2. 调整数组大小
3. 插入
   1. 如果K的hash值在HashMap中不存在，则执行插入，若存在，则发生碰撞
   2. 如果K的hash值在HashMap中存在，且它们两者equals返回true，则更新键值对
   3. 如果K的hash值在HashMap中存在，且它们两者equals返回false，则插入链表的尾部或者红黑树
   JDK 1.7 之前使用头插法、JDK 1.8 使用尾插法）注意：当碰撞导致链表大于 TREEIFY_THRESHOLD = 8 时，就把链表转换成红黑树）

获取对象，将K传给get()方法：

1. 调用hash(K)方法（计算K的hash值）从而获取该键值所在链表的数组下标
2. 顺序遍历链表，equals方法查找相同Node链表中K值对应的V

hashCode是定位的，存储位置；equals是定性的，比较两者是否相等

## 功能实现-方法

### 确定哈希桶数组索引位置

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)
}

static int indexFor(int h , int length) {//jdk1.8源码没有这个方法了，但源码中依然使用(n -1) & hash
    return h & (lenght - 1) //取模运算
}
```

hash值对数组长度取模运算，消耗比较大。通过 h & (table.lenght - 1)得到的对象保存位，由于HashMap底层数组长度总是2的n次方，所以等价，但&比%更高效。

JDK1.8优化了高位运算，通过hashCode的高16位异或低16位实现，可以在数组table的length比较小的时候，也能保证考虑到高低bit都参与Hash的计算中。

三步：取key的hashCode值、高位运算、取模运算

![hashmap_hash](/img/hashmap_hash.png)

### 链表结构

挂载在哈希表上的元素，链表的结构：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    //用来定位数组索引位置，决定元素存储在Node<K,V>[] table哈希表中的位置
    final int hash;
    final K key;
    V value;
    Node<K,V> next;// 链表后置节点，链表的下一个node，用于解决哈希冲突

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey() {
        return key;
    }

    public final V getValue() {
        return value;
    }

// 设置新的value 同时返回旧的value
    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

// 每一个节点的hash值，是将key的hashCode和value的hashCode亦或得到的
// 作用是将2个hashCode的二级制中，同一个位置相同的值为0，不同为1
    public final int hashCode() {
        return Object.hashCode(key) ^ Object.hashCode(value);
    }

}
```

![hashMap内存结构图](/img/hashMap内存结构图.webp)

这是一个单链表，每一个节点的hash值，是将key的hashCode和value的hashCode亦或得到的

### 构造函数和put方法

![put](/img/hashmap_put.png)

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K, V>[] tab; Node<K, V> p; int n, i;
    if ((tab = table) == null) || (n = tab.length) == 0) //1.判断键值对数组table[i]是否为空或者null，否则执行扩容
        n = (tab = resize()).length;
    if (p = tab[i = (n -1) &  hash] == null)//2.根据键值key计算hash得到插入的数组索引i，如果table[i] == null，直接新建节点添加，转向6，如果table[i]不为空，转向3
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K, V> e; K k;
        if (p.hash == hash &&
            ((K = p.key) == key) || (key != null && k.equals(k))) //3.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向4，这样的相同指的是hashCode和equals
            e = p;
        else if (p instanceof TreeNode)//4.判断table[i]是否为红黑树，如果红黑树，则直接在树种插入键值对，否则转向5
            e = ((TreeNode<K, V)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0;; ++binCount) {//5.遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程若发现key已经存在直接覆盖value即可
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //如果插入新节点后链表长度大于8，则判断是否需要树化
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifBin(tab, hash);
                }
                if (e.hash == hash &&
                    ((k = e.key) == key) || (key != null && key.equals(k)))
                    break
                p = e;
            }
            //如果找到了对应key的元素
            if (e != null) {
                //记录旧的值
                V oldValue = e.value;
                //判断是否替换需要替换旧值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)//插入成功后，判断成功实际存在的键值对数量size是否超过了最大容量threshold，如果超过，扩容。
            resize();
        afterNodeInsertion(evict);
        return null;

    }

}
```

```java
// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 哈希桶，存放链表，长度是2的N次方，或者初始化时为0
transient Node<K, V>[] table;

// 加载因子，用于计算哈希表元素数量的阈值，会发生扩容
final float loadFactor;

// 哈希表内元素数量的阈值，当哈希表内元素数量超过阈值时，会发生扩容resize()
int threshold;

public HashMap() {
    // 默认构造，加载因子为默认的0.75
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}

public HashMap<Map<? extend K, ? extends V m> {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}


final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        // 如果当前列表是空的
        if (table == null) {
            // 根据m的元素数量和当前表的加载因子，计算出阈值
            float ft = ((float)s/loadFactor + 1.0f);
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
            //  返回一个新的阈值，满足2的n次方的阈值
            if (t > threshold) {
                threshold = tableSizeFor(t);
            }

        } else if (s > threshold)
            // m元素数量大于阈值，说明要扩容
            resize();

        // 遍历m依次将元素加入当前表中
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}

/**
 * 根据期望容量cap，返回2的n次方形式的哈希桶实际容量length，返回值一般会>=cap
 */
static final int tableSizeFor(int cap) {
    //为什么要对cap做-1操作呢
    // 为了防止，cap已经是2的幂，执行完后面几条无符号右移操作之后，返回的capacity将是这个cap的2倍
    // 1. 如果n这时候为0，经过后面的几个无符号依然是0，字后返回是1
    int n = cap - 1;
    // 2 由于n不等于0，则n的二进制表示中总会有一bit为1，这时考虑最高位的1。通过无符号右移1位，则将最高位的1右移了1位，再做或操作，使得n的二进制表示中与最高位的1紧邻的右边一位也为1，如000011xxxxxx
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

## 扩容机制

```java
final Node<K,V>[] resize() {
    // oldTab 为当前表的哈希桶
    Node<K, V> oldTab = table;
    // 当前哈系统的容量length
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 初始化新的容量和阈值为0
    int newCap, newThr = 0;
    // 如果当前容量大于0
    if (oldCap > 0) {
        // 如果当前容量已经到达上限
        if (old >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;//如果旧的容量大于默认初始容量16，那么新的阈值也等于旧的阈值的两倍
    } //如果当前表示空的，但是有阈值，代表示初始化时指定了容量、阈值的情况
    else if (oldThr > 0) {
        newCap = oldThr;
    } else {//如果当前表是空的，而且也没有阈值。代表是初始化时没有任何容量/阈值参数的情况  
        newCap = DEFAULT_INITIAL_CAPACITY;//此时新表的容量为默认的容量 16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//新的阈值为默认容量16 * 默认加载因子0.75f = 12
    }

    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }

    // 更新阈值
    threshold = newThr;

    // 根据新的容量 构建新的哈希桶
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];

    // 更新哈希桶引用
    table = newTab;

    // 如果以前的哈希桶中有元素
    // 将当前哈希桶中的所有字节转移到新的哈希桶中
    ...
}

```

## 插入元素到红黑树

```java
final TreeNode<K, V> putTreeVal(..) {
    //标记是否找到这个key的节点
    boolean searched = false;
    //找到树的根节点开始遍历
    TreeNode<K, V> root = (parent != null) ? root() : this;
    //从树的根节点开始遍历
    for (TreeNode<K, V> p = root; ;) {
        //dir = direction，标记是在左边还是右边
        //ph=p.hash，当前节点的hash值
        int dir, ph;
        //pk=p.key 当前节点的key值
        K pk;
        if ((ph = p.hash) > h)
            //当前hash比目标hash大，说明在左边
            dir = -1;
        else if (ph < h)
            //当前hash比目标hash小，说明在右边
            dir = 1;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            //两者hash相同且key相等，说明找到了节点，直接返回该节点
            //回到putVal()中判断是否需要修改其value值
            return p;
        else if ((kc == null &&
                    //如果k是Comparable的子类则返回其真实的类，否则返回null
                    (kc comparableClassFor(k)) == null) ||
                    //如果k和pk不是同样的类型则返回0，否则返回两者比较的结构
                    (dir = comparableComparables(kc, k, pk)) == 0) {
            //这个条件表示两者hash相同但是其中一个不是Comparable类型或者两者类型不同
            // 比如key是Object类型，这时可以传String也可以传Integer，两者hash值可能相同
            // 在红黑树中把同样hash值的元素存储在同一颗子树，这里相当于找到了这颗子树的顶点
            // 从这个顶点分别遍历其左右子树去寻找有没有跟待插入的key相同的元素
            if (!searched) {
                TreeNode<K, V> q, ch;
                searched = true;
                //遍历左右子树找到了直接返回
                if (((ch == p.left) != null &&
                        (q = ch.find(h, k, kc)) != null) ||
                        ((ch = p.right) != null &&
                        (q = ch.find(h, k, kc)) != null))
                        return q;
            }
            //如果两者类型相同，再根据它们的内存地址计算hash值进行比较
            dir = tieBreadOrder(k,  pk);
            if ((p = (dir <= 0)? p.left : p.right) == null) {
                //如果最后确实没找到对应key的元素，则新建一个节点
                Node<K, V> xpn = xp.next;
                TreeNode<K, V> x = map.newTreeNode(h, k, v, xpn);
                if (dir <= 0)
                    xp.left = x;
                else 
                    xp.right = x;
                xp.next = x;
                x.parent = x.prev = xp;
                if (xp != null)
                    ((TreeNode<K, V>) xpn).prev = x;
                    //插入树节点后平衡
                    // 把root节点移动到链表的第一个节点
                moveRootToFont(tab, balanceInsertion(root, x));
                return null;
            }
        }

    }
}
```

1. 寻找根节点
2. 从根节点开始查找
3. 比较hash值及key值，如果都相同，直接返回，在putVal()方法中决定是否要替换value的值
4. 根据hash值及key值确定在树的左子树还是右子树查找，找到了直接返回
5. 如果最后没有找到则在树的相应位置插入元素，并做平衡

## treeifyBin方法

如果插入元素后链表的长度大于等于8则判断是否需要树化

```java
final void treeifyBin(Node<K, V>[] tab, int hash) {
    int n, index; Node<K, V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        //如果桶数量小于64，直接扩容而不用树化
        // 因为扩容之后，链表会分化为两个链表，达到减少元素的作用
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K, V> hd = null, tl = null;
        do {
            //把所有节点换成树节点
            TreeNode<K, V> p = repleacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while((e = e.next) != null);
        //如果进入过上面的循环，则从头结点开始树化
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}

```

## get

```java
// HashMap
final Node<K, V> getNode(int hash, Object key) {
    //用于记录当前的hash表
    Node<K, V> table;
    // first用于记录对应hash位置的第一个结点，e充当工作结点的作用
    Node<K, V> first, e;
    // n用于记录hash表长度
    int n;
    // 用于临时存放Key
    K k;
    // 通过(n - 1) & hash当元素的hash值 & hash表长度 -1
    // 判断当前元素的存储位置是否有元素存在 
    if ((tab == table) != null && (n = tab.length) > 0 &&
        (first = table[(n - 1) & hash]) != null) {//元素存在的情况
        // 如果头结点的key的hash值和要获取的key的hash相等
        // 并且头结点key本身和要获取的key相等
            if (first.hash == hash && 
                ((k = first.key) == key || (key != null && key.equals(k))))//返回该位置的头结点
                return first;
            if ((e = first.next) != null)//头结点不相等
                if (first instanceof TreeNode)//如果当前结点是树结点
                //则证明当前位置的链表已变成红黑树结构
                // 通过红黑树结点方式获取对应key结点


        }
}

```

[HashMap问答内容](hashmap问答内容.md)

## 总结

- `HashMap`定位元素位置是通过键`key`讲过扰动函数扰动得到`hash`值，然后再通过`hash & (length - 1)`代替取模的方式进行元素定位的。
- 