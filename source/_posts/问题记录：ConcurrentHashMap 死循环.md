---
title: 问题记录：ConcurrentHashMap 死循环
date: 2024-12-21 13:50:10
tags: 问题记录


---



### 问题的出现和定位

测试环境中，有测试同学反馈说接口请求慢，超时了也没有数据返回。

首先从测试同学那里了解到具体慢的接口，排查了对应的服务的状态，确定了服务的状态是正常在线的。

进入该服务对应的容器内，用 **jstat -gctil** 命令查看了服务进程的 GC 情况，发现不管是内存占用，或是 GC 的频率和时间占比，都不高。

然后又通过 **top** 看了下 CPU 的使用率，发现服务的进程整体使用率到了 90% 以上，随即通过 **top -Hp 1** 指定进程，定位到了占用的的具体线程id是 **196**：

<!--more-->



![image-20241221162635379](http://storage.laixiaoming.space/blog/image-20241221162635379.png)

有了线程 id 后，又通过 **jstack 1 jstack.log** 导出了该服务的线程堆栈信息，通过转换后的 16 进制的线程 id （c4）定位到了占用高的这个线程的堆栈信息：

![image-20241221163133550](http://storage.laixiaoming.space/blog/image-20241221163133550.png)

发现了该线程此时正在执行 **java.util.concurrent.ConcurrentHashMap.computeIfAbsent** 方法。难不成陷入了死循环？

带着疑惑翻开 computeIfAbsent 对应的源码，该方法的逻辑是如果指定的 key 在 map 中不存在，则通过 mappingFunction 方法计算出一个新值，并将其放入 map 中。该方法代码行数不多，逻辑清晰：

```
public V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction) {
    if (key == null || mappingFunction == null)
        throw new NullPointerException();
    // 计算 key 的 hash 值
    int h = spread(key.hashCode());
    V val = null;
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 初始化 
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 定位到桶
        else if ((f = tabAt(tab, i = (n - 1) & h)) == null) {
        	//占位节点
            Node<K,V> r = new ReservationNode<K,V>();
            synchronized (r) {
                if (casTabAt(tab, i, null, r)) {
                    binCount = 1;
                    Node<K,V> node = null;
                    try {
                        if ((val = mappingFunction.apply(key)) != null)
                            node = new Node<K,V>(h, key, val, null);
                    } finally {
                        setTabAt(tab, i, node);
                    }
                }
            }
            if (binCount != 0)
                break;
        }
        // 正在护容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            boolean added = false;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                	// 链表节点
                    if (fh >= 0) {
                        // 省略...
                    }
                    // 红黑树节点
                    else if (f instanceof TreeBin) {
                        // 省略...
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (!added)
                    return val;
                break;
            }
        }
    }
    if (val != null)
        addCount(1L, binCount);
    return val;
}
```

回看导出的线程堆栈信息，发现这个线程进入了两次 computeIfAbsent 方法，而且此时该线程第一次进入到 computeIfAbsent 后停留在了 mappingFunction 的执行，合理猜测是因为 mappingFunction 中包含了对同一个 map 的 computeIfAbsent ，而正是对同一个 map 进行 computeIfAbsent 递归操作导致了问题的产生。

结合现有信息，到网上（[原文](https://juejin.cn/post/6844904191077384200)）查了下，发现 ConcurrentHashMap 的 computeIfAbsent 方法果然有死循环这个问题！

好了，现在我们来看下ConcurrentHashMap 是怎么让自己陷入死循环的：

1. 第一次执行 computeIfAbsent 方法：此时 key 值不存在，会先往对应位放入一个预留节点 ReservationNode ，接着执行 mappingFunction 方法：

![image-20241221174003993](http://storage.laixiaoming.space/blog/image-20241221174003993.png)

2.  当此时 mappingFunction 包含了对同一个 map 的 computeIfAbsent 操作时，会第二次进入 computeIfAbsent 方法，而且当该 computeIfAbsent 操作的 key 与第一次进来时的 key 的 hash 值冲突时，此时两次操作定位到的槽会是同一个，再次进入 for 循环，进入之后一路执行，但发现所有的条件均不满足，也就只能无奈陷入了死循环了：

![image-20241221175332687](http://storage.laixiaoming.space/blog/image-20241221175332687.png)



### 问题的源头

那哪里会存在 computeIfAbsent 的递归调用呢？

通过线程的堆栈可以发现，两次的调用都是由 DataSource#getConnection 发起的，原来是因为该服务引入了 Seata ，而 Seata 默认会对所有数据源进行代理，并将代理后的对象存入 ConcurrentHashMap ，而存入的方式就是通过 computeIfAbsent 完成的，而且巧的是该服务同时也引入了 DynamicDataSource 动态数据源，动态数据源内部会维护多个真实的数据源，所以对动态数据源的操作都会转发到真实的数据源：

![image-20241221180132904](http://storage.laixiaoming.space/blog/image-20241221180132904.png)



### 解决方法

定位到问题的原因就简单了，既然是因为 Seata 对多个数据源的代理导致，那么只针对真实数据源进行代理理论上就可以解决该问题了，而 Seata 也恰好支持通过配置不代理指定的数据源，对应的配置是 **seata.excludes-for-auto-proxying** 。

而在 JDK9 中，作者实际上也修复了该问题，修复的方式也很简单粗暴，就是只要发现了递归调用的情况，直接抛异常：

![image-20241221181724042](http://storage.laixiaoming.space/blog/image-20241221181724042.png)

另外，在 JDK 8 中，computeIfAbsent 方法实际上也通过注释说明了 mappingFunction 不能包含对同一个 map 的递归操作，所以这个好像严格意义上也不算 bug ？只是使用姿势不对？

![image-20241221182622468](http://storage.laixiaoming.space/blog/image-20241221182622468.png)