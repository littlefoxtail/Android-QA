# ArrayList

ArrayList是基于数组的，而数组是定长的。
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