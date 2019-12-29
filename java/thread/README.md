# 线程相关

- [android中的线程性能](thread_performance.md)
- [threadlocal的原理](threadlocal.md)
- [futuretask的原理](futuretask.md)
- [thread priority](thread_priority.md)
- [AsyncTask](asynctask.md)
- [ExecutorService中的关闭](executorservice.md)

## 多线程如何正确优雅的中断线程

![basethread100](/img/basethread100.png)

停止一个线程意味着在任务处理之前停掉正在做的操作

- interrupt()方法该怎么用呢？interrupt()其本身并不是一个强制打断线程的方法，其仅仅会修改线程的interrupt标志位，然后让线程自行去读标志位，自行判断是否需要中断
- 使用终端信号终端（非阻塞状态）的线程：中断线程最好的，最受推荐的方式是，使用共享变量（shared variable）发出信号，告诉线程必须停止正在运行的任务。线程必须周期性的核查这一变量，然后有秩序地中止任务
  - 使用volatile的风险就是可能终结不了阻塞的状态，如果线程被阻塞，它便不能核查共享变量，也就不能停止，这在许多情况下会发生。

## Thread And Object

### wait和notify基本

作用：阻塞、唤醒、遇到中断
特点性质：
1. 用的必须先拥有monitor锁
2. notify只能唤醒其中一个 notifyall全部唤醒
3. 属于object，object是所有对象的父类，意味着任何对象都可以调用它
4. wait和notify相对于来说比较底层


