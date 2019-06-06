# 内存泄露

## 什么是内存泄露

java是有垃圾回收机制的，java虚拟机会派出一些回收线程回收那些不再被需要的内存空间

- Q1：什么叫不再被需要的内存空间

Java没有指针，全凭引用来和对象进行关联，通过引用来操作对象。如果一个对象没有与任何引用关联，那么这对象也就不太可能被使用到了，回收器便是把这些“无任何引用的对象”作为目标，回收了它们占据的内存空间

- Q2：如果分辨为对象无引用
    2种方法：
    1. 引用计数法
    直接计数，简单高效，Python便是采用该方法。但是如果出现两个对象相互引用，即使它们都无法被外界访问到，计数器不为0它们也始终不会被回收。
    2. 可达性分析
    这个方法设置了一系列的“GC Roots”对象作为索引的起点，如果一个对象与起点对象之间均无可达路径，那么这个不可达的对象就会成为回收对象。这种方法处理 两个对象相互引用的问题，如果两个对象均没有外部引用，会
    判断为不可达对象进而被回收
    ![gcroot](/img/gcroot.png)

## 引用类型

### 强引用

```java
Object o = new Object();
```

强引用指向的对象无论在何时，都不会被GC清理掉

### java.lang.ref.Reference

`java.lang.ref.Reference`为软引用、弱引用、虚引用的父类

#### 构造函数

```java
public abstract class Reference<T> {
    Reference(T referent) {
        this(referent, null)
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue = null) ? ReferenceQueue.NULL : queue;
    }
}
```

在创建一个引用对象时候，指定了`ReferenceQueue`，那么当引用对象指向的对象达到合适的状态，GC会把引用对象本身添加到这个队列中，方便我们处理它。
> 引用对象指向的对象GC会自动处理，但是引用对象本身也是对象，所以需要我们自己清理

```java
SoftReference<String> ss = new SoftReference("abc", queue);
```

`ss`为软引用，指向`abc`这个对象，`abc`会在一定时机被GC清理，但是`ss`对象本身的清理工作依赖于`queue`，当`ss`出现在`queue`中时，说明其指向的对象已经无效，可以放心清理`ss`

#### 四种状态

每一时刻，`Reference`对象都处于下面四种状态中。这四种状态用`Reference`的成员变量`queue`与`next`来标示

```java
ReferenceQueue<? super T> queue;
Reference next;
```

- Active:新创建的引用对象都是这个状态，在GC检测到引用对象已经达到合适的reachability时，GC会根据引用对象是否在创建时指定`ReferenceQueue`参数进行状态转移，如果指定了，那么转移到`Pending`，如果没指定，转移到`Inactive`

- Pending:pending-Reference列表中的引用都是这个状态，它们等着被内部线程`ReferenceHandler`处理
  
    ```java
    queue = ReferenceQueue
    next = 该queue中的下一个引用，如果是该队列中的最后一个，那么为`this`
    ```

- Enqueued：调用`ReferenceQueue.enqueued`方法后的引用处于这个状态中。没有注册的实例不会进入这个状态
  
    ```java
    queue = ReferenceQueue.ENQUEUED
    next = 该queue中的下一个引用，如果是该队列中的最后一个，那么为`this`
    ```

- Inactive：最终状态，处于这个状态的引用对象，状态不会在改变

    ```java
    queue = ReferenceQueue.NULL
    next = this
    ```

![reference](/img/reference.png)

### 软引用(soft reference)

软引用“保存”对象的能力稍逊于强引用，但是高于弱引用，一般来实现memory-sensitive caches
> 软引用指向的对象会在程序即将出发OOM时候被GC清理掉，之后，引用对象会被放到`ReferenceQueue`中

### 弱引用(weak reference)

弱引用“保存”对象的能力稍逊于软引用，但是高于虚引用，一般用来实现canonicalizing mapping
> 当弱应用指向的对象只能通过弱引用（没有强引用或软引用）访问时，GC会清理掉该对象，之后，引用对象会被放到`ReferenceQueue`中

### 虚引用(phantom reference)

虚引用是“保存”对象能力最弱的引用，一般用来实现scheduling pre-mortem cleanup actions in a more flexible way than is possible with the Java finalization mechanism
> 调用虚引用的`get`方法，总会返回`null`，与软引用和弱引用不同的是，虚引用被`enqueued`时，GC并不会自动清理虚引用指向的对象，只有当指向该对象的所有虚引用全部被清理(enqueued)后或其本身不可达时，该对象才会被清理

## WeakHashMap

### WeakHashMap.Entry

```java
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K, V> {
    Entry(Object key, V value,
        ReferenceQueue<Object> queue,
        int hash, Entry<K,V> next) {
        //这里把key传给了父类WeakReference，说明key为弱引用（没有显式的 this.key = key）
         //所有如果key只有通过弱引用访问时，key会被 GC 清理掉
         //引用对象本身Entry会进入queue中，等待被处理
         //还可以看到value为强引用（有显式的 this.value = value ），但这并不影响
         //后面可以看到WeakHashMap.expungeStaleEntries方法是如何清理value的
        super(key, queue);
        this.value = value;
        this.hash = hash;
        this.next = next;
    }
}
```

```java
 /**
     * Expunges stale entries from the table.
     */
    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                // e为要清理的对象
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                // while循环冲突链
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```
