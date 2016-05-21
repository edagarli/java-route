# Disruptor使用入门

在最近的项目中看到同事使用到了Disruptor，以前在ifeve上看到过关于Disruptor的文章，但是没有深入研究，现在项目中用到了，就借这个机会对这个并发编程框架进行深入学习。项目中使用到的是disruptor-2.10.4，所以下面分析到的Disruptor的代码是这个版本的。
[并发编程网](http://ifeve.com/disruptor/)介绍Disruptor的文章是disruptor1.0版本，所以有一些术语在2.0版本上已经没有了或者被替代了。


### Disruptor术语

github上Disruptor的wiki对Disruptor中的术语进行了解释，在看Disruptor的过程中，对于几个其他的类，觉得有必要与这些术语放到一起，就加进来了。

* RingBuffer 经常被看作Disruptor最主要的组件，然而从3.0开始RingBuffer仅仅负责存储和更新在Disruptor中流通的数据。对一些特殊的使用场景能够被用户(使用其他数据结构)完全替代。
* Sequence Disruptor使用Sequence来表示一个特殊组件处理的序号。和Disruptor一样，每个消费者(EventProcessor)都维持着一个Sequence。大部分的并发代码依赖这些Sequence值的运转，因此Sequence支持多种当前AtomicLong类的特性。事实上，这两者之间唯一的区别是Sequence包含额外的功能来阻止Sequence和其他值之间的共享。
* Sequencer 这是Disruptor真正的核心。实现了这个接口的两种生产者（单生产者和多生产者）均实现了所有的并发算法，为了在生产者和消费者之间进行准确快速的数据传递。
* SequenceBarrier 由Sequencer生成，并且包含了已经发布的Sequence的引用，这些的Sequence源于Sequencer和一些独立的消费者的Sequence。它包含了决定是否有供消费者来消费的Event的逻辑。
* WaitStrategy：它决定了一个消费者将如何等待生产者将Event置入Disruptor。
* Event 从生产者到消费者过程中所处理的数据单元。Disruptor中没有代码表示Event，因为它完全是由用户定义的。
* EventProcessor 主要的事件循环，用于处理Disruptor中的Event，并且拥有消费者的Sequence。它有一个实现类是BatchEventProcessor，包含了event loop有效的实现，并且将回调到一个EventHandler接口的实现对象。
* EventHandler 由用户实现并且代表了Disruptor中的一个消费者的接口。
* Producer 由用户实现，它调用RingBuffer来插入事件(Event)，Disruptor中没有相应的实现代码，由用户实现。
* WorkProcessor 确保每个sequence只被一个processor消费，在同一个WorkPool中的处理多个WorkProcessor不会消费同样的sequence。
* WorkerPool 一个WorkProcessor池，其中WorkProcessor将消费Sequence，所以任务可以在实现WorkHandler接口的worker吃间移交
* LifecycleAware 当BatchEventProcessor启动和停止时，于实现这个接口用于接收通知。


### Disruptor印象

初看Disruptor，给人的印象就是RingBuffer是其核心，生产者向RingBuffer中写入元素，消费者从RingBuffer中消费元素，如下图：

![](http://img.blog.csdn.net/20140803174131225)

这就是Disruptor最简单的模型。其中的RingBuffer被组织成要给环形队列，但它与我们在常常使用的队列又不一样，这个队列大小固定，且每个元素槽都以一个整数进行编号，RingBuffer中只有一个游标维护着一个指向下一个可用位置的序号，生产者每次向RingBuffer中写入一个元素时都需要向RingBuffer申请一个可写入的序列号，如果此时RingBuffer中有可用节点，RingBuffer就向生产者返回这个可用节点的序号，如果没有，那么就等待。同样消费者消费的元素序号也必须是生产者已经写入了的元素序号。
那么Disruptor是如何实现这些逻辑的呢？先来看一个Disruptor的使用示例


### Disruptor使用示例

不适用Disruptor的dsl，直接使用Disruptor中的类来完成。
```
//RingBuffer中存储的单元
public class IntEvent {
    private int value = -1;

    public int getValue() {
        return value;
    }

    public void setValue(int value) {
        this.value = value;
    }

    public String toString() {
        return String.valueOf(value);
    }

    public static EventFactory<IntEvent> INT_ENEVT_FACTORY = new EventFactory<IntEvent>() {
        public IntEvent newInstance() {
            return new IntEvent();
        }
    };
}
//生产者
public class IntEventProducer implements WorkHandler<IntEvent> {

    private int seq = 0;
    public void onEvent(IntEvent event) throws Exception {
        System.out.println("produced " + seq);
        event.setValue(++seq);
    }

}
//消费者
public class IntEventProcessor implements WorkHandler<IntEvent> {

    public void onEvent(IntEvent event) throws Exception {
        System.out.println(event.getValue());
        event.setValue(1);
    }

}

public class DisruptorTest {

    public static void main(String[] args) throws InterruptedException {
        //创建一个RingBuffer对象
        RingBuffer<IntEvent> ringBuffer = new RingBuffer<IntEvent>(IntEvent.INT_ENEVT_FACTORY,
            new SingleThreadedClaimStrategy(16),
            new SleepingWaitStrategy());

        SequenceBarrier sequenceBarrier = ringBuffer.newBarrier();
        IntEventProducer[] producers = new IntEventProducer[1];
        for (int i = 0; i < producers.length; i++) {
            producers[i] = new IntEventProducer();
        }
        WorkerPool<IntEvent> crawler = new WorkerPool<IntEvent>(ringBuffer,
            sequenceBarrier,
            new IntEventExceptionHandler(),
            producers);
        SequenceBarrier sb = ringBuffer.newBarrier(crawler.getWorkerSequences());
        IntEventProcessor[] processors = new IntEventProcessor[1];
        for (int i = 0; i < processors.length; i++) {
            processors[i] = new IntEventProcessor();
        }

        WorkerPool<IntEvent> applier = new WorkerPool<IntEvent>(ringBuffer,sb,
            new IntEventExceptionHandler(),
            processors);
        List<Sequence> gatingSequences = new ArrayList<Sequence>();
        for(Sequence s : crawler.getWorkerSequences()) {
            gatingSequences.add(s);
        }
        for(Sequence s : applier.getWorkerSequences()) {
            gatingSequences.add(s);
        }
ringBuffer.setGatingSequences(gatingSequences.toArray(new Sequence[gatingSequences.size()]));
        ThreadPoolExecutor executor = new ThreadPoolExecutor(7,7,10,TimeUnit.MINUTES,new LinkedBlockingQueue<Runnable>(5));
        crawler.start(executor);
        applier.start(executor);

        while (true) {
            Thread.sleep(1000);
            long lastSeq = ringBuffer.next();
            ringBuffer.publish(lastSeq);
        }
    }
}

class IntEventExceptionHandler implements ExceptionHandler {
    public void handleEventException(Throwable ex, long sequence, Object event) {}
    public void handleOnStartException(Throwable ex) {}
    public void handleOnShutdownException(Throwable ex) {}
}
```

在上面的代码中IntEvent类就是术语中的Event，IntEventProducer对应Producer，IntEventProcessor对应着EventProcessor，也就是消费者，但是IntEventProcessor类并不是实现的IntEventProcessor接口，这个下面会分析到。
下面从main方法开始分析。
首先是RingBuffer的创建，RingBuffer<IntEvent> ringBuffer = new RingBuffer<IntEvent>(IntEvent.INT_ENEVT_FACTORY,new SingleThreadedClaimStrategy(16), new SleepingWaitStrategy());RingBuffer的构造方法的第一个参数是实现了EventFactory接口的类，主要作用是创建IntEvent对象，在创建RingBuffer对象时，第二个参数是ClaimStrategy，生产者通过ClaimStrategy 来申请下一个可用节点，第三个参数是WaitStrategy的实现类，它定义了消费者的等待策略。
下面一段代码是初始化生产者的过程

```
SequenceBarrier sequenceBarrier = ringBuffer.newBarrier();
IntEventProducer[] producers = new IntEventProducer[1];
for (int i = 0; i < producers.length; i++) {
    producers[i] = new IntEventProducer();
}
WorkerPool<IntEvent> crawler = new WorkerPool<IntEvent>(ringBuffer,
sequenceBarrier,new IntEventExceptionHandler(), producers);

```
ringBuffer.newBarrier()返回的是一个SequenceBarrier对象，Barrier顾名思义就是障碍的意思，这个障碍阻止了生产者越过其所能到达的ringbuffer中的位置。RingBuffer在执行过程中会维持一个游标cursor，所以生产者从RingBuffer中获取到的的游标必须小于等于这个cursor。
然后就是消费者的初始化：
```
SequenceBarrier sb = ringBuffer.newBarrier(crawler.getWorkerSequences());
IntEventProcessor[] processors = new IntEventProcessor[1];
for (int i = 0; i < processors.length; i++) {
    processors[i] = new IntEventProcessor();
}

WorkerPool<IntEvent> applier = new WorkerPool<IntEvent>(ringBuffer,sb,new IntEventExceptionHandler(),processors);
```

消费者初始化也需要设置一个SequenceBarrier对象，这个SequenceBarrier对象指明了消费者可以消费的元素序号，如果消费者的游标大于这个序号，那么消费者必须以WaitStrategy定义的策略等待。
生产者和消费者创建完成后下一步是设置RingBuffer的一个变量gatingSequences，gatingSequences的作用是防止生产者覆盖还未被消费者消费的元素，假设一个RingBuffer的大小为8，消费者消费速度较慢，那么RingBuffer可能是满的，当生产者向RingBuffer申请下一个可用序号时，还未被消费者消费的序号就不能被覆盖，所以RingBuffer就不能给生产者返回可用序号，此时消费者线程就进入等待，在这种情况下RingBuffer会检查当前申请的序号是否大于gatingSequences中的最小序号，如果当前申请的序号大于最小序号，那么生产者就等待。代码如下：

```
List<Sequence> gatingSequences = new ArrayList<Sequence>();
for(Sequence s : crawler.getWorkerSequences()) {
    gatingSequences.add(s);
}
for(Sequence s : applier.getWorkerSequences()) {
    gatingSequences.add(s);
}     ringBuffer.setGatingSequences(gatingSequences.toArray(new Sequence[gatingSequences.size()]));

```
这里之所以加入了生产者的序号，是因为在多生产者的情况下，一个生产者不能覆盖另一个生产者已经申请的序号。
然后下面一段的代码就是启动Disruptor线程来执行生产者和消费者线程。