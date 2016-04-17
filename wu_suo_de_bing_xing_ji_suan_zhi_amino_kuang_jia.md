# 无锁的并行计算之Amino框架

Amino框架是一个采用无锁方式实现并行计算的框架，可惜的是，网上关于Amino框架的介绍甚少。根据所掌握的资料，稍微总结一下：

1. 锁机制到无锁机制
锁机制可以确保程序和数据的线程安全，但是锁是一种阻塞式的同步方式，无论是ReentrantLock、synchronized，还是Semaphore，都受到核心资源的限制。为避免这个问题，便提出了无锁的同步机制。

2. 基于Compare-and-swap（CAS） 算法的无锁并发控制方法
CAS算法过程是：它包含三个参数CAS(V,E,N)，V表示内存位置目前的值，E表示期望的原值，N表示新值。当处理器要更新一个内存位置的值的时候，它首先将V与E进行对比（要知道在多处理的时候，你要更新的内存位置上的值V有可能被其他处理更新过，而你全然不知），如果V与E相同，那么就将V设为N，将N写入内存；否则，就什么也不做，不写入新的值（现在最新的做法是定义内存值的版本号，根据版本号的改变来判断内存值是否被修改）。CAS 的价值所在就在于它是在硬件级别实现的，速度那是相当的快。
这种无锁并发控制方法像极了乐观锁。

在JDK的java.util.concurrent.atomic包下，有一组使用无锁方式实现的原子操作，包括AtomicInteger、AtomicIntegerArray、AtomicLong、AtomicLongArray等。以AtomicInteger为例，其中的getAndSet()方法是这样实现CAS的：


public final int getAndSet(int newValue){  
    for(;;){  
        int current=get();  
        if(compareAndSet(current,newValue))  
            return current;  
    }  
}  