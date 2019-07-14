# 字符串

## 

## String、StringBuffer和StringBuilder

||||
|:--:|:--:|:--:|
|String|字符串常量|-|JDK1.0|
|StringBuilder|字符串变量|线程不安全|JDK1.5|
|StringBuffer|字符串变量|线程安全|JDK1.0|

### String不可变

虽然String、StringBuffer和StringBuilder都是final类，它们生成的对象都是不可变的，而且它们内部也都是靠char数组实现的，但是不同之处在于，String类中定义的char数组是final的，而StringBuffer和StringBuilder都是继承自AbstractStringBuilder类，它们的内部实现都是靠这个父类完成的，而这个父类中定义的char数组只是一个普通是私有变量，可以用append追加。因为AbstractStringBuilder实现了Appendable接口。

### why String要设计成不可变

1. 字符串常量
    字符串常量（String pool，String intern，String保留池）是Java堆内存中一个特殊的存储区域，当创建一个String对象时，假设此字符串值已经存在于常量池中，则不会创建一个新的对象，而是引用已经存在的对象。
2. 允许String对象缓存HashCode
    Java中String对象的哈希码被频繁地使用, 比如在hashMap 等容器中。
    字符串不变性保证了hash码的唯一性,因此可以放心地进行缓存.这也是一种性能优化手段,意味着不必每次都去计算新的哈希码.
3. 安全性
    String被许多的Java类（库）用来当做参数，假若String不是固定不变的，将会引起各种安全隐患。

### 三者区别

String 类型和 StringBuffer 类型的主要性能区别其实在于 String 是不可变的对象, 因此在每次对 String 类型进行改变的时候其实都等同于生成了一个新的 String 对象，然后将指针指向新的 String 对象。如果经常改变字符串内容，最好不要用 String ，因为每次生成对象都会对系统性能产生影响，而且当内存中无引用的对象多了以后， JVM 的 垃圾回收器GC 就会开始工作，程序运行速度就会变慢。如果是定义一个StringBuffer 类型的对象，每次对字符串内容进行操作都会对 StringBuffer对象本身进行操作，而不是生成新的对象。所以一般情况下推荐使用 StringBuffer ，特别是字符串对象经常改变的情况下。StringBuffer中很多方法可以带有synchronized关键字，所以可以保证线程是安全的，但StringBuilder的方法则没有该关键字，所以不能保证线程安全。StringBuilder类提供一个与 StringBuffer 兼容的 API，该类被设计用作 StringBuffer 的一个简易替换，用在字符串缓冲区被单个线程使用的时候。如果可能，建议优先采用该类，因为在大多数实现中，它比 StringBuffer 要快。两者的方法基本相同。

在某些情况下，String的效率比StringBuffer要快，例如：

```java
String s1 = "123" + "456" + "789";
StringBuffer s2 = new StringBuffer("123").append("456").append("789");
```

第一行代码在java虚拟机看来其实就是：

```java
String s1 = "123456789";
```

## 总结

在jdk1.5的时候，终于决定提供一个非线程安全的Stringbuffer实现，并命名为stringbuilder，基本上没有使用StringBuffer的情况了。顺便，javac好像大概也是从这个版本开始，把所有用加号连接的string运算都隐式的改写成StringBuilder，也就是说，从jdk1.5开始，用加号拼接字符串已经没有任何性能损失了。但在有循环的情况下，编译器没法做到足够智能的替换，仍然会有不必要的性能损耗，因此，用循环拼接字符串的时候，还是老老实实的用StringBuilder吧。
