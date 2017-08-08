# 注意点
1. 
```
view.measure(int widthMeasureSpec, int heightMeasureSpec)
```
方法签名上的两个MeasureSpec是父类传递过来的，但并不是完全是父View的要求，而是父View的MeasureSpec和子View自己的LayoutParams共同决定的。

