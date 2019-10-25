# LinkedList
<!-- TOC -->

- [LinkedList](#linkedlist)
  - [总体介绍](#%E6%80%BB%E4%BD%93%E4%BB%8B%E7%BB%8D)
  - [方法剖析](#%E6%96%B9%E6%B3%95%E5%89%96%E6%9E%90)
    - [add()](#add)
    - [remove](#remove)
    - [get()](#get)
    - [set()](#set)

<!-- /TOC -->
## 总体介绍

LinkedList同时实现了List接口和Deque接口，也就说它既可以看作一个顺序容器，又可以看作一个队列（Queue），同时又可以看作一个栈（Stack）。这样看来，LinkedList简直就是一个全能冠军。关于栈和队列，现在首选是ArrayDeque，它有比LinkedList（当作栈或队列使用时）有着更好的性能。

LinkedList底层通过双向链表实现，本节将着重讲解插入和删除元素时双向链表的维护过程，也即是直接跟List接口相关的函数。LinkedList通过first和last引用分别指向链表的第一个和最后一个元素。当链表为空的时候first和last都指向null

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

## 方法剖析

### add()

add()方法有两个版本，一个是`add(E e)`，该方法在LinkedList的末尾插入元素，因为有`last`指向链表末尾，在末尾插入元素的花费是常数时间。只需要简单修改几个相关引用即可；另一个是`add(int index, E element)`，该方法是在指定下表处插入元素，需要先通过线性查找到具体位置。

```java
public boolean add(E e) {
    linklast(e);
    return true;
}

void linklast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}

// 1.根据index找到要插入的位置
// 2.修改引用，完成插入操作
public void add(int index, E element) {
    if (index == size)
        linkLast(element);
    else
        linkBefor(element, node(index));//因为链表是双向的，可以从开始往后找，也可以从结尾往前找，具体朝那个方向找取决于条件index < (size >> 1)，也即是index是靠近前端还是后端。
}

void linkBefor(E e, Node<E> succ) {
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

### remove

`remove()`方法也有两个版本

```java
public E remove(int index) {
    return unlink(node(index));
}

//删除一个Node
E unlink(Node<E> x) {
    final E element = x.item
    final Node<E> next = x.next
    final Node<E> prev = x.prev
    if (prev == null) {//删除的是第一个元素
        first = next;
    } else {
        prev.next = next;
        x.prev = null
    }

    if (next == null) {//删除的是最后一个元素
        last = prev;
    } else {
        next.prev = prev;
        x.next = null
    }
    x.item = null;
    size--;
    modCount++;
    return element;
}
```

### get()

```java
public E get(int index) {
    return node(index).item;
}
```

### set()

```java
public E set(int index, E element) {
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```
