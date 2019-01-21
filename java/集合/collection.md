# Collection

AbstractCollection

## List

### AbstractList

```java
public Iterator<E> iterator() {
    return new Itr();
}
```


```java
public listIterator<E> listIterator() {
    return listIterator(0);
}
```

```java
public int indexOf(Object o) {
    //寻找一个元素首次出现的位置，只需要从前往后遍历，找到哪个元素并返回其位置即可

}
```

### ArrayList

[ArrayList](arraylist.md)