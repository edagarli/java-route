# 如何设置对象进入年老代的年龄

堆中的每一个对象都有自己的年龄。一般情况下，年轻对象存放在年轻代，年老对象存放在年老代。为了做到这点，虚拟机为每个对象都维护一个年龄。如果对象在 Eden 区，经过一次 GC 后依然存活，则被移动到 Survivor 区中，对象年龄加 1。以后，如果对象每经过一次 GC 依然存活，则年龄再加 1。当对象年龄达到阈值时，就移入年老代，成为老年对象。这个阈值的最大值可以通过参数-XX:MaxTenuringThreshold 来设置，默认值是 15。虽然-XX:MaxTenuringThreshold 的值可能是 15 或者更大，但这不意味着新对象非要达到这个年龄才能进入年老代。事实上，对象实际进入年老代的年龄是虚拟机在运行时根据内存使用情况动态计算的，这个参数指定的是阈值年龄的最大值。即，实际晋升年老代年龄等于动态计算所得的年龄与-XX:MaxTenuringThreshold 中较小的那个。清单 11 所示代码为 3 个对象申请了若干内存。

清单 11. 申请内存
public class MaxTenuringThreshold {
 public static void main(String args[]){
 byte[] b1,b2,b3;
 b1 = new byte[1024*512];
 b2 = new byte[1024*1024*2];
 b3 = new byte[1024*1024*4];
 b3 = null;
 b3 = new byte[1024*1024*4];
 }
}
参数设置为：-XX:+PrintGCDetails -Xmx20M -Xms20M -Xmn10M -XX:SurvivorRatio=2
运行清单 11 所示代码，输出如清单 12 所示。
清单 12. 清单 11 运行输出
[GC [DefNew: 2986K->690K(7680K), 0.0246816 secs] 2986K->2738K(17920K),
 0.0247226 secs] [Times: user=0.00 sys=0.02, real=0.03 secs] 
[GC [DefNew: 4786K->690K(7680K), 0.0016073 secs] 6834K->2738K(17920K), 
0.0016436 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation total 7680K, used 4888K [0x35c10000, 0x36610000, 0x36610000)
 eden space 5120K, 82% used [0x35c10000, 0x36029a18, 0x36110000)
 from space 2560K, 26% used [0x36110000, 0x361bc950, 0x36390000)
 to space 2560K, 0% used [0x36390000, 0x36390000, 0x36610000)
 tenured generation total 10240K, used 2048K [0x36610000, 0x37010000, 0x37010000)
 the space 10240K, 20% used [0x36610000, 0x36810010, 0x36810200, 0x37010000)
 compacting perm gen total 12288K, used 374K [0x37010000, 0x37c10000, 0x3b010000)
 the space 12288K, 3% used [0x37010000, 0x3706db50, 0x3706dc00, 0x37c10000)
 ro space 10240K, 51% used [0x3b010000, 0x3b543000, 0x3b543000, 0x3ba10000)
 rw space 12288K, 55% used [0x3ba10000, 0x3c0ae4f8, 0x3c0ae600, 0x3c610000)
更改参数为-XX:+PrintGCDetails -Xmx20M -Xms20M -Xmn10M -XX:SurvivorRatio=2 -XX:MaxTenuringThreshold=1，运行清单 11 所示代码，输出如清单 13 所示。
清单 13. 修改运行参数后清单 11 输出
[GC [DefNew: 2986K->690K(7680K), 0.0047778 secs] 2986K->2738K(17920K),
 0.0048161 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC [DefNew: 4888K->0K(7680K), 0.0016271 secs] 6936K->2738K(17920K),
0.0016630 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation total 7680K, used 4198K [0x35c10000, 0x36610000, 0x36610000)
 eden space 5120K, 82% used [0x35c10000, 0x36029a18, 0x36110000)
 from space 2560K, 0% used [0x36110000, 0x36110088, 0x36390000)
 to space 2560K, 0% used [0x36390000, 0x36390000, 0x36610000)
 tenured generation total 10240K, used 2738K [0x36610000, 0x37010000, 0x37010000)
 the space 10240K, 26% used [0x36610000, 0x368bc890, 0x368bca00, 0x37010000)
 compacting perm gen total 12288K, used 374K [0x37010000, 0x37c10000, 0x3b010000)
 the space 12288K, 3% used [0x37010000, 0x3706db50, 0x3706dc00, 0x37c10000)
 ro space 10240K, 51% used [0x3b010000, 0x3b543000, 0x3b543000, 0x3ba10000)
 rw space 12288K, 55% used [0x3ba10000, 0x3c0ae4f8, 0x3c0ae600, 0x3c610000)
清单 13 所示，第一次运行时 b1 对象在程序结束后依然保存在年轻代。第二次运行前，我们减小了对象晋升年老代的年龄，设置为 1。即，所有经过一次 GC 的对象都可以直接进入年老代。程序运行后，可以发现 b1 对象已经被分配到年老代。如果希望对象尽可能长时间地停留在年轻代，可以设置一个较大的阈值。