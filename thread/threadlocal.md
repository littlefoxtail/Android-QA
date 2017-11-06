Threadlocal是一个关于创建线程局部变量的类

通常情况下，创建的变量是可以被任何一个线程访问并修改。而使用ThreadLocal创建的变量只能被当前线程访问，其他线程则无法修改。

```java
ThreadLocal<String> mStringThreadLocal = new ThreadLocal<>();

mStringThreadLocal.set("fda");

mStringThreadLocal.get();
```

# Set
1. 获取当前线程
2. 利用当前线程作为句柄获取一个ThreadLocalMap的对象
3. 如果没有ThreadLocalMap对象，则创建并设置值
> 句柄： 本质相当于带有引用计数的智能指针

总结：实际上ThreadLocal的值是放入了当前线程一个ThreadLocalMap实例中，所以只能在本线程中访问，其他线程无法访问。

# 对象放在哪儿
堆中