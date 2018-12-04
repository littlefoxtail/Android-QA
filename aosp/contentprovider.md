
# ContentProvider

## 时序图

初始化解析：
![ContentProvider初始化时序图](/img/ContentProvider初始化解析时序图.png)

获取：
![ContentProvider获取时序图](/img/ContentProvider获取时序图.png)

四大组件之一，没有activity那样复杂的生命周期，只有简单的onCreate过程。ContentProvider是一个抽象类，当实现自己的ContentProvider类，只需继承与ContentProvider

Uri固定的数据格式，例如：`content://com.gityuan.articles/android/3`

|字段|含义|对应项|
|---|---|--|
|前缀|默认的固定开头格式|content://|
|授权|唯一标识provider|com.gityuan.articles|
|路径|数据类别以及数据项|/android/3|

![get_contentProvider](../img/get_contentProvider.jpg)

## ContentResolver

其他app或者进程想要操作`ContentProvider`，则需要先获取其相应的`ContentResolver`，利用ContentResolver类来完成对数据的增删改查操作
