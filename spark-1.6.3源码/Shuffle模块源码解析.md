# 简要介绍
shuffle模块是spark两个stage之间传递数据的实现模块，涉及到数据的读写操作，对性能的影响非常重要。
# shuffleManager

在spark中 shuffle模块是以插件形式提供的，要自定义shuffle,只要实现了ShuffleManager即可。目前spark提供两种shuffle模式：

sort --> SortShuffleManager

hash --> HashShuffleManager

ShuffleManager需要实现下面的方法:

![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/shuffleManager%E9%9C%80%E8%A6%81%E5%AE%9E%E7%8E%B0%E7%9A%84%E6%96%B9%E6%B3%95.png?raw=true)

# SortShuffleManager

1.只写出单个文件，写出的时候按分区ID排序

文件的内容大概是这样的

分区1 - 分区1对应数据 - 分区2 - 分区2对应数据

2.内存不足会将内存数据溢出到磁盘，最终会合并生成单个数据文件



##　sortshuffle写出文件的两种不同方式

1.序列化排序写出[堆外]
满足三个条件会使用这种输出：
* 没有aggregation操作或者排序操作
* 序列化支持值的重新定位[KryoSerializer和spark sql自定义的序列化目前都支持]
* 分区数量小于16777216

2.反序列化排序写出 除了上面以外的写出

### 序列化写出详细说明
传入的数据被传送到writer时会被序列化，并且在排序的过程会缓冲,spark对这种写出方式做了以下优化:

1.排序操作针对的是序列化对象而非原始Java对象，操作对象必须具有某些属性让其支持序列化排序

2.使用ShuffleExternalSorter进行排序，通过压缩记录指针和分区ID排序，每个压缩记录只占用8字节，这样更适合在内存中缓冲更多记录

3.相同分区的记录溢写合并不需要反序列化

4.当压缩编码支持连接操作，溢写合并的时候只需要简单的合并序列化和压缩溢写的分区组成最后的输出分区。这样就可以在合并的时候使用更有效的复制方式，不需要分配解压或者复制缓冲区

###　UnsafeShuffleWriter[堆外]
#### 满足条件

满足三个条件会使用这种输出：

* 没有aggregation操作或者排序操作

* 序列化支持值的重新定位[KryoSerializer和spark sql自定义的序列化目前都支持]

* 分区数量小于16777216[Long.MAX_VALUE]

最重要的方法是write

```java
@Override
  public void write(scala.collection.Iterator<Product2<K, V>> records) throws IOException
```

write方法调用insertRecordIntoSorter方法

```java
  void insertRecordIntoSorter(Product2<K, V> record) throws IOException {
    assert(sorter != null);
    final K key = record._1();
    final int partitionId = partitioner.getPartition(key);
    serBuffer.reset();
    serOutputStream.writeKey(key, OBJECT_CLASS_TAG);
    serOutputStream.writeValue(record._2(), OBJECT_CLASS_TAG);
    serOutputStream.flush();

    final int serializedRecordSize = serBuffer.size();
    assert (serializedRecordSize > 0);

    sorter.insertRecord(
      serBuffer.getBuf(), Platform.BYTE_ARRAY_OFFSET, serializedRecordSize, partitionId);
  }
```
主要是:
* 序列化key
* 序列化value
insertRecordIntoSorter 继续调用到```ShuffleExternalSorter.insertRecord```

```java
public void insertRecord(Object recordBase, long recordOffset, int length, int partitionId)
    throws IOException {

    // for tests
    assert(inMemSorter != null);
    //1.判断是否需要溢出到磁盘
    if (inMemSorter.numRecords() > numElementsForSpillThreshold) {
      spill();
    }
    //2.判断是否需要增长数组空间
    growPointerArrayIfNecessary();
    // Need 4 bytes to store the record length.
    //数据长度 + 4个字节为该对象占用总字节数 [4字节表示对象的长度]
    final int required = length + 4;
    //3.判断是否需要另起一个新页存储
    acquireNewPageIfNecessary(required);
    assert(currentPage != null);
    final Object base = currentPage.getBaseObject();
    //4.地址编码 使用页码和页的偏移量进行编码得到一个地址
    final long recordAddress = taskMemoryManager.encodePageNumberAndOffset(currentPage, pageCursor);
    //5.存放数据占用的长度
    Platform.putInt(base, pageCursor, length);
    //6.页内存储位置偏移量
    pageCursor += 4;
    // 7.拷贝到内存中的对应位置
    Platform.copyMemory(recordBase, recordOffset, base, pageCursor, length);
    //8.增加偏移量
    pageCursor += length;
    //记录数据在内存的位置 和对应分区ID
    inMemSorter.insertRecord(recordAddress, partitionId);
  }


```

当内存不足的时候，为了缓解内存压力，会将内存的数据溢出到磁盘，减

轻内存压力，具体实现在```ShuffleExternalSorter.spill```方法
分析调用链 ```ShuffleExternalSorter.spill``` --> ```ShuffleExternalSorter.writeSortedFile``` -->

```java
 private void writeSortedFile(boolean isLastFile) throws IOException {

    final ShuffleWriteMetrics writeMetricsToUse;
    //1.如果是最后一次写入  不用统计写出记录数
    if (isLastFile) {
      writeMetricsToUse = writeMetrics;
    } else {
      writeMetricsToUse = new ShuffleWriteMetrics();
    }

    //2.排序后的记录迭代器 并非真正迭代器 而是地址和分区ID的迭代器
    final ShuffleInMemorySorter.ShuffleSorterIterator sortedRecords =
      inMemSorter.getSortedIterator();
    DiskBlockObjectWriter writer;
    final byte[] writeBuffer = new byte[DISK_WRITE_BUFFER_SIZE];
    //3.写出临时文件
    final Tuple2<TempShuffleBlockId, File> spilledFileInfo =
      blockManager.diskBlockManager().createTempShuffleBlock();
    final File file = spilledFileInfo._2();
    final TempShuffleBlockId blockId = spilledFileInfo._1();
    //4.溢出信息 包括分区数量 写出文件  blockId
    final SpillInfo spillInfo = new SpillInfo(numPartitions, file, blockId);

    final SerializerInstance ser = DummySerializerInstance.INSTANCE;

    writer = blockManager.getDiskWriter(blockId, file, ser, fileBufferSizeBytes, writeMetricsToUse);

    int currentPartition = -1;
    // 5.遍历记录
    while (sortedRecords.hasNext()) {
      //5.1载入真实记录
      sortedRecords.loadNext();
      //5.2 获取分区ID
      final int partition = sortedRecords.packedRecordPointer.getPartitionId();
      //5.3 经过排序  遍历分区必须大于等于上一次遍历的分区
      assert (partition >= currentPartition);
      if (partition != currentPartition) {
        // Switch to the new partition
        // 5.3.1 切换到新的分区写出
        if (currentPartition != -1) {
          // 5.3.1.1 写出分区文件并关闭
          writer.commitAndClose();
          spillInfo.partitionLengths[currentPartition] = writer.fileSegment().length();
        }
        currentPartition = partition;
        //5.4 切换到分区writer
        writer =
          blockManager.getDiskWriter(blockId, file, ser, fileBufferSizeBytes, writeMetricsToUse);
      }
      // 6.记录地址
      final long recordPointer = sortedRecords.packedRecordPointer.getRecordPointer();
      //7.记录存放页
      final Object recordPage = taskMemoryManager.getPage(recordPointer);
      //8.记录在页面的偏移量
      final long recordOffsetInPage = taskMemoryManager.getOffsetInPage(recordPointer);
      //9.获取剩余数据
      int dataRemaining = Platform.getInt(recordPage, recordOffsetInPage);
      //10.跳过4个表示记录长度的字节
      long recordReadPosition = recordOffsetInPage + 4; // skip over record length
      //11.
      while (dataRemaining > 0) {
        final int toTransfer = Math.min(DISK_WRITE_BUFFER_SIZE, dataRemaining);
        //12.拷贝内存
        Platform.copyMemory(
          recordPage, recordReadPosition, writeBuffer, Platform.BYTE_ARRAY_OFFSET, toTransfer);
        //13.写出文件
        writer.write(writeBuffer, 0, toTransfer);
        recordReadPosition += toTransfer;
        dataRemaining -= toTransfer;
      }
      writer.recordWritten();
    }

    if (writer != null) {
      writer.commitAndClose();
      // If `writeSortedFile()` was called from `closeAndGetSpills()` and no records were inserted,
      // then the file might be empty. Note that it might be better to avoid calling
      // writeSortedFile() in that case.
      if (currentPartition != -1) {
        spillInfo.partitionLengths[currentPartition] = writer.fileSegment().length();
        spills.add(spillInfo);
      }
    }

    if (!isLastFile) {  // i.e. this is a spill file
      // The current semantics of `shuffleRecordsWritten` seem to be that it's updated when records
      // are written to disk, not when they enter the shuffle sorting code. DiskBlockObjectWriter
      // relies on its `recordWritten()` method being called in order to trigger periodic updates to
      // `shuffleBytesWritten`. If we were to remove the `recordWritten()` call and increment that
      // counter at a higher-level, then the in-progress metrics for records written and bytes
      // written would get out of sync.
      //
      // When writing the last file, we pass `writeMetrics` directly to the DiskBlockObjectWriter;
      // in all other cases, we pass in a dummy write metrics to capture metrics, then copy those
      // metrics to the true write metrics here. The reason for performing this copying is so that
      // we can avoid reporting spilled bytes as shuffle write bytes.
      //
      // Note that we intentionally ignore the value of `writeMetricsToUse.shuffleWriteTime()`.
      // Consistent with ExternalSorter, we do not count this IO towards shuffle write time.
      // This means that this IO time is not accounted for anywhere; SPARK-3577 will fix this.
      writeMetrics.incShuffleRecordsWritten(writeMetricsToUse.shuffleRecordsWritten());
      taskContext.taskMetrics().incDiskBytesSpilled(writeMetricsToUse.shuffleBytesWritten());
    }
  }

```

可见，采用堆外管理内存的方式稍微复杂了不少。要考虑到如何进行对外数据的存

取，还有堆外内存的溢出磁盘管理。不过如果采用RDD的方式来编程，基本上不会用

到这种shuffle写出方式。只有在spark-sql中才可能用到这种方式来写出，spark

团队对spark-sql进行了大量的优化




### BypassMergeSortShuffleWriter
满足这种shuffle方式的条件

1.分区数量少于200 ，可由spark.shuffle.sort.bypassMergeThreshold控制

2.没有map端的合并操作

#### write方法

```java
  @Override
  public void write(Iterator<Product2<K, V>> records) throws IOException {
    //.....忽略部分不重要代码
    final SerializerInstance serInstance = serializer.newInstance();
    final long openStartTime = System.nanoTime();
    //1.diskWriter负责写出磁盘
    partitionWriters = new DiskBlockObjectWriter[numPartitions];
    //2.按分区写出 每个分区对应一个文件 每个文件对应一个writer
    for (int i = 0; i < numPartitions; i++) {
      final Tuple2<TempShuffleBlockId, File> tempShuffleBlockIdPlusFile =
        blockManager.diskBlockManager().createTempShuffleBlock();
      final File file = tempShuffleBlockIdPlusFile._2();
      final BlockId blockId = tempShuffleBlockIdPlusFile._1();
      partitionWriters[i] =
        blockManager.getDiskWriter(blockId, file, serInstance, fileBufferSize, writeMetrics).open();
    }
    // Creating the file to write to and creating a disk writer both involve interacting with
    // the disk, and can take a long time in aggregate when we open many files, so should be
    // included in the shuffle write time.
    writeMetrics.incShuffleWriteTime(System.nanoTime() - openStartTime);
    //3.遍历记录
    while (records.hasNext()) {
      final Product2<K, V> record = records.next();
      final K key = record._1();
      //4.根据key获取分区writer进行写出
      partitionWriters[partitioner.getPartition(key)].write(key, record._2());
    }
    //5.提交并关闭所有分区writer
    for (DiskBlockObjectWriter writer : partitionWriters) {
      writer.commitAndClose();
    }
    //6.最终输出的data文件
    File output = shuffleBlockResolver.getDataFile(shuffleId, mapId);
    //临时文件
    File tmp = Utils.tempFileWith(output);
    try {
      //7.将所有分区输出文件写出到tmp
      partitionLengths = writePartitionedFile(tmp);
      //8.索引文件写出 并重命名tmp文件为对应的data文件
      shuffleBlockResolver.writeIndexFileAndCommit(shuffleId, mapId, partitionLengths, tmp);
    } finally {
      if (tmp.exists() && !tmp.delete()) {
        logger.error("Error while deleting temp file {}", tmp.getAbsolutePath());
      }
    }
    mapStatus = MapStatus$.MODULE$.apply(blockManager.shuffleServerId(), partitionLengths);
  }

```


### SortShuffleWriter

在满足```UnsafeShuffleWriter ```和 ```BypassMergeSortShuffleWriter```的时候，就会默认采用ShortShuffleWriter
重点还是write方法0
```java
 /** Write a bunch of records to this task's output */
  override def write(records: Iterator[Product2[K, V]]): Unit = {
    //省略了部分代码 ....
    //写出记录
    sorter.insertAll(records)

    // Don't bother including the time to open the merged output file in the shuffle write time,
    // because it just opens a single file, so is typically too fast to measure accurately
    // (see SPARK-3570).
    //合并任务的临时文件，合并生成data index文件
    val output = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
    val tmp = Utils.tempFileWith(output)
    try {
      val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
      val partitionLengths = sorter.writePartitionedFile(blockId, tmp)
      shuffleBlockResolver.writeIndexFileAndCommit(dep.shuffleId, mapId, partitionLengths, tmp)
      mapStatus = MapStatus(blockManager.shuffleServerId, partitionLengths)
    } finally {
      if (tmp.exists() && !tmp.delete()) {
        logError(s"Error while deleting temp file ${tmp.getAbsolutePath}")
      }
    }
  }

```
sorter是指ExternalSorter
sorter.insertAll(records)
```java
 def insertAll(records: Iterator[Product2[K, V]]): Unit = {
    //1。是否在map端合并
    val shouldCombine = aggregator.isDefined
    //2.map端合并
    if (shouldCombine) {
      // Combine values in-memory first using our AppendOnlyMap
      val mergeValue = aggregator.get.mergeValue
      val createCombiner = aggregator.get.createCombiner
      var kv: Product2[K, V] = null
      val update = (hadValue: Boolean, oldValue: C) => {
        if (hadValue) mergeValue(oldValue, kv._2) else createCombiner(kv._2)
      }
      while (records.hasNext) {
        addElementsRead()
        kv = records.next()
        map.changeValue((getPartition(kv._1), kv._1), update)
        //3.合并的过程可能需要溢出磁盘
        maybeSpillCollection(usingMap = true)
      }
    } else {
      // Stick values into our buffer
      while (records.hasNext) {
        addElementsRead()
        val kv = records.next()
        //写入缓冲
        buffer.insert(getPartition(kv._1), kv._1, kv._2.asInstanceOf[C])
        //可能需要溢出磁盘
        maybeSpillCollection(usingMap = false)
      }
    }
  }

```
### ExternalSorter


### ExternalSorter.maybeSpillCollection

```scala
  /**
   *溢出可能操作
   */
  private def maybeSpillCollection(usingMap: Boolean): Unit = {
    //预估大小
    var estimatedSize = 0L
    //是否正在使用map
    if (usingMap) {
      //map实现了估计大小的方法
      estimatedSize = map.estimateSize()
      //判断是否溢出
      if (maybeSpill(map, estimatedSize)) {
        map = new PartitionedAppendOnlyMap[K, C]
      }
    } else {
      //没有使用map 调用缓冲的预估大小方法
      estimatedSize = buffer.estimateSize()
      //溢出操作
      if (maybeSpill(buffer, estimatedSize)) {
        buffer = new PartitionedPairBuffer[K, C]
      }
    }
    //如果内存对象占用大小超过最大内存使用  更新变量
    if (estimatedSize > _peakMemoryUsedBytes) {
      _peakMemoryUsedBytes = estimatedSize
    }
  }
```
对于在map端合并数据的情况，使用了PartitionedAppendOnlyMap来进行合并操作，其继承结构
```PartitionedAppendOnlyMap``` <-- ```SizeTrackingAppendOnlyMap``` <--[```AppendOnlyMap``` ，```SizeTracker```]<--```Iterable```
SizeTracker : 可以追中对象的大小,通过调用SizeEstimator来估计对象大小，包含四个方法:```resetSamples,afterUpdate,takeSample,estimateSize```

PartitionedAppendOnlyMap:分区ID做为key ，分区的计算结果作为value
SizeTrackingAppendOnlyMap : 可以追踪大小的Map
AppendOnlyMap:只可追加的Map

####  分析SizeTrackingAppendOnlyMap

1. 可以追踪Map占用空间大小
2. 默认容量是64 * 2 = 128
3. 当map的数量大于可容纳的0.7，动态增加map容量
#### SizeTracker
可以追踪对象的大小,具体详情
如果想要给对象添加追踪使用内存空间大小的功能，只需要继承```SizeTracker```,然后在修改对象的方法中调用```super.afterUpdate```方法就可以拿到当前对象占用内存空间的大小。
实现思路如下：

内部有1个样本队列，队列元素存放占用内存大小和更新次数，表示这一

次更新和上一次更新经过抽象估计得大小，队列控制只能存放两个元素,

表示这一轮抽象大小和上一轮抽象大小。

需要注意的是，并不是每次更新对象都会进行评估内存占用的，而是经过

多次更新才会预估内存占用，那到底是多少次呢？默认是下一轮抽样预估

大小的时机是当更新次数达到上一轮次数的1.1倍的时候，就会执行内存

预估。


几个成员变量说明:

SAMPLE_GROWTH_RATE:1.1 控制抽样频次  相当于默认每一次都评估 如果是2 那就是2 4 8 。。。次的时候进行评估

samples:队列 存放最近两次的抽样结果

bytesPerUpdate:每次更新的字节数

numUpdates： 记录总共更新次数

nextSampleNum: 更新到多少次的时候开始抽样 ```nextSampleNum == numUpdates```会开始抽样

如何计算占用内存 ? 

```
最后一次评估的内存大小 + 每次更新占用字节数 * [总更新次数 - 最近一次抽样的记录点]
目前默认 最后一个数是0
```

### ExternalSorter.maybeSpill 溢出磁盘逻辑
需要溢出磁盘的下面两个条件的任意一个：
1.内存 当前占用内存大于规定内存阀值```initialMemoryThreshold``` [spark.shuffle.spill.initialMemoryThreshold 初始5M 每次以2倍当前内存 减去 阀值内存 来申请预留]

2.写入内存记录大于```spark.shuffle.spill.numElementsForceSpillThreshold```配置的记录数 默认是Long.MaxValue 也就是默认是不可能满足这个条件





#### SizeEstimator.estimate


# HashShuffleManager