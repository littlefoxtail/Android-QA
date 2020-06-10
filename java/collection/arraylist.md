# ArrayList

## 总体介绍

ArrayList的底层是数组队列，相当于动态数组。与Java中的数组相比，它的容量能动态增长。在添加大量元素前，应用程序可以使用`ensureCapacity`操作来增加ArrayList实例的容量。这可以减少递增式再分配的数量。

它继承于`AbstractList`，实现了`List`、`RandomAccess`、`Cloneable`、`java.io.Serializable`这些接口。

1. ArrayList实现了`AbstractList`,，实现了`List`。提供了相关的增删改查等功能，是顺序容器，即元素存放的数据与放进去的顺序相同，允许放入null元素，底层通过数组实现。
2. ArrayList实现了List、`RandomAccess`接口，可插入空数据，也支持随机访问。
3. 实现了`Cloneable`接口，即覆盖了函数clone()，能被克隆。
4. ArrayList是基于数组的，而数组是定长的，capacity表示数组的实际大小。
5. ArrayList可以动态调整大小，可以才可以无感知的插入多条数据，也说明其必然有一个默认的大小。而要想扩充数组的大小，只能通过复制。
6. 和 Vector 不同，ArrayList 中的操作不是线程安全的！所以，建议在单线程中才使用 ArrayList，而在多线程中可以选择 Vector 或者 CopyOnWriteArrayList。

## add

```java
public boolean add(E e) {
    //确保数组大小足够，不够需要扩容
    ensureCapacityInternal(size + 1);
    //直接赋值，线程不安全
    elementData[size++] = e;
    return true;
}

private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)
        elementData = grow();
    elementData[s] = e;
    size = s + 1;
}

```

- 首先进行扩容校验
- 将插入的值放到尾部，并将size+1

```java
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                        size - index);
    elementData[index] = element;
    size++;
}
```

- 首先也是扩容
- 对数据进行复制，目的把index位置空出来放本次插入的数据，并将后面的数据向后移动一个位置。

### 自动扩容


```java
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}

private void ensureCapacityCapacity(int minCapacity) {
    modCount++;
    // 如果我们希望的最小容量大于目前数组的长度，那么就扩容
    if (minCapacity - elementData.length > 0) {
        // 需要扩容
        grow(min(minCapacity));
    }
}
```

```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length();
    // 首先设置新的存储能力为原来的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0) {
        newCapacity = minCapacity;
    }
    // 与最大值比较，求取最大值
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        // ArrayList最大的存储能力为整数的范围
        newCapacity = hugeCapacity(minCapacity);
    }
    elementData = Arrays.copyOf(elementData, newCapacity);

}
```

native方法执行的数组复制
Arrays:
![arrays_copyof](/img/arrays_copyof.png)

### 序列化

由于ArrayList是基于动态数组实现的，所以并不是所有的空间都被使用（防止浪费了）。因此使用`transient`，可以防止自动序列化。

```java
class Arraylist {
    transient Object[] elementData;//数组默认不会被序列化

    private void readObject(java.io.ObjectInputStream s) {

    }

    private void wirteObject(java.io.ObjectOutputStream s) throws java.io.IOException{
        int expectedModCount = modConnt;
        s.defaultWriteObject();

        s.write(size);

        for (int i=0; i<size; i++) {
            s.wirteObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }

    }
}

```

当对象中自定义了writeObject和readObject方法时，JVM会调用这两个自定义方法来实现序列化与反序列化

## 删除元素

```java
public E remove(int index) {
    // 检查索引是否越界
    rangeCheck(index);

    modCount++;
    // 根据索引拿到删除的数据
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0) {
        // 将index + 1作为移动的起点，size - index - 1作为需要移动数组的长度
        // 利用System.arraycopy数组，将后面的数据向前移动一位，并将最后一位置空
        System.arraycopy(elementData, index + 1, elementData, index, numMoved);
        elementData[--size] = null;

        return oldValue;
    }
}
```

```java
public boolean remove(Object o) {
    if (o == null) {
        //找到第一个值是空的删除
        for (int index = 0; index < size; index++) {
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
        }
    } else {
        // 根据equals来判断值相等
        for (int index = 0; index < size; index++) {
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
        }
    }
    return false;
}
```

### 遍历速度

1. 标准迭代器方式
2. 外部for循环+get(indext)
3. forEach(lambda)方法
4. for(E e: arrayList)

## 与LinkedList异同

- 是否保证线程安全：ArrayList和LinkedList都是不同步的，不保证线程安全
- 底层数据结构：ArrayList底层使用Object数组；LinkedList底层使用的是双向链表数据结构
- 插入和删除是否受元素位置的影响：1.ArrayList插入和删除元素的时间复杂度受元素位置的影响。2.LinkedList采用链表存储，所以插入，删除元素时间复杂度不受元素的影响，都是近似 O（1）而数组为近似 O（n）
- 是否支持快速随机访问：LinkedList不支持搞笑的随机元素访问，而ArrayList支持。快速随机访问就是通过元素的序号快速获取元素对象(对应于get(int index) 方法)
- ArrayList的空 间浪费主要体现在在list列表的结尾会预留一定的容量空间，而LinkedList的空间花费则体现在它的每一个元素都需要消耗比ArrayList更多的空间（因为要存放直接后继和直接前驱以及数据）

## Vector

Vector也是实现了`List`，底层数据结构和`ArrayList`类似，也是一个动态数组存放数据。不过在add()方法的时候`synchronized`进行同步写数据，开销较大，所以`Verctor`是一个同步容器并不是一个并发容器。

```java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCont + 1);
    elementData[elementCount++] = e;
    return true;
}
```

![vector_add](/img/vector_add.png)
![insertElementAt](/img/insertElementAt.png)

## CopyOnWriteArrayList

1. 线程安全，多线程环境下可以直接使用，无需加锁；
2. 通过锁+数组拷贝+`volatile`关键字保证了线程安全；
3. 每次数组操作，都会把数组拷贝一份出来，在新数组上进行操作，操作成功之后再赋值回去。

```java
private transient volatile Object[] array;
```

### add如何保证数组安全

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

add(e)方法是根据`ReentrantLock`+数组copy+update Object[] 内存地址 + volatile来保证其数据安全性。

