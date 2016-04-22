# 尝试使用大的内存分页

CPU 是通过寻址来访问内存的。32 位 CPU 的寻址宽度是 0~0xFFFFFFFF ，计算后得到的大小是 4G，也就是说可支持的物理内存最大是 4G。但在实践过程中，碰到了这样的问题，程序需要使用 4G 内存，而可用物理内存小于 4G，导致程序不得不降低内存占用。为了解决此类问题，现代 CPU 引入了 MMU（Memory Management Unit 内存管理单元）。MMU 的核心思想是利用虚拟地址替代物理地址，即 CPU 寻址时使用虚址，由 MMU 负责将虚址映射为物理地址。MMU 的引入，解决了对物理内存的限制，对程序来说，就像自己在使用 4G 内存一样。内存分页 (Paging) 是在使用 MMU 的基础上，提出的一种内存管理机制。它将虚拟地址和物理地址按固定大小（4K）分割成页 (page) 和页帧 (page frame)，并保证页与页帧的大小相同。这种机制，从数据结构上，保证了访问内存的高效，并使 OS 能支持非连续性的内存分配。在程序内存不够用时，还可以将不常用的物理内存页转移到其他存储设备上，比如磁盘，这就是大家耳熟能详的虚拟内存。
在 Solaris 系统中，JVM 可以支持 Large Page Size 的使用。使用大的内存分页可以增强 CPU 的内存寻址能力，从而提升系统的性能。
```
java –Xmx2506m –Xms2506m –Xmn1536m –Xss128k –XX:++UseParallelGC
 –XX:ParallelGCThreads=20 –XX:+UseParallelOldGC –XX:+LargePageSizeInBytes=256m
 ```
 
–XX:+LargePageSizeInBytes：设置大页的大小。
过大的内存分页会导致 JVM 在计算 Heap 内部分区（perm, new, old）内存占用比例时，会出现超出正常值的划分，最坏情况下某个区会多占用一个页的大小。