# Drawable

## 概述

Drawable是一个用于处理各种可绘制资源的抽象类。我们使用Drawable最常见的情况就是将获取到的资源绘制到屏幕上；
Drawable类提供了一些通用的API来处理以下具有多种表现形式的可视资源/视觉资源。
和View不同，Drawable实例不具备任何能力接收事件或与用户交互。

## Drawable绘制流程

### 通过Resource获取Drawable实例

```java
public Drawable getDrawable(int id, Theme theme) {
    return getDrawableForDensity(id, 0, theme);
}
```

### 将获取的Drawable实例当做背景设置给View

```java
public class View {
    public void setBackgroundDrawable(Drawable background) {

    }
}
```
