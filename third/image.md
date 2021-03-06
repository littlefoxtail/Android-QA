# 图片缓存

## 缓存Bitmap

将单个Bitmap加载到UI是简单直接的，如果我们一次性加载大量的图片，在大多数情况下，屏幕上的图片和因滑动将要显示的图片的数量
通常是没有限制的。
通过循环利用子视图可以缓解内存的使用，垃圾回收期也会释放那些不再需要使用的Bitmap。这些机制都非常好，但是为了保证一个流畅的用户体验，我们希望避免在每次屏幕滑动回来时，都要重复处理那些图片。内存与磁盘缓存通常可以起到辅助作用，允许控件可以快速地重新加载那些处理过的图片。

### 使用内存缓存(Use a Memory Cache)

内存缓存以花费宝贵的程序内存为前提来快速访问位图。`LruCache`类特别适合用来缓存Bitmaps，它使用了一个强引用的`LinkedHashMap`保存最近引用的对象，并且在缓存超出设置大小的时候剔除最近最少使用到的对象，
并且在缓存超出设置大小的时候提出最近最少使用到的对象。

> Note:在过去,一种比较流行的内存缓存实现方法是使用软引用(SoftReference)或弱引用(WeakReference)对Bitmap进行缓存，然而不推荐这种做法。
从Android2.3开始，垃圾回收机制变得更加频繁，这使得释放软引用的频率也随之增高，导致使用引用的效率降低很多。而且在Android3.0之前，备份的Bitmap
会存放在Native Memory中，它不是以可预知的方式被释放的，这样可能导致程序超出它的内存限制而崩溃。

为了给LruCache选择一个合适的大小，需要考虑到下面一些因素：

* 引用剩下了多少可用内存
* 多少张图片会同时呈现到屏幕上？有多少图片需要准备好以便马上显示到屏幕
* 设备的屏幕大小与密度是多少？一个具有特别高密度屏幕(xhdpi)的设备，需要一个更大的缓存空间来缓存同样数量的图片。
* Bitmap的尺寸与配置是多少，会花费多少内存？
* 图片被访问的频率如何？是其中一些比另外的访问更加频繁吗？如果是，那么我们可能希望在内存中保存那些最常访问的图片，或者根据访问频率给Bitmap分组，为不同Bitmap组设置多个LruCache对象。
* 是否可以在缓存图片的质量和数量之间寻找平衡点？某些时候保存大量低质量的Bitmap会非常有用，加载高质量图片的任务可以叫给另一个后台线程。

通常没有指定的大小或者公式能够适用于所有的情形，我们需要分析实际的使用情况后，提出一个合适的解决方案。缓存大小会导致额外的开销
却没有明显的好处，缓存太大同样会导致OOM的异常，并且使得你的程序只留下小部分的内存用来工作。

当加载Bitmap显示到ImageView之前，先从LruCache中检查是否存在这个Bitmap。如果确实存在，它会立即被用来显示到ImageView上，如果没有
找到，会触发一个后台线程去处理显示该Bitmap任务。

### 使用磁盘缓存(Use a Disk Cache)

内存缓存能够提高访问最近用过的Bitmap的速度，但是我们无法保证最近访问过的Bitmap能够保存在缓存中。像类似GridView等需要大量数据填充的
控件很容易就会用尽整个内存缓存。另外，我们的应用可能会被类似打电话等行为而暂停并退到后台，因为后台应用可能会被杀死，那么内存缓存就会被销毁，里面的Bitmap也就不存在了。一旦用户恢复应用的状态，那么应用就需要重新处理那些图片。

磁盘缓存可以用来保存那些已经处理过的Bitmap，它还可以减少那些不再内存缓存中的Bitmap的加载次数。当然从磁盘读取图片会比从内存要慢，而且磁盘读取
时间是不可预期的，读取操作需要在后台线程中处理。

> 如果图片会被更频繁的访问，使用ContentProvider或许更加合适，比如在图库应用中。因为初始化磁盘缓存涉及到I/O操作，所以它不应该在主线程中进行。

内存缓存的检查是可以在UI线程中进行的，磁盘缓存的检查需要在后台线程中处理。磁盘操作永远都不应该在UI线程中发生。当图片处理完成后，Bitmap需要添加到内存缓存与磁盘缓存中，方便之后的使用。

## 处理配置改变

如果运行时设备配置信息发生改变，例如屏幕方向的改变会导致Android中当前显示的Activity先被销毁然后重启。
我们需要在配置改变时避免重新处理所有的图片，这样才能提供用户一个良好的平滑过渡的体验。
