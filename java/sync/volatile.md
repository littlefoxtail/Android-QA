# volatile

volatile的作用是：作为指令关键字，确保本条指令不会因编译器的优化而省略，而要求每次直接读值。

一个定义为volatile的变量是说这变量可能会被意想不到的改变，这样，编译器就不会去假设这个变量的值了。精确地说就是，优化器在用到这个变量时必须每次都小心地重新读取这个变量的值，而不是使用保存在寄存器里的备份。

1. 并行设备的硬件寄存器
2. 一个终端服务子程序中会访问的非自动变量
3. 多线程应用中被几个任务共享的变量

在多线程环境下volatile关键字的使用变得非常重要
在当期的java内存模型下，线程可以把变量保存在本地内存(比如机器的寄存器)中，而不是直接在主存中进行读写。这就可能造成一个线程在主存中修改了一个变量的值，而另外一个线程还继续使用它在寄存器中的变量值得拷贝，造成数据的不一致。

volatile修饰的成员变量在每次被线程访问时，都强迫从共享内存中重读成员变量的值。而且，当成员变量发生变化时，强迫线程将变化值回写到共享内存。这样在任何时候，两个不同的线程总是看到某个成员变量的同一个值。
Java语言规范中指出：为了获得最佳速度，允许线程保存共享成员变量的私有拷贝，而且只当线程进入或者离开同步代码块时才与共享成员变量的原始值对比。
这样当多个线程同时与某个对象交互时，就必须要注意到要让线程及时的得到共享成员变量的变化。
volatile关键字就提示JVM：对于这个成员变量不能保存它的私有拷贝，而应直接与共享成员变量交互。
使用建议：在两个或者更多的线程访问的成员变量上使用volatile。当要访问的变量已在synchronized代码块中，或者为常量时，不必使用。
由于使用volatile屏蔽掉了JVM中必要的代码优化，所以在效率上比较低，因此一定在必要时才使用此关键字。

## 正确使用

Java语言包含两种内在的同步机制：同步块或方法和volatile变量。这两种机制的提出都是为了实现代码线程的安全性。其中volatile变量的同步性较差，而且其使用也更容易出错。

volatile变量可以看作一种“程度较轻的synchronized”；与synchronized块相比，volatile变量所需的编码较少，并且运行时开销也较少，但是它所能实现的功能也仅是synchronized的一部分。

synchroniz原理与valatile不一样，释放锁之前其余线程是访问不到这个共享变量的。

## volatile的应用

双重检查锁的单例模式
可以用`volatile`实现一个双重检查锁的单例模式：

```java
public class Singleton {
    private static volatile Singleton singleton;
    private Singleton() {}
    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized(Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }

}ma
```

这里的`volatile`关键字主要是为了防止指令重排。如果不用`volatile`，`singleton = new Singleton()；`，这段代码其实分为三步：

- 分配内存空间。
- 初始化对象
- 将`singleton`对象指向分配的内存地址

加上`volatile`是为了让以上三步操作顺序执行，反之可能；有第二步在第三步值钱被执行就有可能某个线程拿到的单例对象是还没有初始化的

## 总结

`volatile`关键字只能保证可见性，顺序性，不能保证原子性