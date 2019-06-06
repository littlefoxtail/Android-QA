# Android在编码的时候经常使用到位运算

首先通常查看Flags的值，都是16进制数值代表，且只使用一位并只为1|2|4|8(与2的次方相关)

```java
public static final int FLAG_ACTIVITY_NEW_TASK = 0x10000000;
public static final int FLAG_ACTIVITY_SINGLE_TOP = 0x20000000;
public static final int FLAG_ACTIVITY_MULTIPLE_TASK = 0x08000000;
```

|十六进制|二进制|
|:---:|:--:|
|1|0001|
|2|0010|
|4|0100|
|8|1000|

特性：他们通过“或运算”可以组成1~15的数，并且不会出现两种或两种以上的相同的情况。

## 通过Intent Flags对应的值，可以将多种标志通过“或运算”来进行组合

```java
mIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
| Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED
| Intent.FLAG_ACTIVITY_SINGLE_TOP)
```

```java
event.mFlags |= FLAG_CANCELED | FLAG_CANCELED_LONG_PRESS;
```

## 判断Intent Flags是否包含某个标志，通过“与运算”

```java
if ((intent.getFlags & intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
    // intent.getFlags 不包含NEW_TASK
}
```

```java
// 判断该视图是否为disable状态 这里ENABLE_MASK的值与DISABLED的值一样
if ((viewFlags & ENABLED_MASK) == DISABLED) {

}
```

```java
// 返回是否可点击  
     return (((viewFlags & CLICKABLE) == CLICKABLE ||
                    (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
```

## 清除某个值

```java
mFlag &= ~FLAG_START_TRACKING; //清除mFlags中FLAG_START_TRACKING
```

位运算主要是直接操控二进制时使用，主要目的节约内存
移位运算：

1. 它们都是双目运算符，两个运算分量都是整形，结果也是整形
2. "<<"左移：右边空出的位上补0，左边的位将从字头挤掉，其值相当于乘2
3. ">>"右移：右边的位被挤掉。对于左边移出的空位，如果是正数则空位补0，若为负数，可能补0或补1，这取决于所用的计算机系统
4. ">>>"运算符，右边的位被挤掉，对于左边移出的空位一概补上0
