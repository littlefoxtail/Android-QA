# 集合

## 数组

一种基本类型，其可以通过下标获取到对应位置的数据。
数组在内存中是一段连续的存储单元，每个数据依次放在每个单元中：

- 创建一个数组，必须声明其长度，这意味着数组的大小是固定的，无法动态调整大小。
- 想要获取数组中第i个元素，其时间复杂度是O(1)，因为可以根据其地址直接找到它。同理修改也是。
- 因为地址是连续，想要在数组中插入一个元素是复杂的，因为从插入位置起，后边的所有元素都需要向后移动一位。同理删除也是，只是移动的方向向前。并且数组满时，就无法继续插入
- 因为数组要占据一整块内存，有可能产生许多碎片，也可能因为找不到合适的内存块，而导致存储失败

## 链表

链表是一种离散的存储结构，其在内存中存储是不连续的，每个数据元素都通过一个指针指向其下一个元素的地址。
根据指针域的不同，链表分为单链表、双向链表、循环链表等

- 声明一个链表时，不需要知道其长度，也不需要连续的内存块，所以其大小可以动态调整
- 链表的每个元素分为数据域和指针域，前者是实际存储的数据，后者则指向下一个元素的地址。和数组相比，每个元素需要占用内存更大了
- 要获取链表的第i个元素变得复杂，因为其地址存放在它上一个元素的指针域，所以只能从第一个元素起，进行i次操作
- 链表对查询表现也一般，需要遍历，时间复杂度为O(n)

## 数组与链表的选择

- 数组按照位置查找迅速，链表增删方便
- 数组是固定大小，链表可以随时扩充与缩减
- 链表每个元素占据内存略多与数组
- 数组和链表在查询方面表现都比较一般，耗时较长

### 表

无论是数组还是链表，对其数据的查询表现都比较无力，要想知道一个元素是否在数组或链表中，只能从前后挨个对比。出现这个问题的根源在于，没有办法直接根据一个元素找到它存储的位置。
哈希表就是解决查询问题的一种方案

### 哈希表与Hash函数

哈希表就是通过关键字来获取数据的一种数据结构，它通过把关键字映射为表中的位置来获取元素，这种映射主要使用Hash函数

Hash函数，实际上是建立起key与int值映射关系的函数。好比身份证号一样。
哈希表，就是一个数组，只是其元素不是按照数组的规则排列的。任何一个元素要放进哈希表，都必须先通过Hash函数获取一个int数值，这个数值经过处理后将作为它的存放位置，然后这个元素才能放进哈希表中。
哈希表完全继承了数组的优点，又显著的提高了查询的速度，又显著的提高了查询的速度，但有一个缺陷，这就是哈希碰撞

#### 哈希碰撞

无论什么对象，都根据一个规则映射为一个int值。被转换的对象有无数种可能，但是int的值是有限的，这样必然会有不同的对象，映射到相同的int值，这就是所谓的哈希碰撞。发生碰撞之后，单纯的一维数组已经无法满足需求了

比较通用的方法，就是使用数组+链表组合的方式。当出现哈希碰撞时，在该位置的数据就通过链表的方式链接起来

JDK1.7及之前的版本，HashMap的存储结构和上图是一致的，JDK1.8之后还加入了红黑树进一步优化

哈希表是一种优化存储的思想，具体存储元素的依然是其他的数据结构。设计良好的哈希表，能同时兼备数组和链表的优点，它能在插入和查找时都具备良好的性能。然而设计不好的哈希表，有可能会出现较多的哈希碰撞，导致链表过长，从而哈希表会更像一个链表。还有当数据量很大时，为防止链表过长，就需要对数组进行扩容，这时就涉及到了数组的拷贝，其对性能的影响也很严重，所以需要提前对可能的情况有良好的预测，才能真正发挥哈希表的优势

## 树与二叉树

一对多问题则需要树来解决
树(Tree)是n个节点的有限集。n=0时称为空树。在任意一棵非空树种：

1. 有且仅有一个特定的称为根(Root)的结点
2. 当n>1，其余节点可分为m(m>0)个互不相交的有限集T1、T2、...、Tm，其中每一个集合本身又是一棵树，并且称为根的子树

### 结点分类

树的节点包含一个数据元素及若干指向其子树的分支。结点拥有的子树数称为结点的度（Degree）。度为0的结点称为叶结点（Leaf）或终端结点。

### 结点之间的关系

结点的子树的根称为该结点的孩子（Child），相应地，该结点称为孩子的双亲（Parent）。同一个双亲的孩子之间互称兄弟（Sibling）。

### 深度

结点的层次（Level）从根开始定义起，根为第一层，根的孩子为第二层。某结点在第L层，则其子树的根就在L+1层。其双亲在同一层的结点互为堂兄弟

### 有序树，无序树

#### 二叉树

二叉树（Binary Tree）是n个结点的有限集合，该集合或者空集，或者由一个根结点和两棵互不相交的、分别称为根结点的左子树和右子树的二叉树组成。二叉树就是每个结点的度≤2的树

##### 二叉树遍历

二叉树的遍历是指从根结点出发，按照某种次序依次访问二叉树中所有结点，使得每个结点被访问一次仅被访问一次

##### 前序遍历

若二叉树为空，则空操作返回，否则先访问根结点，然后前序遍历左子树，再前序遍历右子树。

##### 中序遍历

中序遍历根结点的左子树，然后是访问根结点，最后中序遍历右子树

##### 后序遍历

从左到右先叶子结点的方式遍历访问左右子树，最后是访问根结点

## 二叉排序树

二叉排序树的方案则使元素有序，这样便可以使用二分法进行查找了

### 定义

二叉排序树(Binary Sort Tree)，又称为二叉查找树。它或者是一棵空树，或者是具有下列性质的二叉树：

- 若它的左子树不空，则左子树上所有结点的值均小于它的根结点的值
- 若它的右子树不空，则右子树上所有结点的值均大于它的根结点的值
- 它的左、右子树也分别为二叉排序树







## HashMap

### 概要

HashMap是一个关联数组、哈希表，它是线程不安全的，允许key为null，value为null。遍历时无序。其底层结构式数组称之为哈希桶，每个桶里放的是链表，链表中的每个节点，就是哈希表中的每个元素。
在JDK8中，当链表长度达到8，会转化红黑树，以提升它的查询、插入效率，它实现了`Map<K,V>`，`Cloneable`，`Serializable`接口

它根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。 HashMap最多只允许一条记录的键为null，允许多条记录的值为null。HashMap非线程安全，即任一时刻可以有多个线程同时写HashMap，可能会导致数据的不一致。如果需要满足线程安全，可以用 Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。

1. 默认容量 1<<4 = 16
2. 最大容量 1<<30 = 2^32
3. 默认加载因子 0.75f，当容量大于(容量x加载因子)，会进行扩容操作。
> 备注：异或的操作如下：0 ^ 0=0，1 ^ 1 =0，0 ^ 1=1，1 ^ 0=1，也就是相同时返回0，不同时返回1。

### 链表节点

挂载在哈希表上的元素，链表的结构：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;// 链表后置节点

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
    public final int hashCode() {
        return Object.hashCode(key) ^ Object.hashCode(value);
    }

}
```

这是一个单链表，每一个节点的hash值，是将key的hashCode和value的hashCode亦或得到的

### 构造函数

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

public HashMap<Map ? extend K, ? extends V m> {
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

// 根据期望容量cap，返回2的n次方形式的哈希桶实际容量length，返回值一般会>=cap
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```



扩容：

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

## TreeSet和TreeMap的关系

TreeMap是Map接口的常用实现类，而TreeSet是Set接口的常用实现类。TreeSet底层是通过TreeMap来实现，而TreeMap的实现就是红黑树算法。

相同点：

1. TreeMap和TreeSet都是有序的集合
2. TreeMap和TreeSet都是非同步集合
3. 运行速度都要比HashMap集合慢，他们内部对元素的操作实现复杂度为O(logN)。

不同点：

1. 最主要区别是TreeSet和TreeMap分别实现了不同接口
    TreeSet只存储一个对象，而TreeMap存储两个对象Key和Value（仅仅key对象有序）
    TreeSet中不能有重复对象，而TreeMap中可以存在
