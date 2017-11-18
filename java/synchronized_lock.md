# synchronized(this)



# ReentranLock
ReentranLock，字面意思就是可重入锁，它支持同一个线程对资源的重复加锁，也是我们平时在处理java并发情况下用的最多的同步组件之一
(volatile, synchronized)

|公平锁|多线程并发获取锁时，按照等待时间排序获得锁，等待时间最久的线程先获得锁|
|:----:|:----|
|非公平锁 |获取锁时不考虑排队问题，直接尝试获取锁|
