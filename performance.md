## Android内存分析方向
* Java内存分析
 * Java中的内存泄露主要特征：可达，无用
 * 无用指的是创建了但是不再使用之后没有释放
 * 能重用但是却创建了新的对象进行处理

* Native内存分析
 * 堆中new的对象未释放
 * 对象引用导致无法释放

* JS中内存分析

### 一.日志分析
查看日志中是否有频繁的GC。通常通过log

### 二.常见内存泄露查找
Context 泄漏, 主要为Activity 传递泄漏， context 未使用applciationConext 在单例创建时。  
Handler 泄漏 , handler中持有view ，context 等做耗时操作。  
Cursor 泄漏 , cursor未关闭  
register 未 unregister  
Bitmap  
adapter 未使用convertView  
不良代码等  

