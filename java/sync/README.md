# 同步

几种同步方式：
1. 使用`synchronized`关键字：可以在方法或代码块中使用，以防止多个线程同时访问共享资源。
2. 使用`java.util.concurrent`包中的锁：提供了不同类型的锁，如`ReentrantLock`和`ReadWriteLock`，可以用于同步线程。
3. 使用`java.util.concurrent`包中的信号量：信号量可以用于限制可以访问某些资源的线程数。
4. 使用`java.util.concurrent.atomic`包中的原子类：原子类提供了一组线程安全的变量，可以用于多线程环境中进行原子操作。

通常情况下的使用的同步方法：
1.  使用`synchronized`关键字：
	- 当您需要将方法或代码块完全保护起来，以防止多个线程同时访问时，可以使用`synchronized`关键字。
	- 当您需要在线程之间传递信息或数据时，可以使用`synchronized`关键字。

2.  使用`java.util.concurrent`包中的锁：
	- 当您需要更精细的控制来同步线程时，可以使用`java.util.concurrent`包中的锁。 例如，可以使用`ReentrantLock`来实现可重入的互斥锁。
	- 当您需要在读和写操作之间同步线程时，可以使用`ReadWriteLock`。

3.  使用`java.util.concurrent`包中的信号量：
	-  当您需要限制可以访问某些资源的线程数时，可以使用信号量。 例如，您可以使用信号量来限制对数据库的并发访问。

4.  使用`java.util.concurrent.atomic`包中的原子类：
	- 当您需要在多线程环境中进行原子操作时，可以使用原子类。 例如，您可以使用`AtomicInteger`来实现线程安全的计数器。

[死锁](deadlock.md)
[ReentrantLock](reentrantLock.md)
[Synchronized](synchronized.md)
[volatile](volatile.md)
[Semaphore信号量](semaphore.md)

## 线程状态转换

### 新建（NEW）

创建后尚未启动

### 可运行

可能正在运行，也可能正在等待CPU时间片。
包含了操作系统线程状态中的Running和Ready。

### 阻塞（Blocked）

等待获取一个排它锁，如果其线程释放了锁就会结束此状态。

### 无限期等待（Waiting）

等待其它线程显式地唤醒，否则不会被分配 CPU 时间片。

|进入方法	|退出方法|
|:--:|:--:|
|没有设置 Timeout 参数的 Object.wait() 方法	|Object.notify() / Object.notifyAll()|
|没有设置 Timeout 参数的 Thread.join() 方法	|被调用的线程执行完毕|
|LockSupport.park() 方法	|LockSupport.unpark(Thread)|

### 限期等待（Timed Waiting）

无需等待其它线程显式地唤醒，在一定时间之后会被系统自动唤醒。

调用 Thread.sleep() 方法使线程进入限期等待状态时，常常用“使一个线程睡眠”进行描述。

调用 Object.wait() 方法使线程进入限期等待或者无限期等待时，常常用“挂起一个线程”进行描述。

睡眠和挂起是用来描述行为，而阻塞和等待用来描述状态。

阻塞和等待的区别在于，阻塞是被动的，它是在等待获取一个排它锁。而等待是主动的，通过调用 Thread.sleep() 和 Object.wait() 等方法进入。

|进入方法|退出方法|
|:--:|:--:|
|Thread.sleep() 方法|时间结束|
|设置了 Timeout 参数的 Object.wait() 方法|时间结束 / Object.notify() / Object.notifyAll()|
|设置了 Timeout 参数的 Thread.join() 方法|时间结束 / 被调用的线程执行完毕|
|LockSupport.parkNanos() 方法|LockSupport.unpark(Thread)|
|LockSupport.parkUntil() 方法|LockSupport.unpark(Thread)|

### 死亡（Terminated）

可以是线程结束任务之后自己结束，或者产生了异常而结束。

## 基础线程机制

### Executor

Executor 管理多个异步任务的执行，而无需程序员显式地管理线程的生命周期。这里的异步是指多个任务的执行互不干扰，不需要进行同步操作。

### Daemon

守护线程是程序运行时在后台提供服务的线程，不属于程序中不可或缺的部分。
当所有非守护线程结束时，程序也就终止，同时会杀死所有守护线程。
main() 属于非守护线程。
使用 setDaemon() 方法将一个线程设置为守护线程。

## 中断

一个线程执行完毕之后会自动结束，如果在运行过程中发生异常也会提前结束。

### InterruptedException

通过调用一个线程的 interrupt() 来中断该线程，如果该线程处于阻塞、限期等待或者无限期等待状态，那么就会抛出 InterruptedException，从而提前结束该线程。但是不能中断 I/O 阻塞和 synchronized 锁阻塞。

对于以下代码，在 main() 中启动一个线程之后再中断它，由于线程中调用了 Thread.sleep() 方法，因此会抛出一个 InterruptedException，从而提前结束线程，不执行之后的语句。

### interrupted

如果一个线程的 run() 方法执行一个无限循环，并且没有执行 sleep() 等会抛出 InterruptedException 的操作，那么调用线程的 interrupt() 方法就无法使线程提前结束。

但是调用 interrupt() 方法会设置线程的中断标记，此时调用 interrupted() 方法会返回 true。因此可以在循环体中使用 interrupted() 方法来判断线程是否处于中断状态，从而提前结束线程。

## 互斥同步

Java提供了两种锁机制来控制多个线程对共享资源的互斥访问，第一个是JVM实现的sychronized，而另一个是JDK实现的ReentrantLock。

## J.U.C-AQS

java.util.concurrent(J.U.C)大大提高了并发性能，AQS被认为是J.U.C的核心。

## Java内存模型

Java 内存模型试图屏蔽各种硬件和操作系统的内存访问差异，以实现让 Java 程序在各种平台下都能达到一致的内存访问效果。
