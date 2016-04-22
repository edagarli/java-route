# 如何将新对象预留在年轻代

众所周知，由于 Full GC 的成本远远高于 Minor GC，因此某些情况下需要尽可能将对象分配在年轻代，这在很多情况下是一个明智的选择。虽然在大部分情况下，JVM 会尝试在 Eden 区分配对象，但是由于空间紧张等问题，很可能不得不将部分年轻对象提前向年老代压缩。因此，在 JVM 参数调优时可以为应用程序分配一个合理的年轻代空间，以最大限度避免新对象直接进入年老代的情况发生。清单 1 所示代码尝试分配 4MB 内存空间，观察一下它的内存使用情况。

清单 1. 相同大小内存分配
```
public class PutInEden {
 public static void main(String[] args){
 byte[] b1,b2,b3,b4;//定义变量
 b1=new byte[1024*1024];//分配 1MB 堆空间，考察堆空间的使用情况
 b2=new byte[1024*1024];
 b3=new byte[1024*1024];
 b4=new byte[1024*1024];
 }
}
```
使用 JVM 参数-XX:+PrintGCDetails -Xmx20M -Xms20M 运行清单 1 所示代码，输出如清单 2 所示。

清单 2. 清单 1 运行输出
```
[GC [DefNew: 5504K->640K(6144K), 0.0114236 secs] 5504K->5352K(19840K), 
   0.0114595 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[GC [DefNew: 6144K->640K(6144K), 0.0131261 secs] 10856K->10782K(19840K),
0.0131612 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[GC [DefNew: 6144K->6144K(6144K), 0.0000170 secs][Tenured: 10142K->13695K(13696K),
0.1069249 secs] 16286K->15966K(19840K), [Perm : 376K->376K(12288K)],
0.1070058 secs] [Times: user=0.03 sys=0.00, real=0.11 secs] 
[Full GC [Tenured: 13695K->13695K(13696K), 0.0302067 secs] 19839K->19595K(19840K), 
[Perm : 376K->376K(12288K)], 0.0302635 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured: 13695K->13695K(13696K), 0.0311986 secs] 19839K->19839K(19840K), 
[Perm : 376K->376K(12288K)], 0.0312515 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured: 13695K->13695K(13696K), 0.0358821 secs] 19839K->19825K(19840K), 
[Perm : 376K->371K(12288K)], 0.0359315 secs] [Times: user=0.05 sys=0.00, real=0.05 secs] 
[Full GC [Tenured: 13695K->13695K(13696K), 0.0283080 secs] 19839K->19839K(19840K),
[Perm : 371K->371K(12288K)], 0.0283723 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC [Tenured: 13695K->13695K(13696K), 0.0284469 secs] 19839K->19839K(19840K),
[Perm : 371K->371K(12288K)], 0.0284990 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured: 13695K->13695K(13696K), 0.0283005 secs] 19839K->19839K(19840K),
[Perm : 371K->371K(12288K)], 0.0283475 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured: 13695K->13695K(13696K), 0.0287757 secs] 19839K->19839K(19840K),
[Perm : 371K->371K(12288K)], 0.0288294 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured: 13695K->13695K(13696K), 0.0288219 secs] 19839K->19839K(19840K), 
[Perm : 371K->371K(12288K)], 0.0288709 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured: 13695K->13695K(13696K), 0.0293071 secs] 19839K->19839K(19840K),
[Perm : 371K->371K(12288K)], 0.0293607 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured: 13695K->13695K(13696K), 0.0356141 secs] 19839K->19838K(19840K),
[Perm : 371K->371K(12288K)], 0.0356654 secs] [Times: user=0.01 sys=0.00, real=0.03 secs] 
Heap
 def new generation total 6144K, used 6143K [0x35c10000, 0x362b0000, 0x362b0000)
 eden space 5504K, 100% used [0x35c10000, 0x36170000, 0x36170000)
 from space 640K, 99% used [0x36170000, 0x3620fc80, 0x36210000)
 to space 640K, 0% used [0x36210000, 0x36210000, 0x362b0000)
 tenured generation total 13696K, used 13695K [0x362b0000, 0x37010000, 0x37010000)
 the space 13696K, 99% used [0x362b0000, 0x3700fff8, 0x37010000, 0x37010000)
 compacting perm gen total 12288K, used 371K [0x37010000, 0x37c10000, 0x3b010000)
 the space 12288K, 3% used [0x37010000, 0x3706cd20, 0x3706ce00, 0x37c10000)
 ro space 10240K, 51% used [0x3b010000, 0x3b543000, 0x3b543000, 0x3ba10000)
 rw space 12288K, 55% used [0x3ba10000, 0x3c0ae4f8, 0x3c0ae600, 0x3c610000)
 ```
清单 2 所示的日志输出显示年轻代 Eden 的大小有 5MB 左右。分配足够大的年轻代空间，使用 JVM 参数-XX:+PrintGCDetails -Xmx20M -Xms20M-Xmn6M 运行清单 1 所示代码，输出如清单 3 所示。

清单 3. 增大 Eden 大小后清单 1 运行输出
```
[GC [DefNew: 4992K->576K(5568K), 0.0116036 secs] 4992K->4829K(19904K), 
 0.0116439 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[GC [DefNew: 5568K->576K(5568K), 0.0130929 secs] 9821K->9653K(19904K), 
0.0131336 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[GC [DefNew: 5568K->575K(5568K), 0.0154148 secs] 14645K->14500K(19904K),
0.0154531 secs] [Times: user=0.00 sys=0.01, real=0.01 secs] 
[GC [DefNew: 5567K->5567K(5568K), 0.0000197 secs][Tenured: 13924K->14335K(14336K),
0.0330724 secs] 19492K->19265K(19904K), [Perm : 376K->376K(12288K)],
0.0331624 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured: 14335K->14335K(14336K), 0.0292459 secs] 19903K->19902K(19904K),
[Perm : 376K->376K(12288K)], 0.0293000 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured: 14335K->14335K(14336K), 0.0278675 secs] 19903K->19903K(19904K),
[Perm : 376K->376K(12288K)], 0.0279215 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured: 14335K->14335K(14336K), 0.0348408 secs] 19903K->19889K(19904K),
[Perm : 376K->371K(12288K)], 0.0348945 secs] [Times: user=0.05 sys=0.00, real=0.05 secs] 
[Full GC [Tenured: 14335K->14335K(14336K), 0.0299813 secs] 19903K->19903K(19904K),
[Perm : 371K->371K(12288K)], 0.0300349 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC [Tenured: 14335K->14335K(14336K), 0.0298178 secs] 19903K->19903K(19904K),
[Perm : 371K->371K(12288K)], 0.0298688 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space[Full GC [Tenured: 
14335K->14335K(14336K), 0.0294953 secs] 19903K->19903K(19904K),
[Perm : 371K->371K(12288K)], 0.0295474 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenured
: 14335K->14335K(14336K), 0.0287742 secs] 19903K->19903K(19904K), 
[Perm : 371K->371K(12288K)], 0.0288239 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC [Tenuredat GCTimeTest.main(GCTimeTest.java:16)
: 14335K->14335K(14336K), 0.0287102 secs] 19903K->19903K(19904K),
[Perm : 371K->371K(12288K)], 0.0287627 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
Heap
 def new generation total 5568K, used 5567K [0x35c10000, 0x36210000, 0x36210000)
 eden space 4992K, 100% used [0x35c10000, 0x360f0000, 0x360f0000)
 from space 576K, 99% used [0x36180000, 0x3620ffe8, 0x36210000)
 to space 576K, 0% used [0x360f0000, 0x360f0000, 0x36180000)
 tenured generation total 14336K, used 14335K [0x36210000, 0x37010000, 0x37010000)
 the space 14336K, 99% used [0x36210000, 0x3700ffd8, 0x37010000, 0x37010000)
 compacting perm gen total 12288K, used 371K [0x37010000, 0x37c10000, 0x3b010000)
 the space 12288K, 3% used [0x37010000, 0x3706ce28, 0x3706d000, 0x37c10000)
 ro space 10240K, 51% used [0x3b010000, 0x3b543000, 0x3b543000, 0x3ba10000)
 rw space 12288K, 55% used [0x3ba10000, 0x3c0ae4f8, 0x3c0ae600, 0x3c610000)
 ```
 
通过清单 2 和清单 3 对比，可以发现通过设置一个较大的年轻代预留新对象，设置合理的 Survivor 区并且提供 Survivor 区的使用率，可以将年轻对象保存在年轻代。一般来说，Survivor 区的空间不够，或者占用量达到 50%时，就会使对象进入年老代 (不管它的年龄有多大)。清单 4 创建了 3 个对象，分别分配一定的内存空间。
清单 4. 不同大小内存分配
public class PutInEden2 {
 public static void main(String[] args){
 byte[] b1,b2,b3;
 b1=new byte[1024*512];//分配 0.5MB 堆空间
 b2=new byte[1024*1024*4];//分配 4MB 堆空间
 b3=new byte[1024*1024*4];
 b3=null; //使 b3 可以被回收
 b3=new byte[1024*1024*4];//分配 4MB 堆空间
 }
}
使用参数-XX:+PrintGCDetails -Xmx1000M -Xms500M -Xmn100M -XX:SurvivorRatio=8 运行清单 4 所示代码，输出如清单 5 所示。
清单 5. 清单 4 运行输出
Heap
 def new generation total 92160K, used 11878K [0x0f010000, 0x15410000, 0x15410000)
 eden space 81920K, 2% used [0x0f010000, 0x0f1a9a20, 0x14010000)
 from space 10240K, 99% used [0x14a10000, 0x1540fff8, 0x15410000)
 to space 10240K, 0% used [0x14010000, 0x14010000, 0x14a10000)
 tenured generation total 409600K, used 86434K [0x15410000, 0x2e410000, 0x4d810000)
 the space 409600K, 21% used [0x15410000, 0x1a878b18, 0x1a878c00, 0x2e410000)
 compacting perm gen total 12288K, used 2062K [0x4d810000, 0x4e410000, 0x51810000)
 the space 12288K, 16% used [0x4d810000, 0x4da13b18, 0x4da13c00, 0x4e410000)
No shared spaces configured.
清单 5 输出的日志显示，年轻代分配了 8M，年老代也分配了 8M。我们可以尝试加上-XX:TargetSurvivorRatio=90 参数，这样可以提高 from 区的利用率，使 from 区使用到 90%时，再将对象送入年老代，运行清单 4 代码，输出如清单 6 所示。
清单 6. 修改运行参数后清单 4 输出
Heap
 def new generation total 9216K, used 9215K [0x35c10000, 0x36610000, 0x36610000)
 eden space 8192K, 100% used [0x35c10000, 0x36410000, 0x36410000)
 from space 1024K, 99% used [0x36510000, 0x3660fc50, 0x36610000)
 to space 1024K, 0% used [0x36410000, 0x36410000, 0x36510000)
 tenured generation total 10240K, used 10239K [0x36610000, 0x37010000, 0x37010000)
 the space 10240K, 99% used [0x36610000, 0x3700ff70, 0x37010000, 0x37010000)
 compacting perm gen total 12288K, used 371K [0x37010000, 0x37c10000, 0x3b010000)
 the space 12288K, 3% used [0x37010000, 0x3706cd90, 0x3706ce00, 0x37c10000)
 ro space 10240K, 51% used [0x3b010000, 0x3b543000, 0x3b543000, 0x3ba10000)
 rw space 12288K, 55% used [0x3ba10000, 0x3c0ae4f8, 0x3c0ae600, 0x3c610000)
如果将 SurvivorRatio 设置为 2，将 b1 对象预存在年轻代。输出如清单 7 所示。
清单 7. 再次修改运行参数后清单 4 输出
Heap
 def new generation total 7680K, used 7679K [0x35c10000, 0x36610000, 0x36610000)
 eden space 5120K, 100% used [0x35c10000, 0x36110000, 0x36110000)
 from space 2560K, 99% used [0x36110000, 0x3638fff0, 0x36390000)
 to space 2560K, 0% used [0x36390000, 0x36390000, 0x36610000)
 tenured generation total 10240K, used 10239K [0x36610000, 0x37010000, 0x37010000)
 the space 10240K, 99% used [0x36610000, 0x3700fff0, 0x37010000, 0x37010000)
 compacting perm gen total 12288K, used 371K [0x37010000, 0x37c10000, 0x3b010000)
 the space 12288K, 3% used [0x37010000, 0x3706ce28, 0x3706d000, 0x37c10000)
 ro space 10240K, 51% used [0x3b010000, 0x3b543000, 0x3b543000, 0x3ba10000)
rw space 12288K, 55% used [0x3ba10000, 0x3c0ae4f8, 0x3c0ae600, 0x3c610000)