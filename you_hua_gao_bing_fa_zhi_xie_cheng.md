# 优化高并发之协程

  现在的操作系统都是支持多任务的，多任务可通过多进程或多线程的方式去实现，进程和线程的对比就不在这里说了，在多任务的调度上操作系统采取抢占式和协作式两种方式，抢占式是指操作系统给每个任务一定的执行时间片，在到达这个时间片后如任务仍然未释放对CPU的占用，那么操作系统将强制释放，这是目前多数操作系统采取的方式；协作式是指操作系统按照任务的顺序来分配CPU，每个任务执行过程中除非其主动释放，否则将一直占据CPU，这种方式非常值得注意的是一旦有任务占据CPU不放，会导致其他任务”饿死”的现象，因此操作系统确实不太适合采用这种方式。

  说完操作系统多任务的调度方式后，来看看通常程序是如何实现支持高并发的，一种就是典型的基于操作系统提供的多进程或多线程机制，每个任务占据一个进程或一个线程，当任务中有IO等待等动作时，则将进程或线程放入待调度队列中，这种方式是目前大多数程序采取的方式，这种方式的坏处在于如想支持高的并发量，就不得不创建很多的进程或线程，而进程和线程都是要消耗不少系统资源的，另外一方面，进程或线程创建太多后，操作系统需要花费很多的时间在进程或线程的切换上，切换动作需要做状态保持和恢复，这也会消耗掉很多的系统资源；另外一种方式则是每个任务不完全占据一个进程或线程，当任务执行过程中需要进行IO等待等动作时，任务则将其所占据的进程或线程释放，以便其他任务使用这个进程或线程，这种方式的好处在于可以减少所需要的原生的进程或线程数，并且由于操作系统不需要做进程或线程的切换，而是自行来实现任务的切换，其成本会较操作系统切换低，这种方式也就是本文的重点，Coroutine方式，又称协程方式，这种方式在目前的大多数语言中都有支持。

 各种语言在实现Coroutine方式的支持时，多数都采用了Actor Model来实现，Actor Model简单来说就是每个任务就是一个Actor，Actor之间通过消息传递的方式来进行交互，而不采用共享的方式，Actor可以看做是一个轻量级的进程或线程，通常在一台4G内存的机器上，创建几十万个Actor是毫无问题的，Actor支持Continuations，即对于如下代码：

                         Actor

                           act方法

                            进行一些处理

                            创建并执行另外一个Actor

                            通过消息box阻塞获取另一个Actor执行的结果

                            继续基于这个结果进行一些处理

  在支持Continuations的情况下，可以做到消息box阻塞时并不是进程或线程级的阻塞，而只是Actor本身的阻塞，并且在阻塞时可将所占据的进程或线程释放给其他Actor使用，Actor Model实现最典型的就是erLang了。

 对于Java应用而言，传统方式下为了支持高并发，由于一个线程只能用于处理一个请求，即使是线程中其实有很多IO中断、锁等待也同样如此，因此通常的做法是通过启动很多的线程来支撑高并发，但当线程过多时，就造成了CPU需要消耗不少的时间在线程的切换上，从而出现瓶颈，按照上面对Coroutine的描述，Coroutine的方式理论上而言能够大幅度的提升Java应用所能支撑的并发量。

 Java尚不能从语言层次上支持Coroutine，在java中使用协程，可以使用协程框架，Kilim就是一种比较流行的协程框架。

Kilim是由剑桥的两位博士开发的一个用于在Java中使用Coroutine的框架，Kilim基于Java语法。( 具体了解可看这个，http://www.malhar.net/sriram/kilim/  或 http://www.ibm.com/developerworks/cn/java/j-javadev2-7.html)

<img width="487" alt="2016-04-17 21 29 45" src="https://cloud.githubusercontent.com/assets/5710228/14587299/9fb2d152-04e3-11e6-8f97-c8ea545a7b3e.png">

经测试，当线程数增加时，系统时耗激增。由于操作系统本地线程数量的限制，无法正常运行线程数8000和10000的测试代码。而使用协程的方式，其增长曲线显得十分平缓。使用协程，可以让系统以更低的成本，支持更高的并行度。

总结而言，采用Coroutine方式可以很好的绕开需要启动太多线程来支撑高并发出现的瓶颈，提高Java应用所能支撑的并发量，但在开发模式上也会带来变化，并且需要特别注意不能造成线程被阻塞的现象，从开发易用和透明迁移现有Java应用两个角度而言目前Coroutine方式还有很多不足，但相信随着越来越多的人在Java中使用Coroutine，其易用性必然是能够得到提升的。

相关资料:
 http://en.wikipedia.org/wiki/Computer_multitasking
 http://en.wikipedia.org/wiki/Coroutine
http://en.wikipedia.org/wiki/Actor_model
http://en.wikipedia.org/wiki/Continuation
http://lamp.epfl.ch/~phaller/doc/haller07coord.pdf
http://www.scala-lang.org/sites/default/files/odersky/jmlc06.pdf
 http://www.malhar.net/sriram/kilim/kilim_ecoop08.pdf 
http://lamp.epfl.ch/~phaller/doc/ScalaActors.pdf
http://blog.csdn.net/kobejayandy/article/details/41412787
http://www.blogjava.net/BlueDavy/archive/2010/01/28/311148.html
