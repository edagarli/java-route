# 如何让大对象进入年老代

我们在大部分情况下都会选择将对象分配在年轻代。但是，对于占用内存较多的大对象而言，它的选择可能就不是这样的。因为大对象出现在年轻代很可能扰乱年轻代 GC，并破坏年轻代原有的对象结构。因为尝试在年轻代分配大对象，很可能导致空间不足，为了有足够的空间容纳大对象，JVM 不得不将年轻代中的年轻对象挪到年老代。因为大对象占用空间多，所以可能需要移动大量小的年轻对象进入年老代，这对 GC 相当不利。基于以上原因，可以将大对象直接分配到年老代，保持年轻代对象结构的完整性，这样可以提高 GC 的效率。如果一个大对象同时又是一个短命的对象，假设这种情况出现很频繁，那对于 GC 来说会是一场灾难。原本应该用于存放永久对象的年老代，被短命的对象塞满，这也意味着对堆空间进行了洗牌，扰乱了分代内存回收的基本思路。因此，在软件开发过程中，应该尽可能避免使用短命的大对象。可以使用参数-XX:PetenureSizeThreshold 设置大对象直接进入年老代的阈值。当对象的大小超过这个值时，将直接在年老代分配。参数-XX:PetenureSizeThreshold 只对串行收集器和年轻代并行收集器有效，并行回收收集器不识别这个参数。

清单 8. 创建一个大对象
```
public class BigObj2Old {
 public static void main(String[] args){
 byte[] b;
 b = new byte[1024*1024];//分配一个 1MB 的对象
 }
}
```
使用 JVM 参数-XX:+PrintGCDetails –Xmx20M –Xms20MB 运行，可以得到清单 9 所示日志输出。

清单 9. 清单 8 运行输出
```
Heap
 def new generation total 6144K, used 1378K [0x35c10000, 0x362b0000, 0x362b0000)
 eden space 5504K, 25% used [0x35c10000, 0x35d689e8, 0x36170000)
 from space 640K, 0% used [0x36170000, 0x36170000, 0x36210000)
 to space 640K, 0% used [0x36210000, 0x36210000, 0x362b0000)
 tenured generation total 13696K, used 0K [0x362b0000, 0x37010000, 0x37010000)
 the space 13696K, 0% used [0x362b0000, 0x362b0000, 0x362b0200, 0x37010000)
 compacting perm gen total 12288K, used 374K [0x37010000, 0x37c10000, 0x3b010000)
 the space 12288K, 3% used [0x37010000, 0x3706dac8, 0x3706dc00, 0x37c10000)
 ro space 10240K, 51% used [0x3b010000, 0x3b543000, 0x3b543000, 0x3ba10000)
 rw space 12288K, 55% used [0x3ba10000, 0x3c0ae4f8, 0x3c0ae600, 0x3c610000)
 ```
 
可以看到该对象被分配在了年轻代，占用了 25%的空间。如果需要将 1MB 以上的对象直接在年老代分配，设置-XX:PetenureSizeThreshold=1000000，程序运行后输出如清单 10 所示。

清单 10. 修改运行参数后清单 8 输出
```
Heap
 def new generation total 6144K, used 354K [0x35c10000, 0x362b0000, 0x362b0000)
 eden space 5504K, 6% used [0x35c10000, 0x35c689d8, 0x36170000)
 from space 640K, 0% used [0x36170000, 0x36170000, 0x36210000)
 to space 640K, 0% used [0x36210000, 0x36210000, 0x362b0000)
 tenured generation total 13696K, used 1024K [0x362b0000, 0x37010000, 0x37010000)
 the space 13696K, 7% used [0x362b0000, 0x363b0010, 0x363b0200, 0x37010000)
 compacting perm gen total 12288K, used 374K [0x37010000, 0x37c10000, 0x3b010000)
 the space 12288K, 3% used [0x37010000, 0x3706dac8, 0x3706dc00, 0x37c10000)
 ro space 10240K, 51% used [0x3b010000, 0x3b543000, 0x3b543000, 0x3ba10000)
 rw space 12288K, 55% used [0x3ba10000, 0x3c0ae4f8, 0x3c0ae600, 0x3c610000)
 ```
清单 10 里面可以看到当满 1MB 时进入到了年老代。
