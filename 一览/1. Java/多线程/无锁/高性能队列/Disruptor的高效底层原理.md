#### 1. Disruptor组件

先看一下Disruptor中重要组件之间的协作关系



![img](http://pcc.huitogo.club/5833b00a0505618b978df3ce9351ca24)



**Ring Buffer（DataProvider）**

如其名，环形的缓冲区。曾经 RingBuffer 是 Disruptor 中的最主要的对象，但从3.0版本开始，其职责被简化为仅仅负责对通过 Disruptor 进行交换的数据（事件）进行存储和更新。在一些更高级的应用场景中，Ring Buffer 可以由用户的自定义实现来完全替代。



Sequence 通过顺序递增的序号来编号管理通过其进行交换的数据（事件），对数据(事件)的处理过程总是沿着序号逐个递增处理。一个 Sequence 用于跟踪标识某个特定的事件处理者( RingBuffer/Consumer )的处理进度。虽然一个 AtomicLong 也可以用于标识进度，但定义 Sequence 来负责该问题还有另一个目的，那就是防止不同的Sequence 之间的CPU缓存伪共享(Flase Sharing)问题。



**Sequencer**

是 Disruptor 的真正核心。此接口有两个实现类 SingleProducerSequencer、MultiProducerSequencer ，它们定义在生产者和消费者之间快速、正确地传递数据的并发算法。



**Sequence Barrier**

用于保持对RingBuffer的 main published Sequence 和Consumer依赖的其它Consumer的 Sequence 的引用。 Sequence Barrier 还定义了决定 Consumer 是否还有可处理的事件的逻辑。



**Wait Strategy**

定义 Consumer 如何进行等待下一个事件的策略。 （注：Disruptor 定义了多种不同的策略，针对不同的场景，提供了不一样的性能表现）



**Event**

在 Disruptor 的语义中，生产者和消费者之间进行交换的数据被称为事件(Event)。它不是一个被 Disruptor 定义的特定类型，而是由 Disruptor 的使用者定义并指定。



**EventProcessor**

EventProcessor 持有特定消费者(Consumer)的 Sequence，并提供用于调用事件处理实现的事件循环(Event Loop)。



**EventHandler**

Disruptor 定义的事件处理接口，由用户实现，用于处理事件，是 Consumer 的真正实现。



**Producer**

即生产者，只是泛指调用 Disruptor 发布事件的用户代码，Disruptor 没有定义特定接口或类型。



#### 2. 初始化

```
/**
 * Create a new Disruptor. Will default to {@link com.lmax.disruptor.BlockingWaitStrategy} and
 * {@link ProducerType}.MULTI
 *
 * @param eventFactory   the factory to create events in the ring buffer.
 * @param ringBufferSize the size of the ring buffer.
 * @param threadFactory  a {@link ThreadFactory} to create threads to for processors.
 */
public Disruptor(final EventFactory<T> eventFactory, final int ringBufferSize, final ThreadFactory threadFactory)
{
    this(RingBuffer.createMultiProducer(eventFactory, ringBufferSize), new BasicExecutor(threadFactory));
}
```



其中 EventFactory 即生产者，生产Event数据到RingBuffer中，ringBufferSize即环数组的长度限制，一般是 2的幂等方，方便做位运算，threadFactory 是构建线程池的线程工厂，这里线程池 是执行消费者线程的执行器



生产者分两种，多生产者MultiProducerSequencer 和 单生产者 SingleProducerSequencerPad，这里以创建多生产者为例

```
public final class MultiProducerSequencer extends AbstractSequencer{
    
    // availableBuffer tracks the state of each ringbuffer slot

    private final int[] availableBuffer;
    // 掩码 = ringBufferSize - 1，用于 和 sequence做位于运算 求下标
    private final int indexMask;
    // sequence 下标 index 的偏移量 = bufferSize 的 2的阶值
    private final int indexShift;
    
     /**
     * Construct a Sequencer with the selected wait strategy and buffer size.
     *
     * @param bufferSize   the size of the buffer that this will sequence over.
     * @param waitStrategy for those waiting on sequences.
     */
    public MultiProducerSequencer(int bufferSize, final WaitStrategy waitStrategy)
    {
        super(bufferSize, waitStrategy);
        availableBuffer = new int[bufferSize];
        indexMask = bufferSize - 1;
        // 比如 bufferSize为1024 则indexShift = 10
        indexShift = Util.log2(bufferSize);
        // 初始化 sequence 的下标的 flag值
        initialiseAvailableBuffer();
    }
```



这里waitStrategy 指的是消费者的等待策略，即当前没有可消费event消息时的 消费者线程的行为，有以下几种



![img](http://pcc.huitogo.club/4beec8629c99c6903929b7edfc61f3a9)



availableBuffer 有两种用处，一种用于多生产者情况下 避免重复写入同一个数组单元的安全缓冲地带，另一个也避免消费者 拿到还未生产成功的Event消息，原理就是无论你是读还是写 都需要在 availableBuffer 在指定sequence的位置将flag置为有效，创建Sequencer的时候会初始化availableBuffer数组中的所有元素flag值为-1

```
// 数组在内存中初始位置（偏移量）
private static final long BASE = UNSAFE.arrayBaseOffset(int[].class);
//一个数组元素占用的偏移量
private static final long SCALE = UNSAFE.arrayIndexScale(int[].class);

private void initialiseAvailableBuffer()
{
    for (int i = availableBuffer.length - 1; i != 0; i--)
    {
        setAvailableBufferValue(i, -1);
    }
    setAvailableBufferValue(0, -1);
}

private void setAvailableBufferValue(int index, int flag)
{
    long bufferAddress = BASE + (index * SCALE);
    UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag);
}
```



UNSAFE 里面是native方法，直接和jvm内存做交互，所以有更高的性能，但是使用它修改内存值的时候 得使用它的native方法，比如这里的UNSAFE.putOrderedInt，需要自己计算待修改值在内存中相对于对象的偏移量，比如这里计算修改index的元素位置是BASE + (index * SCALE)；



创建好Sequencer之后，需要初始化 RingBuffer（DataProvider）

```
private static final int BUFFER_PAD;
private static final long REF_ARRAY_BASE;
private static final int REF_ELEMENT_SHIFT;
private static final Unsafe UNSAFE = Util.getUnsafe();

private final long indexMask;
// 物理上是从entries中定位元素的 逻辑上借助Sequencer 获取元素的下标
private final Object[] entries;
protected final int bufferSize;
protected final Sequencer sequencer;

RingBufferFields(EventFactory<E> eventFactory, Sequencer sequencer)
{
    this.sequencer = sequencer;
    this.bufferSize = sequencer.getBufferSize();

    this.indexMask = bufferSize - 1;
    // 实际数组长度  =  128 / scale * 2 + bufferSize
    this.entries = new Object[sequencer.getBufferSize() + 2 * BUFFER_PAD];
    // 将entries中 bufferSize 边界里的元素值填充默认值
    fill(eventFactory);
}

static
{
    // 数组元素 占用精度（位数）
    final int scale = UNSAFE.arrayIndexScale(Object[].class);
    if (4 == scale)
    {
        REF_ELEMENT_SHIFT = 2;
    }
    else if (8 == scale)
    {
        REF_ELEMENT_SHIFT = 3;
    }
    else
    {
        throw new IllegalStateException("Unknown pointer size");
    }
    BUFFER_PAD = 128 / scale;
    // Including the buffer pad in the array base offset
    REF_ARRAY_BASE = UNSAFE.arrayBaseOffset(Object[].class) + (BUFFER_PAD << REF_ELEMENT_SHIFT);
}
```



RingBuffer 物理上存储Event消息的结构是 entries数组，但是数组是有界的，怎么让它循环存储元素达到一个无边界的效果呢，答案就是 使用Sequence做元素的 逻辑序列值，Sequence里面有个 value值，每次申请新的序列值都会新增



Sequence 存储的value值 为了避免 伪共享，使用了 行对齐的方式，即 前后各填充了 7个long值，当然获取 和 修改value值也使用了UNSAFE类

```
class LhsPadding
{
    protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding
{
    protected volatile long value;
}

class RhsPadding extends Value
{
    protected long p9, p10, p11, p12, p13, p14, p15;
}

public class Sequence extends RhsPadding
{
    static final long INITIAL_VALUE = -1L;
    private static final Unsafe UNSAFE;
    private static final long VALUE_OFFSET;

    static
    {
        UNSAFE = Util.getUnsafe();
        try
        {
            VALUE_OFFSET = UNSAFE.objectFieldOffset(Value.class.getDeclaredField("value"));
        }
        catch (final Exception e)
        {
            throw new RuntimeException(e);
        }
    }
    
    public void set(final long value)
    {
        UNSAFE.putOrderedLong(this, VALUE_OFFSET, value);
    }
}
```



初始化RingBuffer会初始化 entries数组中的Event消息默认值，这里就有个巧妙的设计了，正常生产 和 消费 都是基于空的队列 去塞/获取 数据，但是RingBuffer可是一开始就将数组塞满 默认Event事件，当然这个默认Event也是由用户自定义的EventFactory生成

```
private void fill(EventFactory<E> eventFactory)
{
    for (int i = 0; i < bufferSize; i++)
    {
        entries[BUFFER_PAD + i] = eventFactory.newInstance();
    }
}
```



实际entries的长度 = ringBufferSize + 2* BUFFER_PAD，这里赋值的时候是在中间一段 ringBufferSize的长度进行赋值，即前后都预留了 BUFFER_PAD长度的空白作为缓冲地带，BUFFER_PAD = 2^7 >>> 2 的长度，即2^5 的长度



![img](http://pcc.huitogo.club/7b86163b13cc856ced3f8af4815da24a)



#### 3. 配置消费者组

RingBuffer初始化后，我们需要配置消费者组 - 实现EventHandler的消费者们，消息者也就是事件监听器

```
public class SeckillEventConsumer implements EventHandler<SeckillEvent> {

    @Override
    public void onEvent(SeckillEvent seckillEvent, long seq, boolean bool) throws Exception {
//        seckillService.doSeckill(seckillEvent.getSeckill_id());
    }
}

disruptor.handleEventsWith(new SeckillEventConsumer());

EventProcessorInfo(消费者) = BatchEventProcessor（消费者监听线程） + EventHandler（消费事件执行器），多个消费者共用一个SequenceBarrier（序号屏障）
private final ConsumerRepository<T> consumerRepository = new ConsumerRepository<>();

EventHandlerGroup<T> createEventProcessors(
    final Sequence[] barrierSequences,
    final EventHandler<? super T>[] eventHandlers)
{
    checkNotStarted();

    final Sequence[] processorSequences = new Sequence[eventHandlers.length];
    final SequenceBarrier barrier = ringBuffer.newBarrier(barrierSequences);

    // 每有一个 EventHandler 就新增一个消费者 ,假设有n个消费者
    // 1 * SequenceBarrier + n * BatchEventProcessor + n * EventHandler = EventHandlerGroup
    // SequenceBarrier ：序号屏障, 保证消费者和生产者，消费者和消费者之间的可见性和消费速度
    // BatchEventProcessor：消费者线程，从RingBuffer里消费事件 交给EventHandler执行
    // EventHandler：消费事件执行器
    for (int i = 0, eventHandlersLength = eventHandlers.length; i < eventHandlersLength; i++)
    {
        final EventHandler<? super T> eventHandler = eventHandlers[i];

        final BatchEventProcessor<T> batchEventProcessor =
            new BatchEventProcessor<>(ringBuffer, barrier, eventHandler);

        if (exceptionHandler != null)
        {
            batchEventProcessor.setExceptionHandler(exceptionHandler);
        }

        consumerRepository.add(batchEventProcessor, eventHandler, barrier);
        processorSequences[i] = batchEventProcessor.getSequence();
    }
    // 更新消费者的 消费序列到 sequenceGroup中
    updateGatingSequencesForNextInChain(barrierSequences, processorSequences);

    return new EventHandlerGroup<>(this, consumerRepository, processorSequences);
}
```



消费者线程 执行的任务 我们在读数据的时候再认真看下，这里主要看一下 消费者各自的消费序列是怎么维护的

```
private void updateGatingSequencesForNextInChain(final Sequence[] barrierSequences, final Sequence[] processorSequences)
{
    if (processorSequences.length > 0)
    {
        ringBuffer.addGatingSequences(processorSequences);
        for (final Sequence barrierSequence : barrierSequences)
        {
            ringBuffer.removeGatingSequence(barrierSequence);
        }
        consumerRepository.unMarkEventProcessorsAsEndOfChain(barrierSequences);
    }
}
...
class SequenceGroups
{
    static <T> void addSequences(
        final T holder,
        final AtomicReferenceFieldUpdater<T, Sequence[]> updater,
        final Cursored cursor,
        final Sequence... sequencesToAdd)
    {
        long cursorSequence;
        Sequence[] updatedSequences;
        Sequence[] currentSequences;

        do
        {
            currentSequences = updater.get(holder);
            // updatedSequences = currentSequences + sequencesToAdd
            updatedSequences = copyOf(currentSequences, currentSequences.length + sequencesToAdd.length);
            cursorSequence = cursor.getCursor();

            int index = currentSequences.length;
            // sequencesToAdd 的Sequence 赋 当前生产者的消费指针位置 然后添加到 updatedSequences中
            for (Sequence sequence : sequencesToAdd)
            {
                sequence.set(cursorSequence);
                updatedSequences[index++] = sequence;
            }
        }
        while (!updater.compareAndSet(holder, currentSequences, updatedSequences));

        cursorSequence = cursor.getCursor();
        for (Sequence sequence : sequencesToAdd)
        {
            sequence.set(cursorSequence);
        }
    }
```



最终会再SequenceGroups中进行存储，在初始化 和 更新的时候 会将原数组 进行 System.copyof一份，然后借助AtomicReferenceFieldUpdater 和 CAS进行原子性 和 可见性的更新



消费者最终会存储到ConsumerRepository 中，可迭代访问

```
/**
 * Provides a repository mechanism to associate {@link EventHandler}s with {@link EventProcessor}s
 *
 * @param <T> the type of the {@link EventHandler}
 */
class ConsumerRepository<T> implements Iterable<ConsumerInfo>
{
    private final Map<EventHandler<?>, EventProcessorInfo<T>> eventProcessorInfoByEventHandler =
        new IdentityHashMap<>();
    private final Map<Sequence, ConsumerInfo> eventProcessorInfoBySequence =
        new IdentityHashMap<>();
    private final Collection<ConsumerInfo> consumerInfos = new ArrayList<>();

    public void add(
        final EventProcessor eventprocessor,
        final EventHandler<? super T> handler,
        final SequenceBarrier barrier)
    {
        final EventProcessorInfo<T> consumerInfo = new EventProcessorInfo<>(eventprocessor, handler, barrier);
        eventProcessorInfoByEventHandler.put(handler, consumerInfo);
        eventProcessorInfoBySequence.put(eventprocessor.getSequence(), consumerInfo);
        consumerInfos.add(consumerInfo);
    }
}
```



#### 4. 写数据

现在我们看一下生产者是怎么输出数据的，前面我们已经知道RingBuffer中其实每个元素都已经被我们使用EventFactory初始化了Event，那么生产者要做的事就不是new一个Event，而是定位到可生产元素的有效位置然后 塞入事件消息

```
public void publishEvent(EventTranslatorVararg<E> translator, Object... args)
{    
    // 找到有效位置
    final long sequence = sequencer.next();
    // 塞入值
    translateAndPublish(translator, sequence, args);
}
```



我们基于多生产者 MultiProducerSequencer 来看如何找到有效位置

```
/**
 * @see Sequencer#next(int)
 */
@Override
public long next(int n)
{
    if (n < 1)
    {
        throw new IllegalArgumentException("n must be > 0");
    }

    long current;
    long next;

    do
    {
        current = cursor.get();
        next = current + n;

        // 相对于当前 cursor 的 数组起点
        long wrapPoint = next - bufferSize;
        // gatingSequenceCache 最小的消费序列
        long cachedGatingSequence = gatingSequenceCache.get();

        // 限制生产者能不能继续生产，Max(待生效的序列) - Min(消费序列) 不能大于 bufferSize
        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current)
        {
            // 寻找消费者组 和 当前生产游标的 最小值
            long gatingSequence = Util.getMinimumSequence(gatingSequences, current);

            // 生产太快了 需要等一等消费者
            if (wrapPoint > gatingSequence)
            {
                LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
                continue;
            }

            gatingSequenceCache.set(gatingSequence);
        }
        else if (cursor.compareAndSet(current, next))
        {
            break;
        }
    }
    while (true);

    return next;
}
```

这里有两个地方需要注意的是

- 多个生产者申请的区间是不一样的，使用CAS操作保证线程安全
- 最快的生产者 和 最慢的消费者之间相差的序列值不能大于RingBufferSize的长度，否则生产者会一直在自旋加锁（park 1ns），这也就是生产者的等待策略



生产者申请好写入区间之后，塞入值操作 也就是一个translate操作，使用Translator将原entries中的元素值 装配成带有 事件的元素

```
private void translateAndPublish(EventTranslatorVararg<E> translator, long sequence, Object... args)
{
    try
    {
        //translate操作
        translator.translateTo(get(sequence), sequence, args);
    }
    finally
    {
        sequencer.publish(sequence);
    }
}
```



Translator是用于自定义的，需要实现EventTranslatorVararg 接口



这里有个重要点就是**怎么根据 sequence下标 在entires中寻找到实际的元素**，即get(sequence)方法

```
public E get(long sequence)
{
    return elementAt(sequence);
}

/**
 * 根据 序列 获取 在entries数组中的实际元素
 * @param sequence
 * @return
 */
@SuppressWarnings("unchecked")
protected final E elementAt(long sequence)
{
    // 基于 sequence 获取 实际 entries 中的对象 为什么要偏移  scale 的 对阶值？
    // 求的是内存的偏移量 而不仅仅只是一个下标，数据起始地址 + 元素下标 * 单个元素占用位置 = 实际数组元素在对象中的偏移量
    return (E) UNSAFE.getObject(entries, REF_ARRAY_BASE + ((sequence & indexMask) << REF_ELEMENT_SHIFT));
}
```



定位的算法就是 使用sequnce & ringBufferSize -1 寻找下标 * 数据元素的精度（偏移量） + 数组起始偏移量



塞入值之后 有个通知消费者的操作，比如消息者线程在wait的情况下 给他sign

```
/**
 * @see Sequencer#publish(long)
 */
@Override
public void publish(final long sequence)
{
    setAvailable(sequence);
    // 通知阻塞 的消费线程 有消息来了
    waitStrategy.signalAllWhenBlocking();
}

private void setAvailable(final long sequence)
{
    // calculateIndex(sequence) 取 sequence & bufferSize -1 的值
    // calculateAvailabilityFlag(sequence)  计算 sequence 溢出 bufferSize 外的 高位值，用于存储 ringBuffer的环数（轮次）
    setAvailableBufferValue(calculateIndex(sequence), calculateAvailabilityFlag(sequence));
}

private void setAvailableBufferValue(int index, int flag)
{
    long bufferAddress = BASE + (index * SCALE);
    UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag);
}
```



availableBuffer 也是之前讲的用于 生产者 和 消费者的一个 缓存数组，这里生产者发布完事件后也需要在 availableBuffer 中设置有效值，这里的算法是 将 sequence & rinfBufferSize-1 的值做下标，将sequence 偏移 ringBufferSize长度的溢出值 做flag值，这个flag值也就是 当前RingBuffer的轮次



waitStrategy 我们以BlockingWaitStrategy 为例，将消费者线程 从wait状态唤醒 继续消费

```
@Override
public void signalAllWhenBlocking()
{
    lock.lock();
    try
    {
        processorNotifyCondition.signalAll();
    }
    finally
    {
        lock.unlock();
    }
}
```



#### 5. 读数据

读数据 我们回过头来看 配置消费者组的时候 消费者线程BatchEventProcessor的行为，首先看下消费者线程是怎么启动的

```
public RingBuffer<T> start()
{
    checkOnlyStartedOnce();
    for (final ConsumerInfo consumerInfo : consumerRepository)
    {
        consumerInfo.start(executor);
    }

    return ringBuffer;
}
```



在Distruptor启动的时候 就会执行，执行器就是我们初始化RingBuffer的时候 配置的Executor线程池



消费者线程的行为如下，这里有一些钩子函数的处理 ，比如Lifestyle，类比Spring里面的，我们就跳过了

```
/**
 * It is ok to have another thread rerun this method after a halt().
 *
 * @throws IllegalStateException if this object instance is already running in a thread
 */
@Override
public void run()
{
    if (!running.compareAndSet(IDLE, RUNNING))
    {
        if (running.get() == RUNNING)
        {
            throw new IllegalStateException("Thread is already running");
        }
    }
    sequenceBarrier.clearAlert();

    notifyStart();

    try
    {
        if (running.get() == HALTED)
        {
            return;
        }

        T event = null;
        // 获取当前待消费的下一个序列
        long nextSequence = sequence.get() + 1L;

        while (true)
        {
            try
            {
                // 返回生产者 生产的最大有效序列
                final long availableSequence = sequenceBarrier.waitFor(nextSequence);
                if (batchStartAware != null)
                {
                    batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
                }

                while (nextSequence <= availableSequence)
                {
                    event = dataProvider.get(nextSequence);
                    eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
                    nextSequence++;
                }

                sequence.set(availableSequence);
            }
            catch (final TimeoutException e)
            {
                notifyTimeout(sequence.get());
            }
            catch (final AlertException ex)
            {
                if (running.get() != RUNNING)
                {
                    break;
                }
            }
            catch (final Throwable ex)
            {
                exceptionHandler.handleEventException(ex, nextSequence, event);
                sequence.set(nextSequence);
                nextSequence++;
            }
        }
    }
    finally
    {
        notifyShutdown();
        running.set(IDLE);
    }
}
```



消费者线程 也是分为两步走，第一步 取可消费的序列值 第二步 根据序列值找到实际元素进行消费



消费者是使用SequenceBarrier序列屏障 来保障 消费者 和 生产者的序列冲突 和 一致性，取可消费的序列值即sequenceBarrier.waitFor(nextSequence)

```
public long waitFor(final long sequence)
        throws AlertException, InterruptedException, TimeoutException {
    checkAlert();

    long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);

    if (availableSequence < sequence) {
        return availableSequence;
    }

    /*
     * 查询 nextSequence-availableSequence 区间段之间连续发布的最大序号。多生产者模式下可能是不连续的
     *     多生产者模式下{@link Sequencer#next(int)} next是预分配的，因此可能部分数据还未被填充。
     */
    return sequencer.getHighestPublishedSequence(sequence, availableSequence);
}
```



虽然你希望消费nextSequence序列，但还是得按实际的可消费序列来，这里还是以BlockingWaitStrategy 为例

```
@Override
public long waitFor(long sequence, Sequence cursorSequence, Sequence dependentSequence, SequenceBarrier barrier)
	throws AlertException, InterruptedException
{
long availableSequence;
if (cursorSequence.get() < sequence)
{
	lock.lock();
	try
	{
		// 如果当前 待消费序列 一直小于当前生产的序列值 则一直 await
		while (cursorSequence.get() < sequence)
		{
			barrier.checkAlert();
			processorNotifyCondition.await();
		}
	}
	finally
	{
		lock.unlock();
	}
}
// dependentSequence 当前消费者 消费序列时 所依赖的 序列
// 正常是 生产者的序列   当有消费者消费优先关系时 则是 其他消费者的消费队列
while ((availableSequence = dependentSequence.get()) < sequence)
{
	barrier.checkAlert();
	// 自旋
	ThreadHints.onSpinWait();
}
return availableSequence;
}
```



这里可以看出消费者 可消费的序列 和 两个序列值有关，一个是生产者的游标、另一个是 依赖的序列。

第一个好理解，如果你希望消费的序列值大于我生产者所在的游标，那么说明 队列中无可消费Event，就需要按WaitStategy来等待；

第二个依赖的序列分两种，正常情况下也是依赖生产者的游标，如果消费者出现层级关系的时候，也就是依赖其他消费者先执行后才能执行，则需要依赖其他消费者的消费序列，不能比他们大，如果大的话就需要自旋进行原地等待



获取到可消费序列值后，还需要验证它的有效性，即利用我们之前提到的availableBuffer

```
@Override
public long getHighestPublishedSequence(long lowerBound, long availableSequence)
{
    for (long sequence = lowerBound; sequence <= availableSequence; sequence++)
    {
        if (!isAvailable(sequence))
        {
            return sequence - 1;
        }
    }

    return availableSequence;
}

public boolean isAvailable(long sequence)
{
    int index = calculateIndex(sequence);
    int flag = calculateAvailabilityFlag(sequence);
    long bufferAddress = (index * SCALE) + BASE;
    return UNSAFE.getIntVolatile(availableBuffer, bufferAddress) == flag;
}
```



这里会验证 从期望消费序列sequence到 可消费序列availableSequence之间 元素的有效性，有的元素可能生产者还未来得及更改flag值，则暂时不能消费，最后会返回最大的可消费序列值



#### 6. 总结

这里简单基于源码总结下 Distruptor高效的原因

- 无锁 多线程情况下 CAS操作 性能优于 加锁
- UNSAFE，基于UNSAFE的内存操作，大大减少了 更新值的操作时间
- 位运算，也是在计算环节取得了优势
- 采用了数组的底层存储结构，RingBuffer内对数组都是查找和修改操作，大大发挥了数组的优势