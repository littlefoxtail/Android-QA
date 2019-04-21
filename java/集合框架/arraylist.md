# ArrayList

## 总体介绍

ArrayList实现了List接口，是顺序容器，即元素存放的数据与放进去的顺序相同，允许放入null元素，底层通过数组实现。
ArrayList是基于数组的，而数组是定长的，capacity表示数组的实际大小。
ArrayList可以动态调整大小，可以才可以无感知的插入多条数据，也说明其必然有一个默认的大小。而要想扩充数组的大小，只能通过复制

ArrayList实现了List、RandomAccess接口，可插入空数据，也支持随机访问。
ArrayList相当于动态数据，其中最重要的两个属性`elementData`数组，以及`size`大小。在调用`add()`方法的时候：

## add

![array_add](/img/array_add.png)

- 首先进行扩容校验
- 将插入的值放到尾部，并将size+1

![array_add(int, e)](/img/array_add_int_e.png)

- 首先也是扩容
- 对数据进行复制，目的把index位置空出来放本次插入的数据，并将后面的数据向后移动一个位置。

### 自动扩容

![array_ensureCapacityInternal](/img/array_ensureCapacityInternal.png)

```java
private void ensureCapacityCapacity(int minCapacity) {
    modCount++;
    // 如果最低要求的存储能力大于ArrayList已有的存储能力
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
```

当对象中自定义了writeObject和readObject方法时，JVM会调用这两个自定义方法来实现序列化与反序列化

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

