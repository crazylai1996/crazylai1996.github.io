---
title: 认识高性能队列——Disruptor
date: 2023-05-01 19:55:00
tags: 内存队列
---

## Disruptor是什么

Disruptor是一个由英国外汇交易公司LMAX研发并开源的高性能的有界内存队列，其主要用于在线程之间完成数据的传递。[github地址](https://github.com/LMAX-Exchange/disruptor)
那么，以高性能著称的Disruptor到底有多快呢？

<!--more-->

我将常用的2种线程安全队列（ArrayBlockingQueue和LinkedBlockingQueue）与Disruptor作了个简单对比，场景是启动两个线程，一个线程往队列填充自增数字，另一个线程取数字进行累加，其对比结果如下：

```
1000w
ArrayBlockingQueue耗时：927ms
LinkedBlockingQueue耗时：1495ms
Disruptor耗时：598ms
5000w
ArrayBlockingQueue耗时：4044ms
LinkedBlockingQueue耗时：11145ms
Disruptor耗时：2824ms
1e
ArrayBlockingQueue耗时：7514ms
LinkedBlockingQueue耗时：23144ms
Disruptor耗时：4668ms
```
可以看到，Disruptor在速度上较其他两个队列有着明显的优势。



## 为什么可以这么快
### 内存预分配
在Disruptor里，底层存储为数组结构，而事件（Event）作为真实数据的一个载体，在初始化时会调用预设的EventFactory创建对应数量的Event填充数组，加上其环形数组的设计，数组中的Event对象可以很方便地实现复用，这在一定程度可以减少GC的次数，提升了性能。
```
private void fill(EventFactory<E> eventFactory){
    for (int i = 0; i < bufferSize; i++){
        entries[BUFFER_PAD + i] = eventFactory.newInstance();
    }
}
```
### 消除“伪共享”，充分利用硬件缓存
#### 什么是“伪共享”
每个CPU核心都有自己独立的cache和寄存器，主存与CPU之间存在着多级cache，L3，L2，L1，而越靠近CPU核心，速度也越快，为也提高处理速度，处理器不直接与主存通信，主存的访问首先会进入cache，所有的修改默认会异步刷新到主存。同时在多核心处理器下，为了保证各个核心的缓存是一致的，会实现缓存一致性协议。
而伪共享指的是由于共享缓存行（通常为64个字节）导致缓存无效的场景：

![cpu_cache](http://storage.laixiaoming.space/blog/cpu_cache.jpg)

就上图场景而言，线程1和线程2运行分别运行在两个核心上，线程1对putIndex读写，线程2对takeIndex读写，由于putIndex与takeIndex内存的相邻性，在加载到缓存时将被读到同一个缓存行中，而由于对其中一个变量的写操作会使缓存回写到主存，造成整个缓存行的失效，这也导致了同处于同一个缓存行的其他变量的缓存失效。

#### 它是如何被消除的
一方面，底层采用数组结构，CPU在加载数据时，会根据空间局部性原理，把相邻的数据一起加载进来，由于由于数组上结构的内存分配是连续的，也就能更好地利用CPU的缓存；
另一方面，通过增加无意义变量，增大变量间的间隔，使得一个变量可以独占一个缓存行，以空间换取时间（注： Java 8 可以使用@Contended注解，配合JVM参数-XX:-RestrictContended，来消除“伪共享”）：
```
class LhsPadding
{
	//7*8个字节
    protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding
{
    protected volatile long value;
}

class RhsPadding extends Value
{
	//7*8个字节
    protected long p9, p10, p11, p12, p13, p14, p15;
}
```

### 无锁数据结构RingBuffer

![ringbuffer](http://storage.laixiaoming.space/blog/ringbuffer.jpg)

RingBuffer作为Disruptor的底层数据结构，其内部有一个cursor变量，表示当前可读的最大下标，cursor是Sequence类的一个对象，其内部维护了一个long类型的value成员，value使用了volatile修饰，在不使用锁的前提下保证了线程之间的可见性，并通过Unsafe工具封装了对value变量的CAS系列操作。
关于volatile变量，有以下两个特性：
可见性：对一个volatile变量读，总能看到（任意线程）对这个变量的最后写入；
原子性：对任意单个volatile变量的读/写具有原子性；

```
public class Sequence extends RhsPadding
{
	static final long INITIAL_VALUE = -1L;
    private static final Unsafe UNSAFE;
    private static final long VALUE_OFFSET;
	...
}
```

#### 数据写入
RingBuffer数据的写入分为两个阶段，在第一阶段会先申请下一个可写入节点（cursor+1），多写入者模式下通过CAS操作移动cursor，来保存线程安全性；第二阶段，数据提交，提交时为保证顺序写，需要保证cursor追上当前提交的写入位置。
写入成功后，再调用具体的WaitStrategy实现通知其他消费线程

![ringbuffer_write](http://storage.laixiaoming.space/blog/ringbuffer_write.jpg)

#### 数据读取
在读取数据的时候，多个消费者可以同时消费，每个消费者都会维护有一个读取位置，在没有可读数据时，通过具体的WaitStrategy进行等待（阻塞等待或自旋等）。

![ringbuffer_read](http://storage.laixiaoming.space/blog/ringbuffer_read.jpg)

## 简单上手(生产者-消费者模型)

```
public class DisruptorStart {

    public static void main(String[] args) throws Exception {
        // RingBuffer大小，2的幂次
        int bufferSize = 1024;

        // 创建Disruptor
        Disruptor<LongEvent> disruptor = new Disruptor<>(
                LongEvent::new,
                bufferSize,
                DaemonThreadFactory.INSTANCE);

        // 事件消费
        disruptor.handleEventsWith((event, sequence, endOfBatch) -> System.out.println("Event: " + event));

        // 启动
        disruptor.start();

        // 拿到RingBuffer，用于向队列传输数据
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

        ByteBuffer bb = ByteBuffer.allocate(8);
        for (long l = 0; true; l++) {
            bb.putLong(0, l);
            //往队列填充数据
            ringBuffer.publishEvent((event, sequence, buffer) -> event.set(buffer.getLong(0)), bb);
            Thread.sleep(1000);
        }
    }

}

```
参考：
[并发框架Disruptor译文](https://ifeve.com/disruptor)
[高性能队列——Disruptor](https://tech.meituan.com/2016/11/18/disruptor.html)
[Disruptor系列3：Disruptor样例实战](https://blog.csdn.net/twypx/article/details/80398886)
