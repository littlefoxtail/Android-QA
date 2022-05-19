# ThreadLocal

Threadlocal是一个关于创建线程局部变量的类

通常情况下，创建的变量是可以被任何一个线程访问并修改。而使用ThreadLocal创建的变量只能被当前线程访问，其他线程则无法修改。

在ThreadLocal类中有一个Map，用于存储每一个线程的变量副本，Map中元素的键为线程对象，而值对应线程的变量副本(成员变量)

## Thread

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class*/
ThreadLocal.ThreadLocalMap threadLocals = null;
```

## ThreadLocal接口方法

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

### set

```java
public void set(T value) {
    Thread t = Thread.currentThread(); //1
    ThreadLocalMap map = getMap(t);  //2
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value); //3
    }
}
```

1. 获取当前线程
2. 利用当前线程作为句柄获取一个ThreadLocalMap的对象
3. 如果没有ThreadLocalMap对象，则创建并设置值

> 句柄： 本质相当于带有引用计数的智能指针

总结：实际上ThreadLocal的值是放入了当前线程一个ThreadLocalMap实例中，所以只能在本线程中访问，其他线程无法访问。

### get

返回当前线程的局部变量的值

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

### remove

将当前线程局部变量的值删除，目的是为了减少内存的占用
不过当线程结束后，对应该线程的局部变量将自动被垃圾回收，所以显示调用该方法清除线程的局部变量并不是必须的，但它可以加快内存回收速度。

```java
public void remote() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null) {
        m.remove(this);
    }
}
```

## 对象放在哪儿

堆中
