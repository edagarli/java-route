# 稳定的Java堆 VS 动荡的Java堆
一般来说，稳定的堆大小对垃圾回收是有利的。获得一个稳定的堆大小的方法是使-Xms 和-Xmx 的大小一致，即最大堆和最小堆 (初始堆) 一样。如果这样设置，系统在运行时堆大小理论上是恒定的，稳定的堆空间可以减少 GC 的次数。因此，很多服务端应用都会将最大堆和最小堆设置为相同的数值。但是，一个不稳定的堆并非毫无用处。稳定的堆大小虽然可以减少 GC 次数，但同时也增加了每次 GC 的时间。让堆大小在一个区间中震荡，在系统不需要使用大内存时，压缩堆空间，使 GC 应对一个较小的堆，可以加快单次 GC 的速度。基于这样的考虑，JVM 还提供了两个参数用于压缩和扩展堆空间。
-XX:MinHeapFreeRatio 参数用来设置堆空间最小空闲比例，默认值是 40。当堆空间的空闲内存小于这个数值时，JVM 便会扩展堆空间。
-XX:MaxHeapFreeRatio 参数用来设置堆空间最大空闲比例，默认值是 70。当堆空间的空闲内存大于这个数值时，便会压缩堆空间，得到一个较小的堆。
当-Xmx 和-Xms 相等时，-XX:MinHeapFreeRatio 和-XX:MaxHeapFreeRatio 两个参数无效。

清单 14. 堆大小设置

```
import java.util.Vector;

public class HeapSize {
 public static void main(String args[]) throws InterruptedException{
 Vector v = new Vector();
 while(true){
 byte[] b = new byte[1024*1024];
 v.add(b);
 if(v.size() == 10){
 v = new Vector();
 }
 Thread.sleep(1);
 }
 }
}
```

清单 14 所示代码是测试-XX:MinHeapFreeRatio 和-XX:MaxHeapFreeRatio 的作用，设置运行参数为-XX:+PrintGCDetails -Xms10M -Xmx40M -XX:MinHeapFreeRatio=40 -XX:MaxHeapFreeRatio=50 时，输出如清单 15 所示。

清单 15. 修改运行参数后清单 14 输出
```
[GC [DefNew: 2418K->178K(3072K), 0.0034827 secs] 2418K->2226K(9920K),
 0.0035249 secs] [Times: user=0.00 sys=0.00, real=0.03 secs] 
[GC [DefNew: 2312K->0K(3072K), 0.0028263 secs] 4360K->4274K(9920K), 
0.0029905 secs] [Times: user=0.00 sys=0.00, real=0.03 secs] 
[GC [DefNew: 2068K->0K(3072K), 0.0024363 secs] 6342K->6322K(9920K),
0.0024836 secs] [Times: user=0.00 sys=0.00, real=0.03 secs] 
[GC [DefNew: 2061K->0K(3072K), 0.0017376 secs][Tenured: 8370K->8370K(8904K),
0.1392692 secs] 8384K->8370K(11976K), [Perm : 374K->374K(12288K)],
0.1411363 secs] [Times: user=0.00 sys=0.02, real=0.16 secs] 
[GC [DefNew: 5138K->0K(6336K), 0.0038237 secs] 13508K->13490K(20288K),
0.0038632 secs] [Times: user=0.00 sys=0.00, real=0.03 secs]
```

改用参数：-XX:+PrintGCDetails -Xms40M -Xmx40M -XX:MinHeapFreeRatio=40 -XX:MaxHeapFreeRatio=50，运行输出如清单 16 所示。

清单 16. 再次修改运行参数后清单 14 输出
```
[GC [DefNew: 10678K->178K(12288K), 0.0019448 secs] 10678K->178K(39616K), 
 0.0019851 secs] [Times: user=0.00 sys=0.00, real=0.03 secs] 
[GC [DefNew: 10751K->178K(12288K), 0.0010295 secs] 10751K->178K(39616K),
0.0010697 secs] [Times: user=0.00 sys=0.00, real=0.02 secs] 
[GC [DefNew: 10493K->178K(12288K), 0.0008301 secs] 10493K->178K(39616K),
0.0008672 secs] [Times: user=0.00 sys=0.00, real=0.02 secs] 
[GC [DefNew: 10467K->178K(12288K), 0.0008522 secs] 10467K->178K(39616K),
0.0008905 secs] [Times: user=0.00 sys=0.00, real=0.02 secs] 
[GC [DefNew: 10450K->178K(12288K), 0.0008964 secs] 10450K->178K(39616K),
0.0009339 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[GC [DefNew: 10439K->178K(12288K), 0.0009876 secs] 10439K->178K(39616K),
0.0010279 secs] [Times: user=0.00 sys=0.00, real=0.02 secs]
```

从清单 16 可以看出，此时堆空间的垃圾回收稳定在一个固定的范围。在一个稳定的堆中，堆空间大小始终不变，每次 GC 时，都要应对一个 40MB 的空间。因此，虽然 GC 次数减小了，但是单次 GC 速度不如一个震荡的堆。

