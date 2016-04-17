# 无锁的并行计算之Amino框架

Amino框架是一个采用无锁方式实现并行计算的框架，可惜的是，网上关于Amino框架的介绍甚少。根据所掌握的资料，稍微总结一下：

1. 锁机制到无锁机制
锁机制可以确保程序和数据的线程安全，但是锁是一种阻塞式的同步方式，无论是ReentrantLock、synchronized，还是Semaphore，都受到核心资源的限制。为避免这个问题，便提出了无锁的同步机制。

2. 基于Compare-and-swap（CAS） 算法的无锁并发控制方法
CAS算法过程是：它包含三个参数CAS(V,E,N)，V表示内存位置目前的值，E表示期望的原值，N表示新值。当处理器要更新一个内存位置的值的时候，它首先将V与E进行对比（要知道在多处理的时候，你要更新的内存位置上的值V有可能被其他处理更新过，而你全然不知），如果V与E相同，那么就将V设为N，将N写入内存；否则，就什么也不做，不写入新的值（现在最新的做法是定义内存值的版本号，根据版本号的改变来判断内存值是否被修改）。CAS 的价值所在就在于它是在硬件级别实现的，速度那是相当的快。
这种无锁并发控制方法像极了乐观锁。

在JDK的java.util.concurrent.atomic包下，有一组使用无锁方式实现的原子操作，包括AtomicInteger、AtomicIntegerArray、AtomicLong、AtomicLongArray等。以AtomicInteger为例，其中的getAndSet()方法是这样实现CAS的：

```
public final int getAndSet(int newValue){  
    for(;;){  
        int current=get();  
        if(compareAndSet(current,newValue))  
            return current;  
    }  
}  
```

3. 引出Amino框架
该组件将提供一套免锁的集合类（LockFreeVector、LockFreeList、LockFreeSet等）、树结构、图结构。因为这些数据结构采用免锁的运算法则来生成，所以，它们将拥有基本的免锁组件的特性，如可以避免不同类型的死锁，不同类型的线程初始化顺序等。 

Amino 还提供了一些非常有用的并行计算模式，包括 Master-Worker、Map-reduce、Divide and conquer, Pipeline 等。 

Amino并没有加入到官方jdk中，因此需要自行下载、导入。
下载地址：http://sourceforge.net/projects/amino-cbbs/files/

4. 性能测试
下面，在64位4G内存的Windows7测试下Vector与LockFreeVector、CopyOnWriteArrayList与LockFreeList在增加、删除操作的区别。从结果可以看出，后者性能大大提升。

测试代码：
public class CopyList {  
    public static void test1(AccessListTread t, String name){  
        CounterPoolExecutor exe0=new CounterPoolExecutor(2000,2000,0L,TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>());  
        exe0.TASK_COUNT=8000;  
        exe0.funcname=name;  
        exe0.startTime=System.currentTimeMillis();  
        for(int i=0;i<exe0.TASK_COUNT;i++)  
            exe0.submit(t);//测试数据：8000  
        exe0.shutdown();  
    }  
    public static void main(String[] args) {  
        AccessListTread t=new AccessListTread();  
        t.initCopyOnWriteArrayList();  
        test1(t,"testCopyOnWriteArrayList");  
        t.initVector();  
        test1(t,"testVector");  
        t.initLockFreeList();  
        test1(t,"testLockFreeList");  
        t.initLockFreeVector();  
        test1(t,"testLockFreeVector");  
    }  
      
}  
class CounterPoolExecutor extends ThreadPoolExecutor{  
    public AtomicInteger count=new AtomicInteger(0);//统计次数  
    public long startTime=0;  
    public String funcname="";  
    public int TASK_COUNT=0;  
    public CounterPoolExecutor(int corePoolSize, int maximumPoolSize,  
            long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {  
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);  
    }  
    @Override  
    protected void afterExecute(Runnable r,Throwable t){  
        int l=count.addAndGet(1);  
        if(l==TASK_COUNT){  
            System.out.println(funcname+"spend time:"+(System.currentTimeMillis()-startTime));  
        }  
    }  
}  
  
  
  
class AccessListTread implements Runnable{  
    Random rand=new Random();  
    List list;  
    public AccessListTread() {  
    }  
    @Override  
    public void run() {  
        try {  
            for(int i=0;i<1000;i++)  
//              getList(rand.nextInt(1000));  
                handleList(rand.nextInt(1000));  
            Thread.sleep(rand.nextInt(100));  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
  
    private Object getList(int nextInt) {  
        return list.get(nextInt);  
    }  
    private Object handleList(int index) {  
        list.add(index);  
        list.remove(index%list.size());  
        return null;  
    }  
    //test  
    public void initCopyOnWriteArrayList(){  
        list=new CopyOnWriteArrayList();  
        for(int i=0;i<1000;i++)  
            list.add(i);  
    }  
    public void initVector(){  
        list=new Vector();  
        for(int i=0;i<1000;i++)  
            list.add(i);  
    }  
    public void initLockFreeList(){  
        list=new LockFreeList();  
        for(int i=0;i<1000;i++)  
            list.add(i);  
    }  
    public void initLockFreeVector(){  
        list=new LockFreeVector();  
        for(int i=0;i<1000;i++)  
            list.add(i);  
    }  
}  