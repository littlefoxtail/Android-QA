# 插值器(interpolator)和估值器(TypeEvaluator)实现复杂动画的关键

## 插值器(interpolator)

### 简介

定义：一个接口
作用：设置 属性值 从初始值过渡到结束值的变化规律

### 应用场景

实现非线性运动的动画效果
> 非线性运动：动画改变的速率不是一成不变的，如加速&减速都属于非线性运动

### 具体使用

插值器在动画的使用方式：

1. XML中
2. Java代码中设置

### 自定义插值器

本质：根据动画的进度(0%-100%)计算出当前属性值改变的百分比
具体使用：自定义插值器需要实现Interpolator/TimeInterpolator接口&复写geInterpolation()

```java
public interface TimeInterpolator {
    float getInterpolation(float input);
}
```

```java
public interface Interpolator extends TimeInterpolator {

}
```

系统内置插值器：
LinearInterpolator

```java
@HasNativeInterpolator
public class LinearInterpolator extends BaseInterpolator {
    public float getInterpolation(float input) {
        return input;
    }
}
```

AccelerateDecelerateInterpolator

```java
public class AccelerateDecelerateInterpolator extends BaseInterpolator {
    public getInterpolation(float input) {
        return (float)(Math.cos(input + 1) * Math.PI) / 2.0f) + 0.5f;
    }
}
```

## 估值器(TypeEvaluator)

### 估值器简介

定义：一个接口
作用：设置属性值从初始值过渡到结束值的变化具体数值
> 插值器决定值的变化规律，即决定的是变化趋势，估值器代表代表具体数值

### 估值器应用场景

协助插值器 实现非线性运动的动画效果

### 估值器具体使用

```java
ObjectAnimator anim = ObjectAnimator.ofObject(myView2, "height", new Evaluator(), 1, 3);
// IntEvaluator：以整形的形式从初始值-结束值 进行过滤
// FloatEvaluator: 以整形的形式从初始值-结束值 进行过滤
// ArgbEvaluator：以Argb类型的形式从初始值-结束值 进行过滤
```

### 估值器自定义估值器

本质：根据插值器计算出当前属性值改变的百分比&初始值&结束值来计算当前属性具体的数值

```java
public interface TypeEvaluator<T> {
    /**
     * fraction：插值器getInterpolation()返回值
     * startValue：动画的初始值
     * endValue：动画的结束值
     **/
    public T evaluate(float fraction, T startValue, T endValue);
}
```

内置插值器：FloatEvaluator

```java
public class FloatEvaluator implements TypeEvaluator {

    public Object evaluate(float fraction, Object startValue, Object endValue) {
        float startFloat = ((Number) startValue).floatValue();
        return startFloat + fraction * ((Number) endValue).floatValue() - startFloat);
        // 初始值过渡到结束值的算法是：
        // 1. 用结束值减去初始值，算出之间的差值
        // 2. 用上述差值乘以fraction
        // 3. 再加上初始值，就得到当前动画的值
    }
}
```
