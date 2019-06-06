```java
/**
    * Return only the state bits of {@link #getMeasuredWidthAndState()}
    * and {@link #getMeasuredHeightAndState()}, combined into one integer.
    * The width component is in the regular bits {@link #MEASURED_STATE_MASK}
    * and the height component is at the shifted bits
    * {@link #MEASURED_HEIGHT_STATE_SHIFT}>>{@link #MEASURED_STATE_MASK}.
    */
// 将宽高的状态位结合成一个32位的int值并返回
// 宽度的状态位在常规的位置
// 高度的状态位在偏移后的位置
public final int getMeasuredState() {
    return (mMeasuredWidth&MEASURED_STATE_MASK)
            | ((mMeasuredHeight>>MEASURED_HEIGHT_STATE_SHIFT)
                    & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
}
// 用于使高度的状态位偏移的位数
public static final int MEASURED_HEIGHT_STATE_SHIFT = 16;

// 用于得出高度的状态位的掩码
public static final int MEASURED_STATE_MASK = 0xff000000;

// 用于得出宽高的尺寸位的掩码
public static final int MEASURED_SIZE_MASK = 0x00ffffff;


public final int getMeasuredHeight() {
    return mMeasuredHeight & MEASURED_SIZE_MASK;
}

public final int getMeasureHeightAndState() {
    return mMeasureHeight;
}
```

mMeasureHeight或mMeasureWidth都是32位的int值，但这个值并不是一个表示宽高的实际大小的值，而是由宽高的状态和实际大小所组合的值，这里的高8位就表示状态(STATE)，而低24表示实际的尺寸大小(SIZE)

getMeausreState怎么将宽高的状态位组合在一个int值中的，首先mMeasuredWidth&MEASURED\_STATE\_MASK得到了宽度的状态位，保存在高8位，然后通过(mMeasuredHeight >> MEASURED\_HEIGHT\_STATE\_SHIFT)和(MEASURED\_STATE\_MASK >> MEASURED\_HEIGHT\_STATE\_SHIFT)将高度和状态掩码都右移了16位，现在高度的状态位在8到15位上，而MEASURED_STATE_MASK变成了0x0000ff00,接着将两个移位后的数进行按位相与(&)得到了高度的状态位,保存在8-15位上.最后将处理后宽度和高度按位相或(|)得到一个保存了宽度和高度的状态位的int值.如下图

![getMeasuredState](/img/getMeasuredState.png)
