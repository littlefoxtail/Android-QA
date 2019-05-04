# LayoutInflater

## 什么是LayoutInflater

LayoutInflate是一个用于将xml布局，它主要用于加载布局，在Fragment的onCreateView方法，ListView Adapter的getView方法等许多地方都可以见到它的身影。

## 获取LayoutInflater

LayoutInflater本身是一个抽象类，通常有三种方式

### 第一种

```java
LayoutInflater inflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```

### 第二种

```java
LayoutInflater inflater = LayoutInflater.from(context);
```

### 第三种

```java
在Activity内部调用getLayoutInflater()方法
```

后两种方法的实现

```java
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

在Activity内部调用getLayoutInflater方法其实调用的是PhoneWindow的mLayoutInflater

```java
public PhoneWindow(Context context) {
    super(context);
        mLayoutInflater = LayoutInflater.from(context);
    }
}
```

inflate有多个不同的重载方法：

1. inflate(int resource, ViewGroup root)
2. inflate(int resource, ViewGroup root, boolean attachToRoot)
3. inflate(XmlPullParser parser, ViewGroup root)
4. inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot)

前三个方法都调用了

```java
public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
	...
}
```

```java
 if (root != null) {
    // 如果指定了root参数的话，根据节点的布局参数生成合适的LayoutParams
    params = root.generateLayoutParams(attrs);
    // 若指定了attachToRoot为false，会将生成的布局参数应用于上一步生成的View
    if (!attachToRoot) {
        temp.setLayoutParams(params);
    }
}
// 由上至下，递归加载xml内View，并添加到temp里
rInflate(parser, temp, attrs, true);
// 如果root不为空且指定了attachToRoot为true时，会将temp作为子View添加到root中
if (root != null && attachToRoot) {
    root.addView(temp, params);
}
// 如果指定的root为空，或者attachToRoot为false的时候，返回的是加载出来的View，
// 否则返回root
if (root == null || !attachToRoot) {
    result = temp;
}

```

总结：
从这里我们可以理清root和attachToRoot参数的关系了：

root != null， attachToRoot == true：
传进来的布局会被加载成为一个View并作为子View添加到root中，最终返回root；
而且这个布局根节点的android:layout_参数会被解析用来设置View的大小。

root == null， attachToRoot无用：
当root为空时，attachToRoot是什么都没有意义，此时传进来的布局会被加载成为一个View并直接返回；
布局根View的android:layout_xxx属性会被忽略。

root != null， attachToRoot == false：
传进来的布局会被加载成为一个View并直接返回。
布局根View的android:layout_xxx属性会被解析成LayoutParams并保留。(root只用来参与生成布局根View的LayoutParams)

### 加载xml布局的原理

其实就是从根节点开始，递归解析xml的每个节点，每一步递归的过程是：通过节点名称（全类名），使用ClassLoader创建对应类的实例，也就是View，然后，将这个View添加到它的上层节点（父View）。并同时会解析对应xml节点的属性作为View的属性。每个层级的节点都会被生成一个个的View，并根据View的层级关系add到对应的直接父View（上层节点）中，最终返回一个包含了所有解析好的子View的布局根View。

