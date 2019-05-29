
# JavaIO

## 概览

Java的I/O分为：

- 磁盘操作：File
- 字节操作：InputStream和OutputStream
- 字符操作：Reader和Writer
- 对象操作：Serializable
- 网络操作：Socket
- 新的输入/输出：NIO

## 磁盘操作

File。
从Java7开始，可以使用Paths和Files代替File。

## 字节操作

### java中基础类型字节占用情况

在Java中一共有8种基础数据类型，其中4种整型，2种浮点类型，1种用于表示Unicode编码的字符单元的字符类型和1种用于表示真值的boolean类型。

|类型|大小|范围（包含）|
|:--:|:--:|:--:|
|byte（字节）|8 bit（位）|-128~127|
|short|1 byte|-32768~32768|
|int|4 byte||
|long|8 byte|
|float|4 byte|
|double|8 byte|
|char|2 byte|

关于boolean占几个字节，java规范中，没有明确指出boolean的大小。在《Java虚拟机规范》给出了4个字节，和boolean数组1个字节的定义，具体还要看虚拟机实现是否按照规范来，所以1个字节、4个字节都是有可能的

### 装饰着模式

Java I/O使用了装饰者模式来实现。

```java

public class InputStream {

    /**
     * 1. 抽象方法，必须由子类实现。数据源：子类提供
     * 2. 每次读取都会读取一个字节，一个byte出来，是有顺序
     */
    public abstract int read() throws IOException;

    public int read(byte b[], int off, int len) throws Exception {
        //循环调用read，将取到的int转换成byte，然后再放到参数b的特定位置上（从off开始放，放len个）
        // 返回值是实际读出来的字节数，如果没有则返回-1
    }
}
```

```java
public class OutputStream {
    /**
     * 写到哪儿由子类决定
     * 参数b是作为要写入的数据，是int类型而不是byte类型；
     * 因为int是占有4个字节共32位，这里规定是取低8位，忽略24位
     */
    public abstract void write(int b) throws IOException;

    public void write(byte b[], int off, int len) throws IOException {
        //循环调用write方法，将b[off]到b[off + len -1]值写入到目的数据源
    }
}
```

## 字符操作

### 编码与解码

编码就是把字符转换为字节（Byte），而解码就是把字节（Byte）重新组合成字符。
如果编码和解码过程使用不同的编码方式那么就会出现乱码。

- GBK编码中，中文字符占2个字节，英文字符占1个字符。
- UTF-8编码中，中文字符占3个字节，英文字符占1一个字符。
- UTF-16be编码中，中文字符和英文字符都占2个字节。