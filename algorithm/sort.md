
时间复杂度：分析关键字的比较次数和记录的移动次数。  
它定量描述了该算法的运行时间。这是一个代表算法输入值的字符串的长度的函数。时间复杂度用大O符号表述，不包括这个函数的低阶项
和首项系数。使用这种方式时，时间复杂度可被称为是渐近的，亦即考察输入值大小趋近无穷时的情况。  

为了计算时间复杂度，我们通常会估计算法的操作单元数量，每个单元运行的时间都是相同的。因此，总运行时间和算法的操作单元数量最多
相差一个常量系数。

相同大小的不同输入值仍可能造成算法的运行时间不同，因此我们通常使用算法最坏情况复杂度，记为<img src="https://latex.codecogs.com/gif.latex?T(n)" title="T(n)" />，定义为任何大小的输入n所需的
最大运行时间。另一种较少使用的方法是平均情况复杂度，通常有特别指定才会使用。时间复杂度可以用函数`T(n)`的自然特性加以分类。
例如：T(n) = O(n)的算法被称作*线性时间算法*；而 <img src="https://latex.codecogs.com/gif.latex?T(n)&space;=&space;O(Mn)" title="T(n) = O(Mn)" /> 和 <img src="https://latex.codecogs.com/gif.latex?Mn=&space;O(T(n))" title="Mn= O(T(n))" /> ，其中 M ≥ n > 1 的算法被称作指数时间算法。

[时间复杂度](https://zh.wikipedia.org/wiki/时间复杂度)

|名称|复杂度类|运行时间(T(n))|运行时间举例|算法举例|
| ---- | ---- | ---- | ---- | ---- |
|常数时间|      |<img src="https://latex.codecogs.com/gif.latex?O(1)" title="O(1)" />|10|判断一个二进制数的奇偶|
|迭代对数时间|      |<img src="https://latex.codecogs.com/gif.latex?O(log^*n)" title="O(log^*n)" />|    |      |
|对数时间|DLOGTIME|<img src="https://latex.codecogs.com/gif.latex?O(logn)" title="O(logn)" />|<img src="https://latex.codecogs.com/gif.latex?logn" title="logn" />,<img src="https://latex.codecogs.com/gif.latex?logn^2" title="logn^2" />|二分搜索|
|线性时间|      |<img src="https://latex.codecogs.com/gif.latex?O(n)" title="O(n)" />|n|无序数组的搜索|
|线性对数时间|      |<img src="https://latex.codecogs.com/gif.latex?O(nlogn)" title="O(nlogn)" />|<img src="https://latex.codecogs.com/gif.latex?nlogn" title="nlogn" />,<img src="https://latex.codecogs.com/gif.latex?logn!" title="logn!" />|最快的比较排序|
|二次时间|  |<img src="https://latex.codecogs.com/gif.latex?O(n^2)" title="O(n^2)" />|<img src="https://latex.codecogs.com/gif.latex?n^2" title="n^2" />|冒泡排序、插入排序|
|三次时间|  |<img src="https://latex.codecogs.com/gif.latex?O(n^3)" title="O(n^3)" />|<img src="https://latex.codecogs.com/gif.latex?n^3" title="n^3" />|矩阵乘法的基本实现，计算部分相关性|
|阶乘时间||<img src="https://latex.codecogs.com/gif.latex?O(n!)" title="O(n!)" />|<img src="https://latex.codecogs.com/gif.latex?n!" title="n!" />|通过暴力搜索解决旅行推销员问题|
-------
空间复杂度：分析排序算法中需要多少辅助内存
记做<img src="https://latex.codecogs.com/gif.latex?S(n)=O(f(n))" title="S(n)=O(f(n))" />。比如直接插入排序空间复杂度<img src="https://latex.codecogs.com/gif.latex?O(1)" title="O(1)" />。而一般递归算法就要有<img src="https://latex.codecogs.com/gif.latex?O(n)" title="O(n)" />的空间复杂度，
因为每次递归都要存储返回信息。

-----------
稳定性：若两个记录A和B的关键字相等，但是排序后AB的先后次序保持不变（稳定）

- [冒泡排序](https://zh.wikipedia.org/wiki/%E5%86%92%E6%B3%A1%E6%8E%92%E5%BA%8F)（bubble sort）— O(*n*2)
- [插入排序](https://zh.wikipedia.org/wiki/%E6%8F%92%E5%85%A5%E6%8E%92%E5%BA%8F)（insertion sort）—O(*n*2)
- [鸡尾酒排序](https://zh.wikipedia.org/wiki/%E9%B8%A1%E5%B0%BE%E9%85%92%E6%8E%92%E5%BA%8F)（cocktail sort）—O(*n*2)
- [桶排序](https://zh.wikipedia.org/wiki/%E6%A1%B6%E6%8E%92%E5%BA%8F)（bucket sort）—O(*n*)；需要O(*k*)额外空间
- [计数排序](https://zh.wikipedia.org/wiki/%E8%AE%A1%E6%95%B0%E6%8E%92%E5%BA%8F)（counting sort）—O(*n*+*k*)；需要O(*n*+*k*)额外空间
- [归并排序](https://zh.wikipedia.org/wiki/%E5%BD%92%E5%B9%B6%E6%8E%92%E5%BA%8F)（merge sort）—O(*n* log *n*)；需要O(*n*)额外空间
- 原地[归并排序](https://zh.wikipedia.org/wiki/%E5%BD%92%E5%B9%B6%E6%8E%92%E5%BA%8F)— O(*n* log2 *n*)如果使用最佳的现在版本
- [二叉排序树](https://zh.wikipedia.org/wiki/%E4%BA%8C%E5%8F%89%E6%8E%92%E5%BA%8F%E6%A0%91)排序（binary tree sort）— O(*n* log *n*)期望时间；O(*n*2)最坏时间；需要O(*n*)额外空间
- [鸽巢排序](https://zh.wikipedia.org/wiki/%E9%B8%BD%E5%B7%A2%E6%8E%92%E5%BA%8F)（pigeonhole sort）—O(*n*+*k*)；需要O(*k*)额外空间
- [基数排序](https://zh.wikipedia.org/wiki/%E5%9F%BA%E6%95%B0%E6%8E%92%E5%BA%8F)（radix sort）—O(*n*·*k*)；需要O(*n*)额外空间
- [侏儒排序](https://zh.wikipedia.org/w/index.php?title=%E4%BE%8F%E5%84%92%E6%8E%92%E5%BA%8F&action=edit&redlink=1)（gnome sort）— O(*n*2)
- [图书馆排序](https://zh.wikipedia.org/wiki/%E5%9B%BE%E4%B9%A6%E9%A6%86%E6%8E%92%E5%BA%8F)（library sort）— O(*n* log *n*)期望时间；O(*n*2)最坏时间；需要(1+ε)*n*额外空间
- [块排序](https://zh.wikipedia.org/w/index.php?title=%E5%A1%8A%E6%8E%92%E5%BA%8F&action=edit&redlink=1)（block sort）— O(*n* log *n*)

# 1.插入排序
|      |      |
| ---- | ---- |
|数据结构|数组|
|时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(n^2)" title="O(n^2)" />|
|最优时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(n)" title="O(n)" />|
|平均时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(n^2)" title="O(n^2)" />|
|空间复杂度|总<img src="https://latex.codecogs.com/gif.latex?O(n)" title="O(n)" />，需要辅助空间<img src="https://latex.codecogs.com/gif.latex?O(1)" title="O(1)" />|


## 1.1 性能分析
时间复杂度<img src="https://latex.codecogs.com/gif.latex?O(n^2)" title="O(n^2)" />,空间复杂度<img src="https://latex.codecogs.com/gif.latex?O(1)" title="O(1)" />
排序时间与输入相关：输入的元素个数；元素已排序的程度。
最佳情况，输入数组是已经排好序的数组，运行时间是`n`的线性函数；最坏情况，输入数组是逆序，运行时间是`n`的二次函数

插入排序是一种简单直观的排序算法。它的工作原理是通过构建有序序列，
对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。

## 1.2 算法描述
一般来说，插入排序都采用in-place在数组上实现。具体算法描述如下：
1. 从第一个元素开始，该元素可以认为已经被排序
2. 取出下一个元素，在已经排序的元素序列中从后向前扫描
3. 如果该元素（已排序）大于新元素，将该元素移到下一位置
4. 重复步骤3，直到找到已排序的元素小于或者等于新元素的位置
5. 将新元素插入到该位置后
6. 重复步骤2~5

# 2.冒泡排序
|      |      |
| ---- | ---- |
|数据结构|数组|
|最坏时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(n^2)" title="O(n^2)" />|
|最优时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(n)" title="O(n)" />|
|平均时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(n^2)" title="O(n^2)" />|
|空间复杂度|总共<img src="https://latex.codecogs.com/gif.latex?O(n)" title="O(n)" />，需要辅助空间<img src="https://latex.codecogs.com/gif.latex?O(1)" title="O(1)" />|

冒泡排序是一种简单的排序算法。它重复地走访过要排序的数列，一次比较两个元素，如果他们的顺序错误就把他们交换过来。
走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。

冒泡排序对n个项目需要<img src="https://latex.codecogs.com/gif.latex?O(n^2)" title="O(n^2)" />的比较次数，且可以原地排序。尽管这个算法是最简单了解和实现的排序算法之一，但它对于少数
元素之外的数列排序是很没有效率的。

冒泡排序与插入排序拥有相等的运行时间，但是两种算法在需要交换的次数却很大地不同。在最好的情况下，冒泡排序需要<img src="https://latex.codecogs.com/gif.latex?O(n^2)" title="O(n^2)" />
次交换，而插入排序只要最多<img src="https://latex.codecogs.com/gif.latex?O(n)" title="O(n)" />交换。

冒泡排序算法的运作如下：
    1. 比较相邻的元素。如果第一个比第二个大，就交换他们两个。
    2. 对每一对相邻元素作同样的工作，从开始到第一队到结尾的最后一对。这布做完后，最后的元素会是最大的数。
    3. 针对所有的元素重复以上的步骤，除了最后一个
    4. 持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。

# 3.快速排序
|      |      |
| ---- | ---- |
|数据结构|不定|
|最坏时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(n^2)" title="O(n^2)" />|
|最优时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(nlogn)" title="O(nlogn)" />|
|平均时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(nlogn)" title="O(nlogn)" />|
|空间复杂度|根据实现方式不同而不同|

在平均状况下，排序n个项目要<img src="https://latex.codecogs.com/gif.latex?O(nlogn)" title="O(nlogn)" />次比较。在最坏状况下则需要<img src="https://latex.codecogs.com/gif.latex?O(n^2)" title="O(n^2)" />,但这种状况并不常见。事实上，快速排序通常
明显比其他<img src="https://latex.codecogs.com/gif.latex?O(nlogn)" title="O(nlogn)" />算法更快，因为它的内部循环可以在大部分的架构上很有效率地被实现出来。

## 算法描述
快速排序使用分治法策略把一个序列(list)分为两个子序列(sub-lists)。
步骤：
    1. 从数列中挑出一个元素，称为“基准”(pivot)
    2. 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准大的摆在基准的后面。
    在这个分区结束之后，该基准就处于数列的中间位置。这个称为分区(partition)操作。
    3. 递归地(recursive)把小于基准值元素的子数列和大于基准值元素的子数列排序。

伪代码
表示为：
```
function quicksort(q)
     var list less, pivotList, greater
     if length(q) ≤ 1 {
         return q
     } else {
         select a pivot value pivot from q
         for each x in q except the pivot element
             if x < pivot then add x to less
             if x ≥ pivot then add x to greater
         add pivot to pivotList
         return concatenate(quicksort(less), pivotList, quicksort(greater))
     }
```

# 堆排序

|      |      |
| ---- | ---- |
|数据结构|数组|
|时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(nlogn)" title="O(nlogn)" />|
|最优时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(nlogn)" title="O(nlogn)" />|
|平均时间复杂度|<img src="https://latex.codecogs.com/gif.latex?O(nlogn)" title="O(nlogn)" />|
|空间复杂度|总的<img src="https://latex.codecogs.com/gif.latex?O(n)" title="O(n)" />,辅助<img src="https://latex.codecogs.com/gif.latex?O(1)" title="O(1)" /> |

堆排序(Heapsort)是指利用堆这种数据结构所设计的一种排序算法。堆积是一个近似完全二叉树的结构，
并同时满足堆积的性质，即子结点的键值或索引总是小于（或者大于）它的父节点。

## 堆节点的访问
通常堆是通过一维数组来实现的。在数组起始位置为0的情形中：
* 父节点i的左子节点在位置(1*i+1);
* 父节点i的右子节点在位置(2*i+2);
* 子节点i的父节点在位置floor((i-1)/2); # floor功能，即取不大于x的最大整数

## 堆的操作
在堆得数据结构中，堆中的最大值总是位于根节点(在优先队列中使用堆的话堆中的最小值位于根节点)。堆中定义以下几种操作：
* 最大堆调整(Max_Heaplfy)：将堆得末端子节点作调整，使得子节点永远小于父节点。
* 创建最大堆(Build\_Max_Heap):将堆所有数据重新排序。
* 堆排序(HeapSort)：移除位在第一个数据的根节点，并做最大堆调整的递归运算。

# 归并操作
归并操作(merge)，也叫归并算法，指的是将两个已经排序的序列合并成一个序列的操作。归并排序算法依赖归并操作。

## 迭代法
    1. 申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列
    2. 设定两个指针，最初位置分别为两个已经排序序列的起始位置
    3. 比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置
    4. 重复步骤3直到某一指针到达序列尾
    5. 将另一序列所剩下的所有元素直接复制到合并序列尾

## 递归法
原来如下(假设序列共有n个元素)：
    1. 将序列每相邻两个数字进行归并操作，形成floor(n/2)个序列，排序后每个序列包含两个元素
    2. 将上述序列再次归并，形成floor(n/4)个序列，每个序列包含四个元素
    3. 重复步骤2，直到所有元素排序完毕
