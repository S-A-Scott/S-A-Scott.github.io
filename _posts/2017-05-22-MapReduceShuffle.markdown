---
layout:     post
title:      "MapReduce源码分析（三）Shuffle阶段"
subtitle:   "Source Code of MR Job Commit"
date:       2017-05-22 12:00:00
author:     "Spencer"
header-img: "img/post-bg-2015.jpg"
tags:
    - MapReduce
    - Hadoop
---

### 一. shuffle阶段执行过程
![shuffle.png](/img/in-post/post-js-version/shuffle.png)

* shuffle是MapReduce处理流程中的一个过程，大致分为如下过程
1. maptask收集map方法输出的k,v对，放入内存缓冲区中
2. 在内存缓冲区中不断溢出本地磁盘文件
3. 多个溢出的文件会被合并成大的溢出文件
4. 在溢出过程中，及合并的过程中，都要调用partitioner进行分组和针对key进行排序
5. reducetask根据自己的分区号，去各个maptask机器上取相应的分区数据
6. reducetask会将取到的同一分区的来自不同maptask的结果文件进行合并
7. 合并成大文件后，shuffle阶段结束。

### 二. shuffle阶段源码分析

* MapOutputCollector
 用户自定义的Mapper类的map方法中调用context.write(key, value)来将结果进行输出，context.write会调用NewOutputCollector的write方法
```java
public void write(K key, V value) throws IOException, InterruptedException {
    // collector默认为MapOutputBuffer
    collector.collect(key, value,
            partitioner.getPartition(key, value, partitions));
}
```
collector会在NewOutputCollector构造方法中通过createSortingCollector方法获得，如果没有手动设置，默认为MapOutputBuffer

---
* 首先先看MapOutputBuffer的init方法（因为长度原因，省略了部分不涉及数据逻辑的代码）
```java
public void init(MapOutputCollector.Context context
    ) throws IOException, ClassNotFoundException {
    // 获取reduce个数，客户端中通过job.setNumReduceTasks()设置
    // 如果不设置默认为1
    partitions = job.getNumReduceTasks();
    //sanity checks
    // 默认当缓冲区占用容量至80%时进行溢写，也是可以优化的地方
    final float spillper =
            job.getFloat(JobContext.MAP_SORT_SPILL_PERCENT, (float)0.8);
    // 默认缓冲区大小为100mb，为了提升性能，可以调高
    final int sortmb = job.getInt(JobContext.IO_SORT_MB, 100);
    indexCacheMemoryLimit = job.getInt(JobContext.INDEX_CACHE_MEMORY_LIMIT,
            INDEX_CACHE_MEMORY_LIMIT_DEFAULT);
    // sort的时候默认为快速排序
    sorter = ReflectionUtils.newInstance(job.getClass("map.sort.class",
            QuickSort.class, IndexedSorter.class), job);
    // buffers and accounting
    // 默认值为104857600，<< 20相当于 sortmb * (2^20)，表示需要104857600bytes的内存
    int maxMemUsage = sortmb << 20;
    // METASIZE的值为16， NMETA = 4表示num meta ints, METASIZE = NMETA * 4
    // 表示size in bytes
    // 这里减去maxMemUsage % METASIZE是为了使得maxMemUsage刚好是METASIZE的整数倍
    maxMemUsage -= maxMemUsage % METASIZE;
    // 100mb大小的数组，用作数据存放缓冲区
    kvbuffer = new byte[maxMemUsage];
    // bufvoid : marks the point where we should stop reading at the end of the buffer
    bufvoid = kvbuffer.length;
    // kvmeta : metadata overlay on backing store
    // kvmeta是NIO的IntBuffer对象，表示在kvbuffer中存放索引数据的区域
    kvmeta = ByteBuffer.wrap(kvbuffer)
            .order(ByteOrder.nativeOrder())
            .asIntBuffer();
    setEquator(0);
    // bufindex是指向kvbuffer写数据的位置指针
    bufstart = bufend = bufindex = equator;
    // kvindex是指向kvmeta写索引的位置指针
    kvstart = kvend = kvindex;

    maxRec = kvmeta.capacity() / NMETA;
    // 当缓冲区数据达到softLimit时开始溢写, 默认为
    softLimit = (int)(kvbuffer.length * spillper);
    // bufferRemaining表示再写入多少数据时，才会发生溢写
    bufferRemaining = softLimit;
    LOG.info(JobContext.IO_SORT_MB + ": " + sortmb);
    LOG.info("soft limit at " + softLimit);
    LOG.info("bufstart = " + bufstart + "; bufvoid = " + bufvoid);
    LOG.info("kvstart = " + kvstart + "; length = " + maxRec);

    // k/v serialization
    // key比较器，map溢写结果在分区内有序
    // 如果key为自定义的类，这个类实现WritableComparable后
    // 这里得到的comparator将会是getMapOutputKeyClass().asSubclass(WritableComparable.class)
    comparator = job.getOutputKeyComparator();
    // 获得客户端中设置的输出key类型 job.setMapOutputKeyClass(...)
    keyClass = (Class<K>)job.getMapOutputKeyClass();
    // 获得客户端中设置的输出value类型 job.setMapOutputValueClass(...)
    valClass = (Class<V>)job.getMapOutputValueClass();
    // 从配置文件读取序列化类的集合
    serializationFactory = new SerializationFactory(job);
    keySerializer = serializationFactory.getSerializer(keyClass);
    // 打开BlockingBuffer流，也就是后面输出key，value时会往这个BlockingBuffer
    // 流中写内容，BlockingBuffer内部封装了一个Buffer
    keySerializer.open(bb);
    valSerializer = serializationFactory.getSerializer(valClass);
    valSerializer.open(bb);

    // compression
    // 检查是否设置了压缩编码
    if (job.getCompressMapOutput()) {
        Class<? extends CompressionCodec> codecClass =
                job.getMapOutputCompressorClass(DefaultCodec.class);
        codec = ReflectionUtils.newInstance(codecClass, job);
    } else {
        codec = null;
    }

    // combiner
    final Counters.Counter combineInputCounter =
            reporter.getCounter(TaskCounter.COMBINE_INPUT_RECORDS);
    // 获得用户设置的CombinerClass，如何没有设置，默认为null
    combinerRunner = CombinerRunner.create(job, getTaskID(),
            combineInputCounter,
            reporter, null);
    if (combinerRunner != null) {
        final Counters.Counter combineOutputCounter =
                reporter.getCounter(TaskCounter.COMBINE_OUTPUT_RECORDS);
        combineCollector= new CombineOutputCollector<K,V>(combineOutputCounter, reporter, job);
    } else {
        combineCollector = null;
    }
    spillInProgress = false;
    minSpillsForCombine = job.getInt(JobContext.MAP_COMBINE_MIN_SPILLS, 3);
    spillThread.setDaemon(true);
    spillThread.setName("SpillThread");
    spillLock.lock();
    try {
        // 数据的溢写是由另一个线程完成的
        // 这样可以使得数据边写入缓冲区的时候边溢写，提高效率
        spillThread.start();
        while (!spillThreadRunning) {
            spillDone.await();
        }
    } catch (InterruptedException e) {
        throw new IOException("Spill thread failed to initialize", e);
    } finally {
        spillLock.unlock();
    }
    if (sortSpillException != null) {
        throw new IOException("Spill thread failed to initialize",
                sortSpillException);
    }
}
```
init方法中初始化接下来完成溢写工作需要的类，并启动了Spill线程。

* 接着看下MapOutputBuffer中的collect方法。
```java
public synchronized void collect(K key, V value, final int partition
    ) throws IOException {
    // bufferRemaining表示再写入多少数据时，才会发生溢写
    bufferRemaining -= METASIZE;
    // 当bufferRemaining <= 0时，开启溢写
    if (bufferRemaining <= 0) {
        // start spill if the thread is not running and the soft limit has been
        // reached
        spillLock.lock();
        try {
            do {
                if (!spillInProgress) {
                    // ...
                    // startSpill设置信号量，使SpillThread调用sortAndSpill方法对
                    // 缓存中的数据进行排序后溢写出文件
                    startSpill();
                    // ....
                }
            } while (false);
        } finally {
            spillLock.unlock();
        }
    }

    try {
        // serialize key bytes into buffer
        int keystart = bufindex;
        keySerializer.serialize(key);
        if (bufindex < keystart) {
            // wrapped the key; must make contiguous
            bb.shiftBufferedKey();
            keystart = 0;
        }
    }
}
```
