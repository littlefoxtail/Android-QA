- 跨平台，c++编写，可以支持多平台
- 跨进程，通过文件共享可以实现多个进程共享内存，实现进程通信
- 高性能，实现用户空间和内核空间的零拷贝，速度快且节约内存等
- 高稳定，页中断保护神，由操作系统实现的
# 函数介绍
```cpp
void* mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset);
```
-   addr 代表映射的虚拟内存起始地址；
-   length 代表该映射长度；
-   prot 描述了这块新的内存区域的访问权限；
-   flags 描述了该映射的类型；
-   fd 代表文件描述符；
-   offset 代表文件内的偏移值。
mmap的强大之处在于，它可以根据参数配置，用于创建共享内存，从而提高文件映射区域的IO效率，实现IO零拷贝。
# MMAP背后的保护神
内存页：在页式虚拟存储器中，会在虚拟存储空间和物理主存储空间都分割为一个个固定大小的页，为线程分配内存是也是以页为单位。比如页的大小为4k，那么4GB存储空间就需要4GB/4KB=1M条记录，即有100多万个4Kb的页，内存页中，当用户发生文件读写时，内核会申请一个内存页与文件进行读写操作。
![[Pasted image 20220730114546.png]]
这时如果内存页中没有数据，就会发生一种中断机制，它就叫缺页中断。

---
mmap函数调用后，在分配时只是建立了进程虚拟地址空间，并没有分配虚拟内存对应的物理内存，当访问这些没有建立映射关系的虚拟内存时，CPU加载指令发现代码时缺失的，就触发了缺页中断，中断后，内核会通过检查虚拟内存地址所在区域，发现存在内存映射，就可以通过虚拟内存地址计算文件偏移，定位到内存所缺的页对应的文件的页，由内核启动磁盘IO，将对应的页从磁盘加载到内存中。保护mmap能顺利进行。
# 四种场景分配原理
![[Pasted image 20220730121032.png]]
